Fitness Assessment JavaScript Code

Add this code to your index.html <script> section (before </script>):

=== PASTE THIS INTO YOUR SCRIPT ===

// CONSTANTS
const FA_METRICS = ['beep', 'burpees', 'situps', 'jsquats', 'agility', 'laps'];
const FA_KEY = 'gsc2_fa';
const FA_LABELS = {beep:'Beep Test',burpees:'Burpees',situps:'Sit-Ups',jsquats:'Jump Squats',agility:'Speed/Agility',laps:'Laps'};
const FA_ICONS = {beep:'🔊',burpees:'⬆️',situps:'📤',jsquats:'🦵',agility:'⚡',laps:'🏃'};
let currentMetric = null;

function initFitnessAssessment(){
  const type=document.getElementById('sType').value;
  if(type!=='Fitness Assessment')return;
  const presentIds=Object.keys(att).filter(id=>att[id]==='x');
  const presentPlayers=players.filter(p=>presentIds.includes(p.id));
  const faSession={date:document.getElementById('sDate').value,presentPlayers:presentPlayers.map(p=>({id:p.id,num:p.num,name:p.name})),metrics:{},playerOfSession:null};
  FA_METRICS.forEach(metric=>{faSession.metrics[metric]={done:false,scores:{}};presentPlayers.forEach(p=>{faSession.metrics[metric].scores[p.id]=null;});});
  save(FA_KEY,faSession);
  renderFitnessMetrics();
}

function renderFitnessMetrics(){
  const container=document.getElementById('faMetricsGrid');
  if(!container)return;
  container.innerHTML='';
  const fa=load(FA_KEY,{});
  FA_METRICS.forEach(metric=>{
    const tile=document.createElement('div');
    tile.className='fa-metric-tile';
    tile.dataset.metric=metric;
    tile.onclick=()=>activateMetric(metric);
    if(fa.metrics?.[metric]?.done)tile.classList.add('done');
    tile.innerHTML=`<div class="fa-tile-icon">${FA_ICONS[metric]}</div><div class="fa-tile-label">${FA_LABELS[metric]}</div><div class="fa-tile-status">${fa.metrics?.[metric]?.done?'✓':''}</div>`;
    container.appendChild(tile);
  });
  updateFitnessProgress();
}

function activateMetric(metric){
  currentMetric=metric;
  const fa=load(FA_KEY,{});
  const panel=document.getElementById('faActivePanel');
  if(!panel)return;
  panel.style.display='block';
  document.getElementById('faPanelTitle').textContent=FA_LABELS[metric];
  const playerScoresDiv=document.getElementById('faPlayerScores');
  if(!playerScoresDiv)return;
  playerScoresDiv.innerHTML='';
  fa.presentPlayers?.forEach(player=>{
    const currentScore=fa.metrics?.[metric]?.scores?.[player.id]??null;
    const row=document.createElement('div');
    row.className='fa-score-row';
    if(currentScore!==null&&currentScore!=='')row.classList.add('filled');
    row.innerHTML=`<div class="fa-score-player"><span class="fa-pnum">${player.num}</span><span class="fa-pname">${player.name}</span></div><input type="number" class="fa-score-input" data-player-id="${player.id}" value="${currentScore!==null?currentScore:''}" placeholder="0" step="0.1" min="0" oninput="updateMetricProgress()" />`;
    playerScoresDiv.appendChild(row);
  });
  updateMetricProgress();
}

function updateMetricProgress(){
  const inputs=document.querySelectorAll('.fa-score-input');
  const filled=Array.from(inputs).filter(i=>i.value.trim()!=='').length;
  const total=inputs.length;
  const progressDiv=document.getElementById('faPanelProgress');
  if(progressDiv)progressDiv.textContent=`${filled}/${total}`;
  inputs.forEach(input=>{const row=input.closest('.fa-score-row');if(row){if(input.value.trim()!=='')row.classList.add('filled');else row.classList.remove('filled');}});
}

