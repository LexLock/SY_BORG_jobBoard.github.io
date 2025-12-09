      fFaction: document.getElementById('g-faction'),
      fTag: document.getElementById('g-tag'),
      fDanger: document.getElementById('g-fDanger'),
      sort: document.getElementById('g-sort'),
      btnAdd: document.getElementById('g-btnAdd'),
      btnExport: document.getElementById('g-btnExport'),
      btnReset: document.getElementById('g-btnReset'),
      fileImport: document.getElementById('g-fileImport'),
      modal: document.getElementById('g-modal'),
      closeModal: document.getElementById('g-closeModal'),
      formTitle: document.getElementById('g-formTitle'),
      fTitle: document.getElementById('g-fTitle'),
      fFactionIn: document.getElementById('g-fFactionIn'),
      fPayout: document.getElementById('g-fPayout'),
      fDangerIn: document.getElementById('g-fDanger'),
      fTags: document.getElementById('g-fTags'),
      fLocation: document.getElementById('g-fLocation'),
      fExpires: document.getElementById('g-fExpires'),
      fContact: document.getElementById('g-fContact'),
      fDesc: document.getElementById('g-fDesc'),
      btnSave: document.getElementById('g-btnSave'),
      btnDelete: document.getElementById('g-btnDelete'),
    }
    const state = {
      gigs: [],
      filter: { q:'', status:'', faction:'', tag:'', danger:'', sort:'expires' },
      editingId: null
    }
    function plusDays(d){ const dt=new Date(Date.now()+d*86400000); return dt.toISOString().slice(0,10) }
    function seed(){
      return [
        { id:uid(), title:"Blackglass Heist", faction:"SiloDyne Internal", payout:9000, danger:4, tags:["stealth","corp","data"], location:"Arcology 6, Level -22", expires:plusDays(5), contact:"Proxy Canary — bin 7F", description:"Lift a quantum blackglass shard. Cameras loop 11m @03:12. Dual ICE on vault. No on-site casualties.", status:"open", createdAt: Date.now() },
        { id:uid(), title:"Noise at the Docks", faction:"Dock Union 404", payout:3500, danger:2, tags:["sabotage","industrial"], location:"Dry Dock 3B", expires:plusDays(2), contact:"Foreman Vex — 7k-ghost", description:"Jam a rival's crane telemetry. Make it look like 'weather'.", status:"open", createdAt: Date.now()-86400000 },
        { id:uid(), title:"Ghost the CFO", faction:"Blackgate", payout:15000, danger:5, tags:["wetwork","vip","urban"], location:"Skyline Spire, Penthouse", expires:plusDays(3), contact:"Agent Helios — @helios", description:"Hard kill, soft footprint. Expect 2–3 bodyguards, drone overwatch, panic room.", status:"claimed", createdAt: Date.now()-3600_000 },
        { id:uid(), title:"Scrap Ferry Escort", faction:"Rust Saints", payout:4800, danger:3, tags:["escort","vehicle","open-water"], location:"Estuary Route K", expires:plusDays(7), contact:"Dock chapel | SAINT-77", description:"Keep raiders off a slow barge for 12km. No heavy ordnance near reactor pods.", status:"open", createdAt: Date.now()-7200_000 },
        { id:uid(), title:"Counter-Message", faction:"CityNet Moderation (off-books)", payout:2600, danger:1, tags:["net","social","lowrisk"], location:"Anywhere", expires:plusDays(1), contact:"ticket: CN-17-663", description:"Drown a viral clip with seeded counter-memes. 24h window. Avoid obvious botnets.", status:"done", createdAt: Date.now()-5*86400000 }
      ]
    }
    function load(){
      const raw = storage.getItem(storeKey)
      if(raw){ try { state.gigs = JSON.parse(raw) } catch(e){} }
      if(!state.gigs?.length){ state.gigs = seed(); persist() }
      refreshFilters(); render()
    }
    function persist(){ try { storage.setItem(storeKey, JSON.stringify(state.gigs)) } catch(e){} }
    function refreshFilters(){
      const factions = [...new Set(state.gigs.map(g=>g.faction).filter(Boolean))].sort()
      const tags = [...new Set(state.gigs.flatMap(g=>g.tags||[]))].sort((a,b)=>a.localeCompare(b))
      el.fFaction.innerHTML = `<option value="">Faction: any</option>` + factions.map(f=>`<option>${escapeHtml(f)}</option>`).join('')
      el.fTag.innerHTML = `<option value="">Tag: any</option>` + tags.map(t=>`<option>${escapeHtml(t)}</option>`).join('')
    }
    function daysUntil(dateStr){
      if(!dateStr) return null
      const d=new Date(dateStr); if(isNaN(d)) return null
      const now=new Date(); return Math.ceil((d - new Date(now.toDateString()))/86400000)
    }
    function applyFilters(arr){
      const f = state.filter
      let out = arr.slice()
      if(f.q){
        const q=f.q.toLowerCase()
        out = out.filter(g => [g.title,g.faction,g.location,g.contact,g.description,(g.tags||[]).join(' ')].join(' ').toLowerCase().includes(q))
      }
      if(f.status) out = out.filter(g => g.status===f.status)
      if(f.faction) out = out.filter(g => g.faction===f.faction)
      if(f.tag) out = out.filter(g => (g.tags||[]).includes(f.tag))
      if(f.danger){ const d=+f.danger; out = out.filter(g => d===5 ? g.danger===5 : g.danger>=d) }
      switch(f.sort){
        case 'expires': out.sort((a,b)=> (a.expires||'9999-12-31').localeCompare(b.expires||'9999-12-31') || b.danger-a.danger); break;
        case 'new': out.sort((a,b)=> (b.createdAt||0)-(a.createdAt||0)); break;
        case 'payout': out.sort((a,b)=> (b.payout||0)-(a.payout||0)); break;
        case 'danger': out.sort((a,b)=> (b.danger||0)-(a.danger||0)); break;
      }
      return out
    }
    function render(){
      const gigs = applyFilters(state.gigs)
      el.list.innerHTML = gigs.map(renderCard).join('')
      el.empty.style.display = gigs.length ? 'none' : 'block'
      gigs.forEach(g => {
        const root = document.querySelector(`[data-gid="${g.id}"]`)
        root?.querySelector('.g-toggle')?.addEventListener('click', ()=>root.classList.toggle('open'))
        root?.querySelector('.g-claim')?.addEventListener('click', ()=>updateStatus(g.id,'claimed'))
        root?.querySelector('.g-done')?.addEventListener('click', ()=>updateStatus(g.id,'done'))
        root?.querySelector('.g-open')?.addEventListener('click', ()=>updateStatus(g.id,'open'))
        root?.querySelector('.g-edit')?.addEventListener('click', ()=>openEdit(g.id))
      })
    }
    function renderCard(g){
      const statusCls = g.status==='open'?'open':g.status==='claimed'?'claimed':'done'
      const daysLeft = daysUntil(g.expires)
      return `
      <div class="card" data-gid="${g.id}">
        <div class="row">
          <div class="title">${escapeHtml(g.title||'Untitled')}</div>
          <span class="pill hl">${escapeHtml(g.faction||'Freelance')}</span>
          ${(g.tags||[]).slice(0,4).map(t=>`<span class="pill">#${escapeHtml(t)}</span>`).join('')}
          <div class="spacer"></div>
          <span class="pill">¢ ${Number(g.payout||0).toLocaleString()}</span>
          <span class="pill">${escapeHtml('D'+(g.danger||1))}</span>
          <span class="status ${statusCls}">${g.status||'open'}</span>
          <button class="btn g-toggle">Details</button>
        </div>
        <div class="row" style="display:flex; gap:16px; align-items:flex-start">
          <div style="flex:3 1 520px">
            <div class="small muted"><span>Location:</span> ${escapeHtml(g.location||'—')} • <span>Expires:</span> ${escapeHtml(g.expires||'—')} • <span>Contact:</span> ${escapeHtml(g.contact||'—')}</div>
            <div class="rule"></div>
            <div>${escapeHtml(g.description||'')}</div>
          </div>
          <div style="flex:1 1 220px">
            <div class="small muted">Admin</div>
            <div class="inline" style="margin:8px 0 10px">
              <span class="pill">${g.expires ? `Due ${g.expires} ${daysLeft!=null?`(${daysLeft}d)`:''}` : 'No deadline'}</span>
            </div>
            <div class="inline" style="gap:8px;flex-wrap:wrap">
              ${g.status!=='claimed'?`<button class="btn g-claim">Mark Claimed</button>`:''}
              ${g.status!=='done'?`<button class="btn g-done">Mark Done</button>`:''}
              ${g.status!=='open'?`<button class="btn g-open">Reopen</button>`:''}
              <button class="btn g-edit">Edit</button>
            </div>
          </div>
        </div>
      </div>`
    }
    function updateStatus(id,status){ const g=state.gigs.find(x=>x.id===id); if(!g) return; g.status=status; persist(); render() }
    function showModal(on){ el.modal.classList.toggle('show', !!on); el.modal.setAttribute('aria-hidden', on?'false':'true') }
    function openNew(){ state.editingId=null; el.formTitle.textContent='New Gig'; el.btnDelete.style.display='none'; fillForm({}); showModal(true) }
    function openEdit(id){ state.editingId=id; const g=state.gigs.find(x=>x.id===id)||{}; el.formTitle.textContent='Edit Gig'; el.btnDelete.style.display='inline-flex'; fillForm(g); showModal(true) }
    function fillForm(g){
      el.fTitle.value=g.title||''; el.fFactionIn.value=g.faction||''; el.fPayout.value=g.payout??''; el.fDangerIn.value=g.danger??'';
      el.fTags.value=(g.tags||[]).join(', '); el.fLocation.value=g.location||''; el.fExpires.value=g.expires||'';
      el.fContact.value=g.contact||''; el.fDesc.value=g.description||'';
    }
    function readForm(){
      const title=el.fTitle.value.trim()
      const faction=el.fFactionIn.value.trim()
      const payout=Number(el.fPayout.value||0)
      const danger=Math.max(1,Math.min(5, Number(el.fDangerIn.value||1)))
      const tags=el.fTags.value.split(',').map(s=>s.trim()).filter(Boolean)
      const location=el.fLocation.value.trim()
      const expires=el.fExpires.value || ''
      const contact=el.fContact.value.trim()
      const description=el.fDesc.value.trim()
      return { title,faction,payout,danger,tags,location,expires,contact,description }
    }
    // Events
    el.q.addEventListener('input', e=>{ state.filter.q=e.target.value; render() })
    el.fStatus.addEventListener('change', e=>{ state.filter.status=e.target.value; render() })
    el.fFaction.addEventListener('change', e=>{ state.filter.faction=e.target.value; render() })
    el.fTag.addEventListener('change', e=>{ state.filter.tag=e.target.value; render() })
    el.fDanger.addEventListener('change', e=>{ state.filter.danger=e.target.value; render() })
    el.sort.addEventListener('change', e=>{ state.filter.sort=e.target.value; render() })
    el.btnAdd.addEventListener('click', openNew)
    el.closeModal.addEventListener('click', ()=>showModal(false))
    el.modal.addEventListener('click', (e)=>{ if(e.target===el.modal) showModal(false) })
    el.btnSave.addEventListener('click', ()=>{
      const data=readForm()
      if(!data.title){ alert('Title is required.'); return }
      if(state.editingId){
        const i=state.gigs.findIndex(x=>x.id===state.editingId)
        if(i>-1) state.gigs[i]={...state.gigs[i], ...data}
      }else{
        state.gigs.unshift({ id:uid(), status:'open', createdAt: Date.now(), ...data })
      }
      persist(); refreshFilters(); render(); showModal(false)
    })
    el.btnDelete.addEventListener('click', ()=>{
      if(!state.editingId) return
      const g=state.gigs.find(x=>x.id===state.editingId)
      if(g && confirm(`Delete gig: "${g.title}"?`)){
        state.gigs = state.gigs.filter(x=>x.id!==state.editingId)
        persist(); refreshFilters(); render(); showModal(false)
      }
    })
    el.btnExport.addEventListener('click', ()=>{
      const blob = new Blob([JSON.stringify(state.gigs, null, 2)], {type:'application/json'})
      const url = URL.createObjectURL(blob); const a=document.createElement('a')
      a.href=url; a.download='cy_borg_gigs.json'; a.click(); URL.revokeObjectURL(url)
    })
    el.fileImport.addEventListener('change', async (e)=>{
      const file=e.target.files?.[0]; if(!file) return
      try{
        const text=await file.text()
        const data=JSON.parse(text)
        if(!Array.isArray(data)) throw new Error('Invalid JSON (expected array)')
        data.forEach(x=> x.id = x.id || uid())
        state.gigs=data
        persist(); refreshFilters(); render()
      }catch(err){
        alert('Import failed: ' + err.message)
      }finally{ e.target.value='' }
    })
    el.btnReset.addEventListener('click', ()=>{
      if(confirm('Reset board to sample data? This overwrites current gigs.')){
        state.gigs = seed(); persist(); refreshFilters(); render()
      }
    })
    load()

    // expose helper for Map
    window.__gigs_prefill = function({q, location, tags}) {
      if(typeof q==='string'){ el.q.value=q; el.q.dispatchEvent(new Event('input', {bubbles:true})) }
      if(location!=null){ openNew(); el.fLocation.value = location; if(tags){ el.fTags.value = Array.isArray(tags)?tags.join(', '):String(tags) } }
    }
  })()

  // CHARACTERS
  ;(() => {
    const storeKey = 'cy_borg_chars_v1'
    const el = {
      list: document.getElementById('c-list'),
      empty: document.getElementById('c-empty'),
      q: document.getElementById('c-q'),
      fRole: document.getElementById('c-fRole'),
      fFaction: document.getElementById('c-fFaction'),
      sort: document.getElementById('c-sort'),
      btnNew: document.getElementById('c-btnNew'),
      btnExport: document.getElementById('c-btnExport'),
      btnReset: document.getElementById('c-btnReset'),
      fileImport: document.getElementById('c-fileImport'),
      // modal
      modal: document.getElementById('c-modal'),
      formTitle: document.getElementById('c-formTitle'),
      closeModal: document.getElementById('c-closeModal'),
      btnSave: document.getElementById('c-btnSave'),
      btnDelete: document.getElementById('c-btnDelete'),
      // fields
      alias: document.getElementById('c-alias'),
      handle: document.getElementById('c-handle'),
      pronouns: document.getElementById('c-pronouns'),
      contact: document.getElementById('c-contact'),
      faction: document.getElementById('c-faction'),
      role: document.getElementById('c-role'),
      tags: document.getElementById('c-tags'),
      background: document.getElementById('c-background'),
      AGI: document.getElementById('c-AGI'),
      PRE: document.getElementById('c-PRE'),
      STR: document.getElementById('c-STR'),
      TGH: document.getElementById('c-TGH'),
      TEC: document.getElementById('c-TEC'),
      KNO: document.getElementById('c-KNO'),
      hpOut: document.getElementById('c-hpOut'),
      loadOut: document.getElementById('c-loadOut'),
      implants: document.getElementById('c-implants'),
      gear: document.getElementById('c-gear'),
      perks: document.getElementById('c-perks'),
      privacy: document.getElementById('c-privacy'),
      terms: document.getElementById('c-terms'),
    }
    const state = {
      chars: [],
      filter: { q:'', role:'', faction:'', sort:'new' },
      editingId: null
    }
    function seed(){
      return [
        {
          id: uid(),
          alias: "NEON SICKLE",
          handle: "@sickle.exe",
          pronouns: "they/them",
          contact: "node://glitch-bite",
          faction: "Dock Union 404",
          role: "deckburner",
          tags: ["netrunner","stealth","union"],
          background: "Born under the bridge. Learned to swim in wires. Owes a foreman a life.",
          abilities: { AGI: 2, PRE: 0, STR: -1, TGH: 1, TEC: 3, KNO: 0 },
          hp: 8, load: 7,
          implants: [ {name:"GhostSkin MkII", note:"optical dampening"} ],
          gear: [ {name:"Deck: RATS-9", note:"burner OS"}, {name:"Mono-wire", note:"quiet"} ],
          perks: ["night vision","lucky"],
          privacy: false, terms: true,
          createdAt: Date.now()
        },
        {
          id: uid(),
          alias: "BAY SAINT",
          handle: "@ferrite.psalm",
          pronouns: "she/her",
          contact: "dock-chapel://SAINT-77",
          faction: "Rust Saints",
          role: "escort",
          tags: ["open-water","escort","industrial"],
          background: "Pilgrim of rust. Keeps raiders off the slow boats.",
          abilities: { AGI: 0, PRE: 1, STR: 2, TGH: 2, TEC: -1, KNO: -1 },
          hp: 10, load: 10,
          implants: [ {name:"Myomer weave", note:"+carry"} ],
          gear: [ {name:"Slug thrower", note:"loud"}, {name:"Rebreather", note:"salt-proof"} ],
          perks: ["steady","stubborn"],
          privacy: true, terms: true,
          createdAt: Date.now()-7200_000
        }
      ]
    }
    function load(){
      const raw = storage.getItem(storeKey)
      if(raw){ try { state.chars = JSON.parse(raw) } catch(e){} }
      if(!state.chars?.length){ state.chars = seed(); persist() }
      refreshFilters(); render()
    }
    function persist(){ try { storage.setItem(storeKey, JSON.stringify(state.chars)) } catch(e){} }
    function refreshFilters(){
      const roles = [...new Set(state.chars.map(c=>c.role).filter(Boolean))].sort()
      const factions = [...new Set(state.chars.map(c=>c.faction).filter(Boolean))].sort()
      el.fRole.innerHTML = `<option value="">Role: any</option>` + roles.map(r=>`<option>${escapeHtml(r)}</option>`).join('')
      el.fFaction.innerHTML = `<option value="">Faction: any</option>` + factions.map(f=>`<option>${escapeHtml(f)}</option>`).join('')
    }
    function applyFilters(arr){
      const f = state.filter
      let out = arr.slice()
      if(f.q){
        const q=f.q.toLowerCase()
        out = out.filter(c => [c.alias,c.handle,c.faction,c.role,c.contact,c.background,(c.tags||[]).join(' ')].join(' ').toLowerCase().includes(q))
      }
      if(f.role) out = out.filter(c => c.role===f.role)
      if(f.faction) out = out.filter(c => c.faction===f.faction)
      switch(f.sort){
        case 'new': out.sort((a,b)=> (b.createdAt||0)-(a.createdAt||0)); break;
        case 'alias': out.sort((a,b)=> (a.alias||'').localeCompare(b.alias||'')); break;
        case 'role': out.sort((a,b)=> (a.role||'').localeCompare(b.role||'')); break;
      }
      return out
    }
    function initials(s=''){ const w=s.trim().split(/\s+/).slice(0,2).map(x=>x[0]?.toUpperCase()||'').join(''); return w||'??' }
    function render(){
      const chars = applyFilters(state.chars)
      el.list.innerHTML = chars.map(renderCard).join('')
      el.empty.style.display = chars.length ? 'none' : 'block'
      chars.forEach(c=>{
        const root = document.querySelector(`[data-cid="${c.id}"]`)
        root?.querySelector('.c-toggle')?.addEventListener('click', ()=>root.classList.toggle('open'))
        root?.querySelector('.c-edit')?.addEventListener('click', ()=>openEdit(c.id))
        root?.querySelector('.c-del')?.addEventListener('click', ()=>{ if(confirm(`Delete ${c.alias}?`)){ delChar(c.id) } })
      })
    }
    function renderCard(c){
      const tags=(c.tags||[]).slice(0,4)
      return `
      <div class="card" data-cid="${c.id}">
        <div class="row">
          <div class="avatar">${initials(c.alias)}</div>
          <div class="title">${escapeHtml(c.alias||'Unnamed')}</div>
          <span class="pill hl">@${escapeHtml((c.handle||'').replace(/^@/,''))}</span>
          ${c.role?`<span class="pill">${escapeHtml(c.role)}</span>`:''}
          ${tags.map(t=>`<span class="pill">#${escapeHtml(t)}</span>`).join('')}
          <div class="spacer"></div>
          <span class="status open">active</span>
          <button class="btn c-toggle">Profile</button>
          <button class="btn c-edit">Edit</button>
          <button class="btn warn c-del">Delete</button>
        </div>
        <div class="row" style="display:flex;gap:16px;align-items:flex-start">
          <div style="flex:3 1 520px">
            <div class="small muted">Faction</div>
            <div>${escapeHtml(c.faction||'—')}</div>
            <div class="rule"></div>
            <div class="small muted">Background</div>
            <div style="white-space:pre-wrap">${escapeHtml(c.background||'')}</div>
          </div>
          <div style="flex:1 1 240px">
            <div class="small muted">Stats</div>
            <div class="inline" style="gap:6px;margin:6px 0;flex-wrap:wrap">
              ${['AGI','PRE','STR','TGH','TEC','KNO'].map(k=>`<span class="pill">${k}:${(c.abilities?.[k]??0)>=0?'+':''}${c.abilities?.[k]??0}</span>`).join('')}
              <span class="pill">HP:${c.hp||6}</span>
              <span class="pill">Load:${c.load||8}</span>
            </div>
            <div class="small muted">Implants</div>
            <div class="inline" style="gap:6px;flex-wrap:wrap;margin:6px 0">
              ${(c.implants||[]).map(i=>`<span class="chip">${escapeHtml(i.name)} <span class="muted small">(${escapeHtml(i.note||'')})</span></span>`).join('') || '<span class="muted small">none</span>'}
            </div>
            <div class="small muted">Gear</div>
            <div class="inline" style="gap:6px;flex-wrap:wrap;margin:6px 0">
              ${(c.gear||[]).map(g=>`<span class="chip">${escapeHtml(g.name)} <span class="muted small">(${escapeHtml(g.note||'')})</span></span>`).join('') || '<span class="muted small">none</span>'}
            </div>
          </div>
        </div>
      </div>`
    }
    function delChar(id){ state.chars = state.chars.filter(x=>x.id!==id); persist(); refreshFilters(); render() }
    function showModal(on){ el.modal.classList.toggle('show', !!on); el.modal.setAttribute('aria-hidden', on?'false':'true') }
    function openNew(){ state.editingId=null; el.formTitle.textContent='Create Account'; el.btnDelete.style.display='none'; fillForm(blank()); showModal(true) }
    function openEdit(id){ state.editingId=id; const c=state.chars.find(x=>x.id===id)||blank(); el.formTitle.textContent='Edit Account'; el.btnDelete.style.display='inline-flex'; fillForm(c); showModal(true) }
    function blank(){
      return { alias:'',handle:'',pronouns:'',contact:'',faction:'',role:'',tags:[],background:'',
        abilities:{AGI:0,PRE:0,STR:0,TGH:0,TEC:0,KNO:0}, hp:6, load:8, implants:[], gear:[], perks:[], privacy:false, terms:false }
    }
    function fillForm(c){
      el.alias.value=c.alias||''; el.handle.value=c.handle||''; el.pronouns.value=c.pronouns||''; el.contact.value=c.contact||'';
      el.faction.value=c.faction||''; el.role.value=c.role||''; el.tags.value=(c.tags||[]).join(', '); el.background.value=c.background||'';
      el.AGI.value=c.abilities?.AGI ?? 0; el.PRE.value=c.abilities?.PRE ?? 0; el.STR.value=c.abilities?.STR ?? 0;
      el.TGH.value=c.abilities?.TGH ?? 0; el.TEC.value=c.abilities?.TEC ?? 0; el.KNO.value=c.abilities?.KNO ?? 0;
      el.hpOut.textContent=c.hp||6; el.loadOut.textContent=c.load||8;
      el.implants.value = (c.implants||[]).map(i=>`${i.name} — ${i.note||''}`.trim()).join('\n')
      el.gear.value = (c.gear||[]).map(g=>`${g.name} — ${g.note||''}`.trim()).join('\n')
      el.perks.value = (c.perks||[]).join(', ')
      el.privacy.checked = !!c.privacy
      el.terms.checked = !!c.terms
    }
    function parseLines(s){
      return s.split('\n').map(l=>l.trim()).filter(Boolean).map(l=>{
        const idx = l.indexOf('—'); if(idx===-1) return {name:l, note:''}
        return { name:l.slice(0,idx).trim(), note:l.slice(idx+1).trim() }
      }).filter(x=>x.name)
    }
    function clamp(n,min,max){ return Math.max(min, Math.min(max, n)) }
    function readForm(){
      const alias=el.alias.value.trim(), handle=el.handle.value.trim(), pronouns=el.pronouns.value.trim(), contact=el.contact.value.trim()
      const faction=el.faction.value.trim(), role=el.role.value.trim(), tags=el.tags.value.split(',').map(s=>s.trim()).filter(Boolean)
      const background=el.background.value.trim()
      const abilities = { AGI: clamp(+el.AGI.value||0,-3,3), PRE: clamp(+el.PRE.value||0,-3,3), STR: clamp(+el.STR.value||0,-3,3), TGH: clamp(+el.TGH.value||0,-3,3), TEC: clamp(+el.TEC.value||0,-3,3), KNO: clamp(+el.KNO.value||0,-3,3) }
      const hp = 6 + Math.max(0, abilities.TGH)*2
      const load = 8 + abilities.STR
      const implants = parseLines(el.implants.value)
      const gear = parseLines(el.gear.value)
      const perks = el.perks.value.split(',').map(s=>s.trim()).filter(Boolean)
      const privacy = !!el.privacy.checked
      const terms = !!el.terms.checked
      return { alias,handle,pronouns,contact,faction,role,tags,background,abilities,hp,load,implants,gear,perks,privacy,terms }
    }
    function updateDerived(){
      const T = clamp(+el.TGH.value||0,-3,3), S = clamp(+el.STR.value||0,-3,3)
      el.hpOut.textContent = 6 + Math.max(0,T)*2
      el.loadOut.textContent = 8 + S
    }
    // Events filters
    el.q.addEventListener('input', e=>{ state.filter.q=e.target.value; render() })
    el.fRole.addEventListener('change', e=>{ state.filter.role=e.target.value; render() })
    el.fFaction.addEventListener('change', e=>{ state.filter.faction=e.target.value; render() })
    el.sort.addEventListener('change', e=>{ state.filter.sort=e.target.value; render() })
    // Modal
    el.btnNew.addEventListener('click', openNew)
    el.closeModal.addEventListener('click', ()=>showModal(false))
    document.getElementById('c-modal').addEventListener('click', (e)=>{ if(e.target===document.getElementById('c-modal')) showModal(false) })
    ;['AGI','PRE','STR','TGH','TEC','KNO'].forEach(id=> el[id].addEventListener('input', updateDerived))
    el.btnSave.addEventListener('click', ()=>{
      const data = readForm()
      if(!data.alias || !data.handle || !data.terms){ alert('Alias, Handle, and Terms are required.'); return }
      if(state.editingId){
        const i = state.chars.findIndex(x=>x.id===state.editingId)
        if(i>-1) state.chars[i] = { ...state.chars[i], ...data }
      } else {
        state.chars.unshift({ id: uid(), createdAt: Date.now(), ...data })
      }
      persist(); refreshFilters(); render(); showModal(false)
    })
    el.btnDelete.addEventListener('click', ()=>{
      if(!state.editingId) return
      const c = state.chars.find(x=>x.id===state.editingId)
      if(c && confirm(`Delete account "${c.alias}"?`)){ delChar(state.editingId); showModal(false) }
    })
    // Export/Import
    el.btnExport.addEventListener('click', ()=>{
      const blob = new Blob([JSON.stringify(state.chars, null, 2)], {type:'application/json'})
      const url = URL.createObjectURL(blob); const a=document.createElement('a')
      a.href=url; a.download='cy_borg_chars.json'; a.click(); URL.revokeObjectURL(url)
    })
    el.fileImport.addEventListener('change', async (e)=>{
      const file=e.target.files?.[0]; if(!file) return
      try{
        const text=await file.text()
        const data=JSON.parse(text)
        if(!Array.isArray(data)) throw new Error('Invalid JSON (expected array)')
        data.forEach(x=> x.id = x.id || uid())
        state.chars=data
        persist(); refreshFilters(); render()
      }catch(err){ alert('Import failed: ' + err.message) }
      finally{ e.target.value='' }
    })
    el.btnReset.addEventListener('click', ()=>{
      if(confirm('Reset to sample accounts? This overwrites current data.')){
        state.chars = seed(); persist(); refreshFilters(); render()
      }
    })
    load()

    // expose helpers for Map
    window.__chars_search = function(q){
      el.q.value=q||''; el.q.dispatchEvent(new Event('input', {bubbles:true}))
    }
    window.__chars_new_prefill = function({faction, role, tags}){
      openNew()
      if(faction) el.faction.value=faction
      if(role) el.role.value=role
      if(tags) el.tags.value = Array.isArray(tags)?tags.join(', '):String(tags)
    }
  })()

  // MAP
  ;(() => {
    const el = {
      svg: document.getElementById('mapSVG'),
      gLinks: document.getElementById('links'),
      gDistricts: document.getElementById('districts'),
      tooltip: document.getElementById('tooltip'),
      loreBody: document.getElementById('loreBody'),
      loreMeta: document.getElementById('loreMeta'),
      toGigs: document.getElementById('m-toGigs'),
      newGig: document.getElementById('m-newGig'),
      toChars: document.getElementById('m-toChars'),
      q: document.getElementById('m-q'),
      tag: document.getElementById('m-tag'),
      danger: document.getElementById('m-danger'),
      reset: document.getElementById('m-reset'),
    }

    // Clean schematic: grid-aligned rounded blocks
    const W=1000,H=700
    const blocks = [
      { id:'chats',     name:'Chatswood Control', x:100, y:80,  w:160, h:80,  danger:2, tags:['metro','driverless','emergency'], blurb:"Trains hum without drivers until they don’t. Doors argue with lungs.", hooks:['Push a ghost train through rush.','Swap a card logic.','Blind a platform 90s.'] },
      { id:'lanecove',  name:'Lane Cove Node',    x:140, y:190, w:160, h:80,  danger:3, tags:['suburban','data','node'], blurb:"Thumbprint glass older than sin. Backyards hide black fiber.", hooks:['Plant a bypass then vanish.','Piggyback a garbage truck.','Sell a bandwidth ghost.'] },
      { id:'parra',     name:'Parramatta Gate',   x:60,  y:340, w:180, h:90,  danger:2, tags:['river','tide','escort'], blurb:"Tide gates creak. Junk islands bump your hull like old friends.", hooks:['Guide a barge through a squall.','Detach pirates from your stern.','Find a tide window that isn’t real.'] },
      { id:'whitebay',  name:'White Bay Power',   x:280, y:200, w:180, h:90,  danger:4, tags:['blackout','infrastructure','heist'], blurb:"Old turbines echo. New guards run on grudges. Dark isn’t safe.", hooks:['Rip the copper bible.','Plant a gridworm.','Ride the flywheel without arms.'] },
      { id:'rozelle',   name:'Rozelle Interchg',  x:280, y:320, w:180, h:90,  danger:3, tags:['tunnel','industrial','sabotage'], blurb:"Pumps cough methane prayers. Hydrogen pockets sing sleep.", hooks:['Rust a pump from inside.','Falsify a flood alert.','Lose a body in the sumps.'] },
      { id:'glebe',     name:'Blackwattle/Glebe', x:450, y:330, w:190, h:90,  danger:2, tags:['looting','markets','cold-storage'], blurb:"Fish stink covers crimes. Cold rooms hum. Storms make guards lazy.", hooks:['Lift five unmarked cases.','Poison a bid.','Swap a barcode on live tuna.'] },
      { id:'marrick',   name:'Marrickville Lat.', x:280, y:450, w:220, h:90,  danger:2, tags:['rooftops','ads','sabotage'], blurb:"Ad-lattices stitch the sky with lies. Drones loop—until they don’t.", hooks:['Unbolt three nodes.','Hijack a feed 60m.','Tag a sponsor live.'] },
      { id:'alex',      name:'Alexandria Maze',   x:520, y:450, w:180, h:90,  danger:3, tags:['warehouses','forklifts','night'], blurb:"Concrete canyons. Loaders dream of crushing. Slag dust sticks.", hooks:['Brick a loader quietly.','Hide a pallet in sight.','Map a private corridor.'] },
      { id:'kingsx',    name:'Kings Cross',       x:650, y:360, w:160, h:90,  danger:3, tags:['riot','neon','crowd'], blurb:"Neon spine with teeth. Protests flex. Cops flex harder.", hooks:['Shepherd a crowd from glass.','Plant a whisper in a megaphone.','Black out a sign at worst time.'] },
      { id:'harbour',   name:'Harbour Spine',     x:560, y:240, w:180, h:90,  danger:3, tags:['bridge','infrastructure','surveillance'], blurb:"Masts and arrays feed pattern-recognition to a hungry grid.", hooks:['Loop mast cams 8m.','Smuggle a corpse-still crate.','Ghost a patrol drone.'] },
      { id:'rocks',     name:'The Rocks',         x:620, y:280, w:150, h:70,  danger:2, tags:['historic','dead-drops','smuggling'], blurb:"Argyle Cut bleeds secrets. Old stone, new debts.", hooks:['Clean four caches.','Run a tunnel sprint.','Lose a tail in stairs.'] },
      { id:'quay',      name:'Circular Quay',     x:750, y:280, w:160, h:70,  danger:4, tags:['tourist','ferries','wetwork'], blurb:"Ferries and glass. Crowds film everything. Doors close like jaws.", hooks:['Make a stumble look real.','Swap a briefcase mid-flood.','Ride under a ferry.'] },
      { id:'barangaroo',name:'Barangaroo Spire',  x:640, y:200, w:170, h:70,  danger:4, tags:['corp','penthouse','extraction'], blurb:"Mirrored halls. Scent-tag drones. Smiles with knives.", hooks:['Lift an analyst and a dog.','Spoof a gala guestlist.','Swap a shard in the lift.'] },
      { id:'unsw',      name:'UNSW Spine',        x:820, y:420, w:160, h:80,  danger:4, tags:['data','labs','net'], blurb:"Research caches hum under student debt. ICE names bite.", hooks:['Backdoor moderation mill.','Borrow a blade, return worse.','Swap a thesis for a killfile.'] },
      { id:'randwick',  name:'Randwick BioLab',   x:840, y:520, w:160, h:80,  danger:4, tags:['biotech','vaults','sabotage'], blurb:"Seeds and sins in sub-cellars. Vaults purge like lungs.", hooks:['Corrupt a checksum.','Escort a whistleblower.','Ghost a temp dip.'] },
      { id:'botany',    name:'Port Botany',       x:780, y:610, w:180, h:80,  danger:3, tags:['quarantine','piers','escort'], blurb:"Cold cargo, colder rules. Skiffs ride dirty wakes.", hooks:['Run meds under a cordon.','Spoof a quarantine flag.','Extract a sick captain.'] },
      { id:'kurnell',   name:'Kurnell Stacks',    x:900, y:610, w:180, h:80,  danger:4, tags:['refinery','hazard','explosion'], blurb:"Vapor clouds bloom into rolling fire. Saints pray in boiler rooms.", hooks:['Stage inspection outage.','Siphon a tank that screams.','Hide a drone in a flare.'] }
    ]
    const edges = [
      ['chats','lanecove'],['lanecove','whitebay'],['whitebay','harbour'],['harbour','rocks'],['rocks','quay'],['quay','barangaroo'],
      ['whitebay','rozelle'],['rozelle','glebe'],['glebe','kingsx'],['kingsx','unsw'],['unsw','randwick'],['randwick','botany'],['botany','kurnell'],
      ['parra','lanecove'],['parra','rozelle'],['marrick','alex'],['rozelle','marrick'],['marrick','glebe']
    ]

    let filter = { q:'', tag:'', danger:'' }
    let selected = null

    function renderControls(){
      const tags = [...new Set(blocks.flatMap(d=>d.tags))].sort()
      document.getElementById('m-tag').innerHTML = `<option value="">Tag: any</option>` + tags.map(t=>`<option>${escapeHtml(t)}</option>`).join('')
    }
    function colorFor(d){
      return d<=1?'#7ce9a1':d<=2?'#9de96e':d<=3?'#ffd35a':d<=4?'#ff816a':'#ff3d76'
    }
    function center(b){ return {cx:b.x+b.w/2, cy:b.y+b.h/2} }
    function matches(d){
      const q = filter.q.toLowerCase()
      const mQ = !q || [d.name, d.tags.join(' '), d.blurb].join(' ').toLowerCase().includes(q)
      const mT = !filter.tag || d.tags.includes(filter.tag)
      const mD = !filter.danger || (+filter.danger===5 ? d.danger===5 : d.danger>=+filter.danger)
      return mQ && mT && mD
    }
    function renderMap(){
      // links
      el.gLinks.innerHTML = edges.map(([a,b])=>{
        const A = blocks.find(x=>x.id===a), B = blocks.find(x=>x.id===b)
        if(!A||!B) return ''
        const pA=center(A), pB=center(B)
        return `<line class="conn connGlow" x1="${pA.cx}" y1="${pA.cy}" x2="${pB.cx}" y2="${pB.cy}" />`
      }).join('')

      // districts
      el.gDistricts.innerHTML = blocks.map(d=>{
        const vis = matches(d)
        const c = colorFor(d.danger)
        return `
        <g class="district ${d.id===selected?'sel':''}" data-id="${d.id}" style="display:${vis?'block':'none'}">
          <rect x="${d.x}" y="${d.y}" rx="14" ry="14" width="${d.w}" height="${d.h}" fill="${c}" fill-opacity="0.2" stroke="${c}" stroke-opacity="0.8" stroke-width="2"/>
          <text class="label" x="${d.x+10}" y="${d.y+22}">${escapeHtml(d.name)}</text>
          <text class="sub" x="${d.x+10}" y="${d.y+d.h-10}">D${d.danger} • ${escapeHtml(d.tags.slice(0,3).join(', '))}</text>
        </g>`
      }).join('')

      // events
      el.gDistricts.querySelectorAll('.district').forEach(n=>{
        const id = n.getAttribute('data-id')
        const d = blocks.find(x=>x.id===id)
        n.addEventListener('mouseenter', e=>showTip(d,e))
        n.addEventListener('mouseleave', hideTip)
        n.addEventListener('mousemove', moveTip)
        n.addEventListener('click', ()=> select(d.id))
      })
    }
    function showTip(d, evt){
      el.tooltip.style.display='block'
      el.tooltip.textContent = `${d.name} — D${d.danger}`
      moveTip(evt)
    }
    function moveTip(evt){
      el.tooltip.style.left = (evt.clientX) + 'px'
      el.tooltip.style.top = (evt.clientY - 12) + 'px'
    }
    function hideTip(){ el.tooltip.style.display='none' }

    function select(id){
      selected = id
      const d = blocks.find(x=>x.id===id)
      renderMap()
      el.loreBody.textContent = `${d.blurb}\n\nHooks:\n- ${d.hooks[0]}\n- ${d.hooks[1]}\n- ${d.hooks[2]}`
      el.loreMeta.innerHTML = `
        <div class="inline" style="gap:6px;flex-wrap:wrap">
          <span class="pill hl">${escapeHtml(d.name)}</span>
          <span class="pill">D${d.danger}</span>
          ${d.tags.map(t=>`<span class="pill">#${escapeHtml(t)}</span>`).join('')}
        </div>
      `
      el.toGigs.disabled=false; el.newGig.disabled=false; el.toChars.disabled=false
    }

    // Actions
    el.toGigs.addEventListener('click', ()=>{
      if(!selected) return
      const d = blocks.find(x=>x.id===selected)
      history.replaceState(null,'','#gigs'); setView('#gigs')
      const i = document.getElementById('g-q'); i.value = d.name; i.dispatchEvent(new Event('input', {bubbles:true}))
    })
    el.newGig.addEventListener('click', ()=>{
      if(!selected) return
      const d = blocks.find(x=>x.id===selected)
      history.replaceState(null,'','#gigs'); setView('#gigs')
      window.__gigs_prefill({ location: d.name, tags: d.tags })
    })
    el.toChars.addEventListener('click', ()=>{
      if(!selected) return
      const d = blocks.find(x=>x.id===selected)
      history.replaceState(null,'','#chars'); setView('#chars')
      window.__chars_search(d.name.split(' ')[0])
    })

    // Filters
    el.q.addEventListener('input', e=>{ filter.q=e.target.value; renderMap() })
    el.tag.addEventListener('change', e=>{ filter.tag=e.target.value; renderMap() })
    el.danger.addEventListener('change', e=>{ filter.danger=e.target.value; renderMap() })
    el.reset.addEventListener('click', ()=>{
      filter={q:'', tag:'', danger:''}
      el.q.value=''; el.tag.value=''; el.danger.value=''
      renderMap()
    })

    renderControls()
    renderMap()
  })()

  // Start
  setView()
  </script>
</body>
</html>
