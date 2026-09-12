<script setup>
import { ref, onMounted } from 'vue'
import client from '../api/client'
const emit = defineEmits(['close','changed'])
const lista=ref([]),rekesztipusok=ref([]),hiba=ref(''),betolt=ref(true),ujNev=ref(''),ujRekeszTipusId=ref(''),szerkesztId=ref(null),szerkNev=ref(''),szerkRekeszTipusId=ref(''),kepFolyamatban=ref(null)
const apiRoot=client.defaults.baseURL.replace(/\/api\/?$/,'')
function kepSrc(u){return u?.startsWith('http')?u:apiRoot+u}
async function frissit(){betolt.value=true;hiba.value='';try{const[z,r]=await Promise.all([client.get('/zoldsegek'),client.get('/rekesztipusok')]);lista.value=z.data;rekesztipusok.value=r.data}catch{hiba.value='Nem sikerült betölteni a listát.'}finally{betolt.value=false}}
onMounted(frissit)
function rekeszNev(id){return rekesztipusok.value.find(r=>r.id===id)?.nev||'—'}
async function ujFelvitel(){if(!ujNev.value.trim()){hiba.value='A név kötelező.';return}hiba.value='';try{await client.post('/zoldsegek',{nev:ujNev.value.trim(),alapertelmezettRekeszTipusId:ujRekeszTipusId.value||null});ujNev.value='';ujRekeszTipusId.value='';await frissit();emit('changed')}catch(e){hiba.value=e.response?.data?.message||'Mentés sikertelen.'}}
function szerkesztesInditasa(item){szerkesztId.value=item.id;szerkNev.value=item.nev;szerkRekeszTipusId.value=item.alapertelmezettRekeszTipusId||''}
function megse(){szerkesztId.value=null}
async function mentSzerkesztes(id){if(!szerkNev.value.trim()){hiba.value='A név kötelező.';return}hiba.value='';try{await client.put(`/zoldsegek/${id}`,{nev:szerkNev.value.trim(),alapertelmezettRekeszTipusId:szerkRekeszTipusId.value||null});szerkesztId.value=null;await frissit();emit('changed')}catch(e){hiba.value=e.response?.data?.message||'Mentés sikertelen.'}}
async function torol(item){if(!confirm(`Biztosan törlöd: \"${item.nev}\"?`))return;hiba.value='';try{await client.delete(`/zoldsegek/${item.id}`);await frissit();emit('changed')}catch(e){hiba.value=e.response?.data?.message||'Törlés sikertelen.'}}
async function kepValt(item,e){const file=e.target.files[0];e.target.value='';if(!file)return;if(!file.type.startsWith('image/')){hiba.value='Csak kép tölthető fel.';return}if(file.size>5_000_000){hiba.value='A kép mérete legfeljebb 5 MB lehet.';return}hiba.value='';kepFolyamatban.value=item.id;try{const fd=new FormData();fd.append('kep',file);await client.post(`/zoldsegek/${item.id}/kep`,fd,{headers:{'Content-Type':'multipart/form-data'}});await frissit();emit('changed')}catch(e2){hiba.value=e2.response?.data?.message||'Kép feltöltése sikertelen.'}finally{kepFolyamatban.value=null}}
async function kepTorlese(item){hiba.value='';try{await client.delete(`/zoldsegek/${item.id}/kep`);await frissit();emit('changed')}catch(e2){hiba.value=e2.response?.data?.message||'Kép törlése sikertelen.'}}
</script>
<template>
<div class="overlay" @click.self="emit('close')"><div class="modal">
  <div class="modal-head"><h3>Zöldségek kezelése</h3><button class="close" type="button" @click="emit('close')">✕</button></div>
  <p v-if="hiba" class="hiba">{{hiba}}</p><p v-if="betolt">Betöltés…</p>
  <ul v-else class="lista">
    <li v-for="item in lista" :key="item.id">
      <template v-if="szerkesztId===item.id">
        <div class="edit-main"><input v-model="szerkNev" placeholder="Zöldség neve"/><select v-model="szerkRekeszTipusId"><option value="">— nincs alap rekesztípus —</option><option v-for="r in rekesztipusok" :key="r.id" :value="r.id">{{r.nev}}</option></select></div>
        <div class="actions"><button class="action-btn save" type="button" title="Mentés" @click="mentSzerkesztes(item.id)">✓</button><button class="action-btn" type="button" title="Mégse" @click="megse">✕</button></div>
      </template>
      <template v-else>
        <img v-if="item.kepUrl" :src="kepSrc(item.kepUrl)" class="thumb" alt=""/><div v-else class="thumb empty-thumb">🥬</div>
        <div class="item-info"><strong>{{item.nev}}</strong><span v-if="item.alapertelmezettRekeszTipusId">Alap: {{rekeszNev(item.alapertelmezettRekeszTipusId)}}</span></div>
        <div class="actions"><label class="action-btn" title="Kép feltöltése/cseréje">{{kepFolyamatban===item.id?'⏳':'📷'}}<input type="file" accept="image/*" @change="kepValt(item,$event)"/></label><button v-if="item.kepUrl" class="action-btn danger" type="button" title="Kép törlése" @click="kepTorlese(item)">✕</button><button class="action-btn" type="button" title="Szerkesztés" @click="szerkesztesInditasa(item)">✏️</button><button class="action-btn danger" type="button" title="Törlés" @click="torol(item)">🗑️</button></div>
      </template>
    </li>
    <li v-if="!lista.length" class="ures">Még nincs felvéve zöldség.</li>
  </ul>
  <div class="uj-form"><input v-model="ujNev" placeholder="Zöldség neve" @keyup.enter="ujFelvitel"/><select v-model="ujRekeszTipusId"><option value="">— nincs alap rekesztípus —</option><option v-for="r in rekesztipusok" :key="r.id" :value="r.id">{{r.nev}}</option></select><button class="add" type="button" @click="ujFelvitel">+ Új zöldség</button></div>
  <p class="hint">Az alapértelmezett rekesztípust a Felvásárlás és Eladás automatikusan felajánlja, de tételenként felülírható.</p>