function saveMetricScores(){
  const fa=load(FA_KEY,{});
  const inputs=document.querySelectorAll('.fa-score-input');
  inputs.forEach(input=>{const playerId=input.dataset.playerId;const score=input.value?parseFloat(input.value):null;fa.metrics[currentMetric].scores[playerId]=score;});
  fa.metrics[currentMetric].done=true;
  save(FA_KEY,fa);
  const today=document.getElementById('sDate').value;
  fa.presentPlayers?.forEach(player=>{const playerObj=players.find(p=>p.id===player.id);if(playerObj){let entry=playerObj.fitness.find(f=>f.date===today);if(!entry){entry={date:today,beep:null,burpees:null,situps:null,jsquats:null,agility:null,laps:null,notes:'Fitness Assessment'};playerObj.fitness.push(entry);}const score=fa.metrics[currentMetric].scores[player.id];entry[currentMetric]=score;}});
  save(K.PL,players);
  document.getElementById('faActivePanel').style.display='none';
  const savedMetric=currentMetric;
  currentMetric=null;
  renderFitnessMetrics();
  showToast(`✓ ${FA_LABELS[savedMetric]} scores saved`);
  checkFitnessComplete();
}

function cancelMetric(){
  document.getElementById('faActivePanel').style.display='none';
  currentMetric=null;
}

function updateFitnessProgress(){
  const fa=load(FA_KEY,{});
  const completed=FA_METRICS.filter(m=>fa.metrics?.[m]?.done).length;
  const progressDiv=document.getElementById('faProgress');
  if(progressDiv)progressDiv.textContent=`${completed}/6`;
}

function checkFitnessComplete(){
  const fa=load(FA_KEY,{});
  if(FA_METRICS.every(m=>fa.metrics?.[m]?.done)){
    showToast('🎉 All metrics complete! Select Player of Session');
    const motmSection=document.getElementById('motmSection');
    if(motmSection)setTimeout(()=>{motmSection.scrollIntoView({behavior:'smooth'});},500);
  }
}

const originalSaveSession=saveSession;
function saveSession(){
  const type=document.getElementById('sType').value;
  if(type==='Fitness Assessment'){saveFitnessSession();return;}
  originalSaveSession();
}

function saveFitnessSession(){
  const fa=load(FA_KEY,{});
  if(!FA_METRICS.every(m=>fa.metrics?.[m]?.done)){showToast('⚠️ Complete all 6 metrics first');return;}
  const historyEntry={id:`${Date.now()}-FA`,date:fa.date||document.getElementById('sDate').value,type:'Fitness Assessment',opp:'',present:fa.presentPlayers,absent:[],late:[],injured:[],motm:motm||null,fitness:fa.metrics,tags:[],notes:document.getElementById('attNotes')?.value||'',timestamp:new Date().toISOString()};
  history.push(historyEntry);
  save(K.HI,history);
  localStorage.removeItem(FA_KEY);
  att={};scorers=[];motm=null;tags=[];
  showToast('✓ Fitness Assessment saved!');
  setTimeout(()=>showHome(),800);
}

=== END PASTE ===

Also add this to toggleGameFields() function after the fitness section line:
  if(isFitness) initFitnessAssessment();

And update your faSection HTML to include:
  <div class="fa-metrics-grid" id="faMetricsGrid"></div>
  <div class="fa-active-panel" id="faActivePanel" style="display:none">
    <div class="fa-panel-hdr">
      <div class="fa-metric-title" id="faPanelTitle">Beep Test</div>
      <div class="fa-panel-progress" id="faPanelProgress">0/6</div>
    </div>
    <div class="fa-player-scores" id="faPlayerScores"></div>
    <div class="fa-panel-actions">
      <button class="fa-save-btn" onclick="saveMetricScores()">💾 Save</button>
      <button class="fa-cancel-btn" onclick="cancelMetric()">✕ Cancel</button>
    </div>
  </div>