</div></div>
</template>
<style scoped>
.overlay{position:fixed;inset:0;background:rgba(35,38,32,.55);display:flex;align-items:center;justify-content:center;z-index:50;padding:12px}.modal{background:#F7F3E8;border-radius:12px;padding:18px;width:100%;max-width:620px;max-height:90vh;overflow-y:auto;box-shadow:0 20px 60px rgba(0,0,0,.25)}.modal-head{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}.modal-head h3{margin:0;font-size:18px;color:#2B3A2E}.close{background:none;border:0;font-size:20px;cursor:pointer;padding:6px;color:#6B7A4F}.hiba{color:#B3441E;font-size:13px;margin:4px 0 10px}.lista{list-style:none;margin:0 0 14px;padding:0}.lista li{display:flex;align-items:center;gap:10px;padding:10px 2px;border-bottom:1px solid #D8CFB8}.thumb{width:46px;height:46px;border-radius:9px;object-fit:cover;flex-shrink:0}.empty-thumb{display:flex;align-items:center;justify-content:center;background:rgba(107,122,79,.1);font-size:20px}.item-info{flex:1;min-width:0;display:flex;flex-direction:column;gap:3px}.item-info strong{color:#2B3A2E}.item-info span{font-size:12px;color:#6B7A4F}.actions{display:flex;gap:6px;flex-shrink:0}.action-btn{position:relative;width:42px;height:42px;border:1px solid rgba(43,58,46,.18);border-radius:9px;background:#fff;cursor:pointer;font-size:17px;display:flex;align-items:center;justify-content:center}.action-btn.save{background:#2B3A2E;color:#fff}.action-btn.danger{color:#B3441E}.action-btn input{position:absolute;inset:0;opacity:0;width:100%;height:100%;cursor:pointer}.edit-main{display:grid;grid-template-columns:1fr 1fr;gap:8px;flex:1;min-width:0}.edit-main input,.edit-main select,.uj-form input,.uj-form select{box-sizing:border-box;width:100%;padding:10px 11px;border:1px solid #D8CFB8;border-radius:8px;background:#fff;font-size:14px}.uj-form{display:grid;grid-template-columns:1fr 1fr auto;gap:8px}.add{min-height:42px;background:#C1440E;color:#fff;border:0;border-radius:8px;padding:0 15px;font-weight:700;cursor:pointer}.hint{font-size:12px;color:#6B7A4F;margin:10px 0 0}.ures{color:#6B7A4F;text-align:center;padding:18px 4px}
@media(max-width:650px){.overlay{padding:8px;align-items:flex-end}.modal{max-width:none;max-height:94vh;border-radius:14px 14px 0 0;padding:16px}.lista li{align-items:flex-start;flex-wrap:wrap}.item-info{min-width:calc(100% - 60px)}.actions{margin-left:auto}.action-btn{width:46px;height:46px}.edit-main{grid-template-columns:1fr;order:1;width:100%}.edit-main + .actions{order:2;width:100%;justify-content:flex-end}.uj-form{grid-template-columns:1fr}.add{width:100%;min-height:46px}}
</style>
