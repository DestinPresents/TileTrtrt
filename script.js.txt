const $=s=>document.querySelector(s);
const icons=["🍓","🌸","🍄","🎈","🍊","🍧","🍉","🥝","🌺","🍋","🍀","🧁","💎","👑","🧿","🌙","⭐","🦋","🐚","🌈","🍒","🫐","🥥","🌻","🌷","🍀","🪷","🍭","🧸","🎁","☀️","🪄","🦄","🐝","🍪","🫧","🌟","🍑","🍇","🥭"];
const screens={home:$("#home"),map:$("#map"),game:$("#game")};
function numStore(key, fallback, min=0, max=999999){
  const n=Number(localStorage.getItem(key));
  return Number.isFinite(n)?Math.min(max,Math.max(min,n)):fallback;
}
function loadPowerups(){
  try{
    const p=JSON.parse(localStorage.getItem("tt_pu")||"{}");
    return {undo:numStoreFrom(p.undo,3),shuffle:numStoreFrom(p.shuffle,3),hint:numStoreFrom(p.hint,3)};
  }catch{return {undo:3,shuffle:3,hint:3};}
}
function numStoreFrom(v,f){const n=Number(v);return Number.isFinite(n)&&n>=0?Math.floor(n):f;}
const MAX_LEVEL=3000;
const GAME_NAME="Tile Trails";
const DEVELOPER="Destin Studios";

// Firebase account/cloud-save setup. Guest mode deliberately stays local-only.
const firebaseConfig={
  apiKey:"AIzaSyBMaB2v9lbm9R2m4Nv31hRT6vImaenOQEc",
  authDomain:"tiles-96c54.firebaseapp.com",
  databaseURL:"https://tiles-96c54-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId:"tiles-96c54",
  storageBucket:"tiles-96c54.firebasestorage.app",
  messagingSenderId:"511133312503",
  appId:"1:511133312503:web:7c8735404949fef93f3af2"
};
let firebaseReady=false,auth=null,db=null,currentUser=null,authBusy=false,cloudHydrated=false,cloudSaveTimer=null;
let confirmationResult=null,recaptchaVerifier=null,authPhonePending=false,authPhoneName="";
try{
  if(window.firebase){
    firebase.initializeApp(firebaseConfig);
    auth=firebase.auth(); db=firebase.database(); firebaseReady=true;
    auth.setPersistence(firebase.auth.Auth.Persistence.LOCAL);
    auth.onAuthStateChanged(async user=>{
      currentUser=user||null;
      updateAccountButton();
      if(user) await loadCloudState(user);
    });
  }
}catch(e){console.error("Firebase init error",e)}
const state={
 level:numStore("tt_level",21,1,MAX_LEVEL),
 coins:numStore("tt_coins",340,0,999999999),
 streak:numStore("tt_streak",3,0,9999),
 lastDaily:localStorage.getItem("tt_daily")||"",
 pu:loadPowerups(),
 sound:localStorage.getItem("tt_sound")!=="0",
 music:localStorage.getItem("tt_music")!=="0",
 extraSlot:false
};
let board=[],tray=[],history=[],matched=0,target=36,time=60,timerId=null,levelSeed=0,pickBusy=false,gameOver=false,pendingPickTimer=null,matchTimer=null,activeLevel=state.level;
let traySpecial=[];
let audioCtx=null,musicTimer=null,comboCount=0,matchEpoch=0;
function save(){
  localStorage.setItem("tt_last_save",String(Date.now()));
  localStorage.setItem("tt_level",state.level);localStorage.setItem("tt_coins",state.coins);localStorage.setItem("tt_streak",state.streak);localStorage.setItem("tt_daily",state.lastDaily);localStorage.setItem("tt_pu",JSON.stringify(state.pu));localStorage.setItem("tt_sound",state.sound?"1":"0");localStorage.setItem("tt_music",state.music?"1":"0");localStorage.setItem("tt_extra_slot",state.extraSlot?"1":"0");
  cloudSave();
}
function cloudPayload(){return {level:state.level,coins:state.coins,streak:state.streak,lastDaily:state.lastDaily,pu:state.pu,sound:state.sound,music:state.music,updatedAt:Date.now()}}
function cloudSave(){
  if(!cloudHydrated||!firebaseReady||!currentUser||currentUser.isAnonymous)return;
  clearTimeout(cloudSaveTimer);
  cloudSaveTimer=setTimeout(()=>db.ref("users/"+currentUser.uid).update(cloudPayload()).catch(e=>console.error("Cloud save failed",e)),180);
}
async function loadCloudState(user){
  try{
    const snap=await db.ref("users/"+user.uid).once("value");
    const d=snap.exists()?snap.val():null;
    const localStamp=Number(localStorage.getItem("tt_last_save")||0);
    const cloudStamp=d?Number(d.updatedAt||0):0;
    if(d&&cloudStamp>localStamp){
      if(Number.isFinite(Number(d.level)))state.level=Math.min(MAX_LEVEL,Math.max(1,Number(d.level)));
      if(Number.isFinite(Number(d.coins)))state.coins=Math.max(0,Number(d.coins));
      if(Number.isFinite(Number(d.streak)))state.streak=Math.max(0,Number(d.streak));
      if(typeof d.lastDaily==="string")state.lastDaily=d.lastDaily;
      if(d.pu)state.pu={undo:numStoreFrom(d.pu.undo,3),shuffle:numStoreFrom(d.pu.shuffle,3),hint:numStoreFrom(d.pu.hint,3)};
      if(typeof d.sound==="boolean")state.sound=d.sound;
      if(typeof d.music==="boolean")state.music=d.music;
      localStorage.setItem("tt_level",state.level);localStorage.setItem("tt_coins",state.coins);localStorage.setItem("tt_streak",state.streak);localStorage.setItem("tt_daily",state.lastDaily);localStorage.setItem("tt_pu",JSON.stringify(state.pu));localStorage.setItem("tt_sound",state.sound?"1":"0");localStorage.setItem("tt_music",state.music?"1":"0");localStorage.setItem("tt_last_save",String(cloudStamp));
    }else if(!d||localStamp>=cloudStamp){
      await db.ref("users/"+user.uid).set(cloudPayload());
    }
    cloudHydrated=true;
    updateCoins();updatePowerups();buildPreview();buildMap();
    if(document.querySelector('#home')?.classList.contains('active')) buildPreview();
  }catch(e){cloudHydrated=true;console.error("Cloud load failed",e);showToast("CLOUD SAVE UNAVAILABLE","bad")}
}
function updateAccountButton(){const b=$("#accountBtn");if(!b)return;b.textContent=currentUser?"✓":"👤";b.title=currentUser?(currentUser.displayName||currentUser.email||"Account"):"Login"}
function show(n){if(n!=="game")stopMusic();Object.values(screens).forEach(x=>x.classList.remove("active"));screens[n].classList.add("active")}
function todayKey(){const d=new Date();const y=d.getFullYear(),m=String(d.getMonth()+1).padStart(2,"0"),day=String(d.getDate()).padStart(2,"0");return `${y}-${m}-${day}`}
function dailyAvailable(){return state.lastDaily!==todayKey()}
function updateCoins(){ $("#coinCount").textContent=state.coins.toLocaleString();$("#homeLevel").textContent=state.level;$("#playLevel").textContent=state.level;$("#gameLevel").textContent=activeLevel;$("#streakText").textContent=state.streak+" days"; const a=dailyAvailable();$("#dailyBtn").textContent=a?"🪙 CLAIM":"✓ CLAIMED";$("#dailyBtn").disabled=!a;$("#dailyBtn").classList.toggle("claimed",!a);}
function claimDaily(){if(!dailyAvailable())return;
  const last=state.lastDaily; const now=todayKey();
  if(last){const a=new Date(last),b=new Date(now);const days=Math.round((b-a)/86400000);state.streak=days===1?state.streak+1:1;}else state.streak=1;
  const reward=50;state.coins+=reward;state.lastDaily=now;save();updateCoins();
  modal(`<div class="reward-icon">🪙</div><h2>Daily Reward</h2><p>Today's reward has been added to your wallet.</p><div class="big-reward">+50 Coins</div><button class="modal-action" onclick="closeModal()">AWESOME!</button>`)}
function applyTheme(level){
  const themes=[
    [1,"tropical","🌴 TROPICAL ISLAND"],[101,"lagoon","🌊 BLUE LAGOON"],[201,"garden","🌺 FLOWER GARDEN"],[301,"forest","🌲 MYSTIC FOREST"],[401,"snow","❄️ SNOW VALLEY"],[501,"fantasy","✨ FANTASY REALM"]
  ];
  let theme=themes[0]; for(const t of themes)if(level>=t[0])theme=t;
  $("#app").dataset.theme=theme[1];
  const el=$(".game-level"); if(el)el.dataset.themeName=theme[2];
}
function buildPreview(){const p=$("#previewGrid");p.innerHTML="";const start=(state.level*3)%icons.length;for(let i=0;i<12;i++){const d=document.createElement("div");d.textContent=icons[(start+i)%icons.length];p.appendChild(d)}}
function buildMap(){const m=$("#mapPath");m.innerHTML="";const frag=document.createDocumentFragment();for(let n=1;n<=MAX_LEVEL;n++){const b=document.createElement("button");b.className="map-node "+(n<state.level?"done ":"")+(n===state.level?"current ":"")+(n>state.level?"locked":"");b.textContent=n>state.level?"🔒 "+n:(n<state.level?"✓ "+n:n);if(n<=state.level)b.onclick=()=>startLevel(n);frag.appendChild(b)}m.appendChild(frag)}
function levelTarget(level){if(level<=25)return 24;if(level<=50)return 30;if(level<=75)return 36;if(level<=100)return 45;return Math.min(180,45+Math.floor((level-100)/25)*6)}
function levelTime(level){if(level<100)return 90;if(level<200)return 120;if(level<500)return 150;if(level<1000)return 180;return 240}
function winReward(level){return 100+Math.floor((Math.max(1,level)-1)/50)*10}
function trayCapacity(){return state.extraSlot?7:6}
function startLevel(level=state.level){activeLevel=Math.min(MAX_LEVEL,Math.max(1,Number(level)||1));state.extraSlot=false;clearInterval(timerId);stopMusic();matchEpoch++;gameOver=false;pickBusy=false;show("game");applyTheme(activeLevel);levelSeed=activeLevel;target=levelTarget(activeLevel);matched=0;tray=[];traySpecial=[];history=[];time=levelTime(activeLevel);comboCount=0;$("#target").textContent=target;$("#matched").textContent=0;$("#time").textContent=time;$("#progress").style.width="0%";updatePowerups();generateSolvableLevel();renderTray();startMusic();clearInterval(timerId);timerId=setInterval(()=>{time--;$("#time").textContent=time;if(time<=0){clearInterval(timerId);lose("⏰ Time's Up!")}},1000)}
function shuffle(a){return a.sort(()=>Math.random()-.5)}
function generateSolvableLevel(){
  const el=$("#board"); el.innerHTML=""; board=[];
  const w=Math.max(280,el.clientWidth||390), h=Math.max(320,el.clientHeight||420);
  const groups=Math.ceil(target/3), tileW=57, tileH=57, rand=()=>Math.random();
  const clamp=(v,min,max)=>Math.max(min,Math.min(max,v));
  // Early levels intentionally use a light, readable pile. As levels rise,
  // spacing tightens and the pile gains more depth/overlap.
  const density=activeLevel<=50?0.82:activeLevel<=100?0.9:activeLevel<=200?0.96:1.02;
  const stepX=Math.max(48,Math.round(57*density));
  const stepY=Math.max(46,Math.round(57*density));
  const cols=Math.max(4,Math.floor((w-22)/stepX));
  const rows=Math.max(5,Math.floor((h-22)/stepY));
  const cx=w/2,cy=h/2, positions=[];
  for(let r=0;r<rows;r++) for(let c=0;c<cols;c++){
    const baseX=8+c*stepX+(r%2?stepX*.48:0);
    const baseY=7+r*stepY;
    const radial=Math.min(1,Math.hypot(baseX-cx,baseY-cy)/(Math.max(w,h)*.72));
    const spread=1+radial*(activeLevel>100?.18:.08);
    const jitter=activeLevel<=50?7:activeLevel<=100?10:15;
    positions.push({x:clamp(cx+(baseX-cx)*spread+(rand()-.5)*jitter,2,w-tileW-2),y:clamp(cy+(baseY-cy)*spread+(rand()-.5)*jitter,2,h-tileH-2)});
  }
  shuffle(positions);
  const all=[];
  for(let g=0;g<groups;g++){
    const icon=icons[Math.floor(rand()*icons.length)];
    for(let k=0;k<3;k++){
      const base=positions[(g*3+k)%positions.length];
      const spread=activeLevel<=50?9:activeLevel<=100?15:26;
      const q={x:clamp(base.x+(rand()-.5)*spread,2,w-tileW-2),y:clamp(base.y+(rand()-.5)*spread,2,h-tileH-2)};
      all.push({id:all.length,icon,x:q.x,y:q.y,z:0,group:g,removed:false,special:(activeLevel>=200&&k===0&&g%7===0)?'gold':null});
    }
  }
  shuffle(all);
  all.forEach((t,i)=>{t.z=i+1;t.layer=Math.floor(i/Math.max(1,Math.ceil(all.length/(activeLevel<=50?3:activeLevel<=100?4:5))))});
  board=all; renderBoard(); renderTray();
}
function overlaps(a,b){return Math.abs(a.x-b.x)<48&&Math.abs(a.y-b.y)<48}
function isBlocked(t){return board.some(o=>!o.removed&&o.id!==t.id&&o.z>t.z&&overlaps(t,o))}
function renderBoard(){const el=$("#board");el.querySelectorAll(".tile").forEach(x=>x.remove());board.filter(t=>!t.removed).forEach(t=>{const b=document.createElement("button");b.className="tile";b.dataset.id=t.id;b.style.left=t.x+"px";b.style.top=t.y+"px";b.style.setProperty("--z",t.z);b.style.zIndex=t.z; b.style.setProperty("--dx",((t.id*17)%7-3)+"px");if(isBlocked(t))b.classList.add("blocked");else b.setAttribute("aria-label","Select "+t.icon);if(t.special)b.classList.add("special-"+t.special);b.textContent=t.icon;el.appendChild(b)})}
function record(){history.push({board:board.map(t=>({...t})),tray:[...tray],traySpecial:[...traySpecial],matched})}
function playSound(type="click"){
  if(!state.sound)return;
  try{
    audioCtx=audioCtx||new (window.AudioContext||window.webkitAudioContext)();
    if(audioCtx.state==='suspended')audioCtx.resume();
    const now=audioCtx.currentTime;
    const o=audioCtx.createOscillator(),g=audioCtx.createGain();o.connect(g);g.connect(audioCtx.destination);
    const cfg={click:[580,.11,'sine',.24],match:[760,.18,'triangle',.34],match2:[1120,.24,'sine',.28],buy:[650,.14,'sine',.22],win:[680,.26,'triangle',.28],lose:[170,.30,'sawtooth',.20]}[type]||[560,.12,'sine',.22];
    o.type=cfg[2];o.frequency.setValueAtTime(cfg[0],now);
    if(type==='match'||type==='win')o.frequency.exponentialRampToValueAtTime(cfg[0]*1.42,now+cfg[1]);
    g.gain.setValueAtTime(cfg[3],now);g.gain.exponentialRampToValueAtTime(.001,now+cfg[1]);
    o.start(now);o.stop(now+cfg[1]+.03);
    if(type==='match'){setTimeout(()=>playSound('match2'),90)}
  }catch{}
}
function getBgMusic(){return $("#bgMusic")}
function loadCustomMusic(){const a=getBgMusic();return !!(a&&a.src)}
function startMusic(){
  if(!state.music||musicTimer)return;
  const audio=getBgMusic();
  if(audio&&audio.src){
    audio.volume=.55;
    const p=audio.play();
    if(p&&p.catch)p.catch(()=>{});
    musicTimer="audio";
    return;
  }
  const notes=[261.63,329.63,392,523.25,392,329.63]; let i=0;
  const tick=()=>{if(!state.music)return;try{audioCtx=audioCtx||new (window.AudioContext||window.webkitAudioContext)();if(audioCtx.state==='suspended')audioCtx.resume();const o=audioCtx.createOscillator(),g=audioCtx.createGain();o.type='sine';o.frequency.value=notes[i++%notes.length];g.gain.value=.022;o.connect(g);g.connect(audioCtx.destination);const n=audioCtx.currentTime;g.gain.exponentialRampToValueAtTime(.001,n+.35);o.start(n);o.stop(n+.38)}catch{}};
  tick(); musicTimer=setInterval(tick,900);
}
function stopMusic(){const audio=getBgMusic();if(audio){try{audio.pause();audio.currentTime=0}catch{}};if(musicTimer&&musicTimer!=="audio")clearInterval(musicTimer);musicTimer=null}
function vibrate(pattern=[18]){if(navigator.vibrate)try{navigator.vibrate(pattern)}catch{}}
function showToast(text,kind='good'){
  const old=document.querySelector('.game-toast');if(old)old.remove();
  const d=document.createElement('div');d.className='game-toast '+kind;d.textContent=text;
  document.body.appendChild(d);requestAnimationFrame(()=>d.classList.add('show'));
  setTimeout(()=>{d.classList.remove('show');setTimeout(()=>d.remove(),220)},900);
}
function flyToTray(tileEl){
  const slot=document.querySelector(`#tray .slot:not(.locked-slot):nth-child(${Math.min(7,tray.length)})`)||document.querySelector('#tray .slot:not(.locked-slot)');
  if(!tileEl||!slot)return;
  const a=tileEl.getBoundingClientRect(),b=slot.getBoundingClientRect();
  const c=tileEl.cloneNode(true);c.className='tile-fly';c.textContent=tileEl.textContent;
  c.style.left=a.left+'px';c.style.top=a.top+'px';c.style.width=a.width+'px';c.style.height=a.height+'px';
  document.body.appendChild(c);
  requestAnimationFrame(()=>{c.style.transform=`translate(${b.left-a.left+b.width*.1}px,${b.top-a.top}px) scale(.62) rotate(8deg)`;c.style.opacity='0.2'});
  setTimeout(()=>c.remove(),360);
}
function pick(id){
  if(gameOver||pickBusy)return;
  const t=board.find(x=>x.id===id);
  if(!t||t.removed||isBlocked(t))return;
  if(tray.length>=trayCapacity()){gameOver=true;clearInterval(timerId);lose("💥 Tray Full!");return;}
  pickBusy=true; record(); const myEpoch=matchEpoch;
  const source=document.querySelector(`[data-id="${id}"]`);
  t.removed=true; tray.push(t.icon); traySpecial.push(!!t.special);
  playSound('click'); vibrate([12]); flyToTray(source);
  if(source)source.classList.add('pop');
  clearTimeout(pendingPickTimer);
  pendingPickTimer=setTimeout(()=>{
    pendingPickTimer=null;
    const groupsToClear=[];
    for(const icon of [...new Set(tray)]){
      const inds=tray.map((x,i)=>x===icon?i:-1).filter(i=>i>=0);
      if(inds.length>=3)groupsToClear.push({icon,inds:inds.slice(0,3)});
    }
    if(groupsToClear.length){
      renderTray();
      groupsToClear.forEach((m,idx)=>{
        const slots=m.inds;
        slots.forEach(i=>{const el=document.querySelector(`#tray .slot:nth-child(${i+1})`);if(el)el.classList.add('match-burst')});
        setTimeout(()=>{
          if(myEpoch!==matchEpoch||gameOver)return;
          const goldBonus=m.inds.some(i=>traySpecial[i])?15:0;
          tray=tray.filter((_,i)=>!slots.includes(i));
          traySpecial=traySpecial.filter((_,i)=>!slots.includes(i));
          matched+=3;state.coins+=3+goldBonus;comboCount++;save();
          const comboWord=goldBonus?`GOLDEN +${goldBonus}!`:comboCount>=4?'PERFECT COMBO!':comboCount===3?'BRILLIANT!':comboCount===2?'AMAZING!':'EXCELLENT!';
          showToast(comboWord,'good');
          playSound('match');vibrate([22,30,22]);
          renderTray();renderBoard();updateProgress();
          if(matched>=target){gameOver=true;clearInterval(timerId);win()}
        },420+idx*80);
      });
      matchTimer=setTimeout(()=>{pickBusy=false;},520+groupsToClear.length*80);
    }else{
      save();renderTray();renderBoard();updateProgress();pickBusy=false;
      if(tray.length>=trayCapacity()){gameOver=true;clearInterval(timerId);lose('💥 Tray Full!')}
    }
  },140);
}
function renderTray(){
  const t=$("#tray");t.innerHTML="";
  for(let i=0;i<7;i++){
    const s=document.createElement("div");
    if(i===6&&!state.extraSlot){
      s.className="slot locked-slot";
      s.innerHTML='<span class="slot-plus">＋</span><small>20</small>';
      s.title="Buy extra slot for 20 coins";
      s.onclick=buyExtraSlot;
    }else{
      s.className="slot"+(tray[i]?" filled":"");s.textContent=tray[i]||"";
    }
    t.appendChild(s);
  }
}
function updateProgress(){$("#matched").textContent=matched;$("#progress").style.width=Math.min(100,matched/target*100)+"%";updatePowerups();updateCoins()}
function updatePowerups(){const prices={undo:50,shuffle:100,hint:50};["undo","shuffle","hint"].forEach(k=>{const b=$("#"+k),c=b.querySelector("b"),s=b.querySelector("small");if(state.pu[k]>0){c.textContent=state.pu[k];s.textContent=k.toUpperCase();b.classList.remove("buy-mode")}else{c.textContent="🪙"+prices[k];s.textContent="BUY";b.classList.add("buy-mode")}})}
function starRating(){
  const trayPenalty=Math.max(0,tray.length-2);
  const timeRatio=time/Math.max(1,levelTime(activeLevel));
  if(timeRatio>.55&&trayPenalty<=1)return 3;
  if(timeRatio>.25)return 2;
  return 1;
}
function milestoneBonus(level){
  if(level%100===0)return 500;
  if(level%50===0)return 250;
  if(level%25===0)return 150;
  return 0;
}
function chestBonus(level){return level%10===0?100+Math.floor(level/100)*25:0}
function redeemCode(){
  const input=$("#redeemInput"); if(!input)return;
  const code=input.value.trim().toLowerCase();
  const rewards={"adi7s":100000,"somu34":3400};
  const used=JSON.parse(localStorage.getItem("tt_redeemed")||"{}");
  if(!rewards[code]){showToast("INVALID CODE","bad");return;}
  if(used[code]){showToast("CODE ALREADY USED","bad");return;}
  state.coins+=rewards[code];used[code]=1;localStorage.setItem("tt_redeemed",JSON.stringify(used));save();updateCoins();playSound("buy");vibrate([20,30,20]);
  showToast("+"+rewards[code].toLocaleString()+" COINS","buy");
  input.value="";
}
function howToPlay(){modal(`<div class="settings-head"><div class="settings-icon">❓</div><div><div class="eyebrow">QUICK GUIDE</div><h2>HOW TO PLAY</h2></div></div><div class="howto-list"><div>1️⃣ Tap only tiles that are not covered by another tile.</div><div>2️⃣ Selected tiles move into the tray at the bottom.</div><div>3️⃣ Collect 3 identical tiles to make a match; the three tiles then disappear.</div><div>4️⃣ You normally have 6 tray slots. The 7th slot can be unlocked for 20 coins for that level only.</div><div>5️⃣ If the tray fills before you clear the board, the level is lost.</div><div>6️⃣ Use Undo to reverse a move and Shuffle to rearrange the remaining tiles.</div><div>7️⃣ <b>Auto Shot</b> automatically removes one random matching group of 3 tiles from the board.</div><div>8️⃣ Finish the level before the timer reaches zero and collect your coin reward.</div></div><button class="modal-action" onclick="closeModal()">GOT IT!</button>`)}
function levelIntro(level){
  modal(`<div class="level-intro"><div class="intro-badge">LEVEL ${level}</div><h2>Ready to Explore?</h2><p>🧩 ${target} tiles &nbsp; • &nbsp; ⏱️ ${time}s</p><button class="modal-action" onclick="closeModal();playSound('click');vibrate([18])">PLAY NOW <span>▶</span></button></div>`);
}
function win(){
  stopMusic(); gameOver=true;clearInterval(timerId);playSound('win');vibrate([35,45,70]);
  const completed=activeLevel;
  const baseReward=winReward(completed);
  const milestone=milestoneBonus(completed);
  const chest=chestBonus(completed);
  const reward=baseReward+milestone+chest;
  const stars=starRating();
  state.coins+=reward;
  const next=Math.min(MAX_LEVEL,completed+1);
  if(completed<MAX_LEVEL)state.level=Math.max(state.level,next);
  save();updateCoins();buildMap();
  const chestHtml=chest?`<div class="chest-reward">🎁 Treasure Chest +${chest}</div>`:'';
  const milestoneHtml=milestone?`<div class="milestone-reward">🏅 Milestone Bonus +${milestone}</div>`:'';
  const nextButton=completed<MAX_LEVEL?`<button class="modal-action" onclick="closeModal();startLevel(${next})">NEXT LEVEL <span>→</span></button>`:`<button class="modal-action" onclick="closeModal();show('home')">FINISH <span>✓</span></button>`;
  modal(`<div class="result-burst win-burst">🏆</div><div class="result-label">LEVEL CLEARED</div><h2>Brilliant!</h2><div class="stars">${'★'.repeat(stars)}${'☆'.repeat(3-stars)}</div><p>Great matching. Your reward is ready.</p><div class="big-reward">🪙 +${reward} Coins</div>${chestHtml}${milestoneHtml}<div class="victory-choice">CHOOSE YOUR NEXT STEP</div>${nextButton}<button class="secondary-action" onclick="closeModal();show('home')">⌂ HOME</button>`);
}
function lose(title){
  stopMusic(); gameOver=true;clearInterval(timerId);playSound('lose');vibrate([80,40,80]);
  modal(`<div class="result-burst lose-burst">💫</div><div class="result-label">RUN ENDED</div><h2>${title}</h2><p>One more smart attempt and you can clear it.</p><button class="modal-action" onclick="closeModal();startLevel(activeLevel)">TRY AGAIN</button><button class="secondary-action" onclick="closeModal();show('home')">⌂ HOME</button>`);
}
function modal(html){$("#modalBody").innerHTML=html;$("#modal").classList.remove("hidden");const card=$("#modal .modal-card");card.classList.toggle("shop-card-modal",!!html.includes("shop-hero"))}
function closeModal(){$("#modal").classList.add("hidden")}
function useUndo(){
  if(gameOver)return;
  if(!state.pu.undo){buyPowerup("undo",50);return;}
  if(!history.length){showToast("NOTHING TO UNDO","bad");return;}
  clearTimeout(pendingPickTimer);
  clearTimeout(matchTimer);
  pendingPickTimer=null;matchTimer=null;
  pickBusy=false;
  gameOver=false;
  state.pu.undo--;playSound("click");vibrate([25]);
  const h=history.pop();
  board=h.board.map(t=>({...t}));
  tray=[...h.tray];
  traySpecial=[...(h.traySpecial||[])];
  matched=h.matched;
  save();
  renderTray();
  renderBoard();
  updateProgress();
  // Keep the countdown alive after an undo instead of leaving the board in a
  // locked/intermediate state.
  clearInterval(timerId);
  timerId=setInterval(()=>{
    time--; $("#time").textContent=time;
    if(time<=0){clearInterval(timerId);lose("⏰ Time's Up!");}
  },1000);
}
function useShuffle(){if(gameOver)return;if(!state.pu.shuffle){buyPowerup("shuffle",100);return}record();state.pu.shuffle--;const alive=board.filter(t=>!t.removed),positions=alive.map(t=>({x:t.x,y:t.y}));shuffle(positions);alive.forEach((t,i)=>{t.x=positions[i].x;t.y=positions[i].y});save();renderBoard();updatePowerups();playSound("buy");vibrate([16,25]);showToast("BOARD SHUFFLED","good")}
function useHint(){
  if(gameOver)return;
  if(!state.pu.hint){buyPowerup("hint",50);return;}
  const alive=board.filter(t=>!t.removed);
  const byIcon={};
  alive.forEach(t=>(byIcon[t.icon]??=[]).push(t));
  const groups=Object.values(byIcon).filter(g=>g.length>=3);
  if(!groups.length){showToast("NO AUTO SHOT AVAILABLE","bad");return;}
  const accessibleGroups=groups.filter(g=>g.filter(t=>!isBlocked(t)).length>=3);
  const pool=accessibleGroups.length?accessibleGroups:groups;
  const group=pool[Math.floor(Math.random()*pool.length)];
  const shot=shuffle([...group]).slice(0,3);
  clearTimeout(pendingPickTimer);clearTimeout(matchTimer);pendingPickTimer=null;matchTimer=null;
  record();
  state.pu.hint--;
  const epoch=++matchEpoch;
  pickBusy=true;
  shot.forEach((t,i)=>{
    const el=document.querySelector(`[data-id="${t.id}"]`);
    if(el){el.classList.add("auto-shot");setTimeout(()=>{if(el)el.style.opacity="0"},i*100+180);}
  });
  setTimeout(()=>{
    if(epoch!==matchEpoch||gameOver)return;
    const goldBonus=shot.some(t=>t.special)?15:0;
    shot.forEach(t=>t.removed=true);
    matched+=3;comboCount++;state.coins+=3+goldBonus;save();
    const word=goldBonus?`AUTO SHOT +${15} GOLD!`:comboCount>=4?"PERFECT COMBO!":comboCount===3?"BRILLIANT!":comboCount===2?"AMAZING!":"AUTO SHOT!";
    showToast(word,"good");playSound("match");vibrate([22,30,22]);
    renderBoard();updateProgress();pickBusy=false;
    if(matched>=target){gameOver=true;clearInterval(timerId);win();}
  },520);
}
function buyExtraSlot(){
  if(state.extraSlot)return;
  if(state.coins<20){playSound('lose');vibrate([50]);showToast('NOT ENOUGH COINS','bad');return;}
  state.coins-=20;state.extraSlot=true;save();updateCoins();renderTray();playSound('buy');vibrate([16,25]);showToast('EXTRA SLOT UNLOCKED • 20 COINS','buy');openShop();
}
function buyPowerup(type,price){
  if(state.coins<price){playSound('lose');vibrate([50]);showToast('NOT ENOUGH COINS','bad');return;}
  state.coins-=price;state.pu[type]++;save();updateCoins();updatePowerups();playSound('buy');vibrate([16,25]);showToast(`${type.toUpperCase()} +1`,'buy');
  openShop();
}
function openShop(){modal(`<div class="shop-hero"><div class="shop-icon">🛒</div><div><div class="eyebrow">TILE TRAILS</div><h2>SHOP</h2><p>Use coins earned by playing.</p></div></div><div class="wallet"><span>YOUR COINS</span><strong>🪙 ${state.coins.toLocaleString()}</strong></div><div class="shop-title">EXTRA TRAY SLOT</div><div class="extra-slot-card ${state.extraSlot?'unlocked':''}"><div class="extra-slot-icon">＋</div><div><strong>${state.extraSlot?'Extra Slot Unlocked':'7th Slot'}</strong><small>${state.extraSlot?'You can now use all 7 tray spaces.':'Unlock the 7th tray space for 20 coins.'}</small></div>${state.extraSlot?'<span class="owned-badge">✓ OWNED</span>':'<button onclick="buyExtraSlot();openShop()">🪙 20</button>'}</div><div class="shop-title">POWER-UPS</div><div class="shop-cards"><div class="shop-card"><div class="shop-card-icon">↩</div><div><strong>Undo</strong><small>Restore your last move</small></div><button onclick="buyPowerup('undo',50)">🪙 50</button></div><div class="shop-card"><div class="shop-card-icon">🔀</div><div><strong>Shuffle</strong><small>Rearrange the board</small></div><button onclick="buyPowerup('shuffle',100)">🪙 100</button></div><div class="shop-card"><div class="shop-card-icon">🎯</div><div><strong>Auto Shot</strong><small>Automatically clears one matching triple</small></div><button onclick="buyPowerup('hint',50)">🪙 50</button></div></div><div class="shop-title">REDEEM CODE</div><div class="redeem-box"><input id="redeemInput" maxlength="20" placeholder="Enter code"><button onclick="redeemCode()">REDEEM</button></div><div class="redeem-note">Each code can be redeemed once on this device.</div><button class="secondary-action full" onclick="closeModal()">CLOSE</button>`)}

function openAbout(){modal(`<div class="settings-head"><div class="settings-icon">✦</div><div><div class="eyebrow">THE TEAM</div><h2>ABOUT US</h2></div></div><div class="about-copy"><h3>${GAME_NAME}</h3><p><b>Tile Trails</b> is a simple, relaxing and challenging tile-matching adventure built for players who love quick puzzles, colorful objects and satisfying triple matches.</p><p>Our goal is to create lightweight games that are easy to understand, fun to replay and enjoyable on both mobile and desktop devices.</p><div class="about-facts"><div><span>DEVELOPER</span><b>${DEVELOPER}</b></div><div><span>GAME</span><b>${GAME_NAME}</b></div><div><span>CO-FOUNDER</span><b>Rudra Pratap Singh</b></div><div><span>STYLE</span><b>Casual • Puzzle • Match 3</b></div></div></div><button class="secondary-action full" onclick="closeModal()">CLOSE</button>`)}
function openEvents(){
  modal(`<div class="events-head"><div class="events-icon">🎪</div><div><div class="eyebrow">LIMITED TIME</div><h2>EVENTS</h2></div></div>
  <div class="event-card"><div class="event-badge">SOON</div><h3>🌸 Flower Festival</h3><p>Special event levels and bonus coins are coming here.</p></div>
  <div class="event-card"><div class="event-badge">SOON</div><h3>🏆 Explorer Challenge</h3><p>Complete daily challenges and collect exclusive rewards.</p></div>
  <button class="secondary-action full" onclick="closeModal()">CLOSE</button>`);
}
function settings(){modal(`<div class="settings-head"><div class="settings-icon">⚙</div><div><div class="eyebrow">GAME</div><h2>SETTINGS</h2></div></div><div class="setting-row">🔊 Sound <button class="toggle ${state.sound?"":"off"}" id="soundToggle">${state.sound?"ON":"OFF"}</button></div><div class="setting-row">🎵 Music <button class="toggle ${state.music?"":"off"}" id="musicToggle">${state.music?"ON":"OFF"}</button></div><div class="setting-row">❓ How to Play <button class="link-btn" onclick="howToPlay()">VIEW</button></div><div class="setting-row"><span>🔒 Privacy Policy</span><button class="link-btn" onclick="openPrivacy()">VIEW</button></div><div class="setting-row"><span>ℹ About</span><button class="link-btn" onclick="openAbout()">VIEW</button></div><button class="secondary-action full" onclick="closeModal()">DONE</button>`);
  $("#soundToggle").onclick=()=>{state.sound=!state.sound;save();settings()};
  $("#musicToggle").onclick=()=>{state.music=!state.music;save();if(state.music)startMusic();else stopMusic();settings()};
}
function openPrivacy(){modal(`<div class="reward-icon">🔒</div><h2>Privacy Policy</h2><p>Your privacy policy page will be connected here later.</p><button class="modal-action" onclick="closeModal()">OK</button>`)}
function accountModal(){
  if(!firebaseReady){modal(`<div class="auth-shell"><div class="auth-brand">🐣</div><div class="eyebrow">TILE TRAILS ACCOUNT</div><h2>CONNECT & PLAY</h2><p>Firebase could not be loaded. Check your internet connection and try again.</p><button class="secondary-action full" onclick="closeModal()">CLOSE</button></div>`);return;}
  if(currentUser){
    const name=currentUser.displayName||"Player";
    const phone=currentUser.phoneNumber||"Phone account";
    modal(`<div class="auth-shell profile-shell"><div class="profile-glow"><div class="account-avatar">${name.slice(0,1).toUpperCase()}</div></div><div class="eyebrow">YOUR ACCOUNT</div><h2>${escapeHtml(name)}</h2><p class="profile-email">${escapeHtml(phone)}</p><span class="cloud-badge">☁ CLOUD SAVE ACTIVE</span><div class="profile-stat"><span>LOGIN METHOD</span><b>PHONE NUMBER</b></div><div class="profile-stat"><span>PROGRESS</span><b>LEVEL ${state.level}</b></div><button class="secondary-action full" onclick="logoutAccount()">LOG OUT</button><button class="secondary-action full" onclick="closeModal()">CLOSE</button></div>`);return;
  }
  modal(`<div class="auth-shell"><div class="auth-top"><div class="auth-brand">📱</div><div><div class="eyebrow">WELCOME BACK</div><h2>SAVE YOUR JOURNEY</h2></div></div><p class="auth-subtitle">Sign in with your phone number to keep your Tile Trails progress safe across devices.</p><div id="phoneAuthArea"></div><div class="guest-note"><span>🎮</span><div><b>Playing as Guest</b><small>Your progress stays only on this device until you connect a phone account.</small></div></div><button class="secondary-action full" onclick="closeModal()">MAYBE LATER</button></div>`);
  showPhoneAuth();
}
function escapeHtml(v){return String(v||"").replace(/[&<>'"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;","'":"&#39;","\"":"&quot;"}[c]));}
function clearRecaptcha(){
  try{if(recaptchaVerifier)recaptchaVerifier.clear();}catch(e){console.warn("reCAPTCHA clear failed",e)}
  recaptchaVerifier=null;
  const host=$("#recaptcha-container"); if(host)host.innerHTML="";
}
function showPhoneAuth(){
  confirmationResult=null;authPhonePending=false;authPhoneName="";clearRecaptcha();
  const a=$("#phoneAuthArea");if(!a)return;
  a.innerHTML=`<div class="auth-form"><label>YOUR NAME</label><input class="auth-input" id="authName" placeholder="Your name" maxlength="30" autocomplete="name"><label>PHONE NUMBER</label><div class="phone-row"><span class="country-code">🇮🇳 +91</span><input class="auth-input phone-input" id="authPhone" type="tel" inputmode="numeric" maxlength="10" placeholder="10 digit mobile number" autocomplete="tel-national"></div><div id="recaptcha-container" class="recaptcha-box"></div><p class="auth-help">We'll send a 6-digit verification code by SMS. Standard SMS charges may apply.</p><button class="modal-action" onclick="sendPhoneOTP()">SEND OTP <span>→</span></button></div>`;
  try{
    recaptchaVerifier=new firebase.auth.RecaptchaVerifier("recaptcha-container",{size:"normal",callback:()=>{}});
    recaptchaVerifier.render().catch(e=>console.error("reCAPTCHA render failed",e));
  }catch(e){console.error("reCAPTCHA init failed",e);showToast("SECURITY CHECK COULD NOT LOAD","bad")}
}
function normalizePhone(){
  const raw=$("#authPhone")?.value.trim().replace(/\s|-/g,"")||"";
  if(/^\d{10}$/.test(raw))return "+91"+raw;
  if(/^\+91\d{10}$/.test(raw))return raw;
  return "";
}
async function sendPhoneOTP(){
  if(authBusy)return;
  if(!firebaseReady||!auth){showToast("FIREBASE NOT READY — CHECK INTERNET","bad");return;}
  const name=$("#authName")?.value.trim()||"Player";
  const phone=normalizePhone();
  if(!phone){showToast("ENTER A VALID 10 DIGIT MOBILE NUMBER","bad");return;}
  if(!recaptchaVerifier){showToast("SECURITY CHECK NOT READY — TRY AGAIN","bad");return;}
  authBusy=true;authPhoneName=name;
  try{
    confirmationResult=await auth.signInWithPhoneNumber(phone,recaptchaVerifier);
    authPhonePending=true;
    const a=$("#phoneAuthArea");
    if(a)a.innerHTML=`<div class="auth-form"><div class="otp-sent">✉️ OTP SENT</div><p class="auth-help">Enter the 6-digit code sent to <b>${escapeHtml(phone)}</b>.</p><label>VERIFICATION CODE</label><input class="auth-input otp-input" id="authOtp" type="tel" inputmode="numeric" maxlength="6" placeholder="••••••" autocomplete="one-time-code"><button class="modal-action" onclick="verifyPhoneOTP()">VERIFY & CONTINUE <span>→</span></button><button class="link-btn full-link" onclick="showPhoneAuth()">CHANGE NUMBER / RESEND</button></div>`;
    showToast("OTP SENT","good");
  }catch(e){
    console.error("Phone OTP failed",e);showToast(firebaseAuthError(e),"bad");
    try{if(recaptchaVerifier)recaptchaVerifier.clear()}catch(_){ }
    recaptchaVerifier=null;
  }finally{authBusy=false}
}
async function verifyPhoneOTP(){
  if(authBusy||!confirmationResult)return;
  const otp=$("#authOtp")?.value.trim()||"";
  if(!/^\d{6}$/.test(otp)){showToast("ENTER THE 6 DIGIT OTP","bad");return;}
  authBusy=true;
  try{
    const r=await confirmationResult.confirm(otp);
    if(authPhoneName)await r.user.updateProfile({displayName:authPhoneName});
    cloudHydrated=true;
    await db.ref("users/"+r.user.uid).set(cloudPayload());
    closeModal();showToast("LOGIN SUCCESS • CLOUD SAVE ON","good");
  }catch(e){console.error("Phone OTP verify failed",e);showToast(firebaseAuthError(e),"bad");}
  finally{authBusy=false;confirmationResult=null;authPhonePending=false;clearRecaptcha()}
}
async function logoutAccount(){try{await auth.signOut();cloudHydrated=false;closeModal();showToast("LOGGED OUT • GUEST MODE ACTIVE","good")}catch(e){showToast("LOGOUT FAILED","bad")}}
function firebaseAuthError(e){const m={
  "auth/invalid-phone-number":"INVALID PHONE NUMBER",
  "auth/missing-phone-number":"ENTER YOUR PHONE NUMBER",
  "auth/quota-exceeded":"SMS LIMIT REACHED — TRY AGAIN LATER",
  "auth/too-many-requests":"TOO MANY ATTEMPTS — TRY AGAIN LATER",
  "auth/captcha-check-failed":"SECURITY CHECK FAILED — TRY AGAIN",
  "auth/invalid-verification-code":"INCORRECT OTP",
  "auth/code-expired":"OTP EXPIRED — REQUEST A NEW ONE",
  "auth/session-expired":"OTP SESSION EXPIRED — REQUEST A NEW ONE",
  "auth/operation-not-allowed":"PHONE SIGN-IN IS DISABLED IN FIREBASE",
  "auth/unauthorized-domain":"ADD THIS SITE DOMAIN IN FIREBASE AUTHORIZED DOMAINS",
  "auth/network-request-failed":"NETWORK ERROR — CHECK INTERNET",
  "auth/user-disabled":"THIS ACCOUNT IS DISABLED",
  "auth/app-not-authorized":"THIS APP IS NOT AUTHORIZED FOR PHONE AUTH",
  "auth/invalid-app-credential":"PHONE AUTH SECURITY CHECK FAILED",
  "auth/missing-verification-code":"ENTER THE OTP",
  "auth/missing-verification-id":"OTP SESSION MISSING — REQUEST A NEW ONE"
};return m[e?.code]||"LOGIN ERROR — TRY AGAIN"}

function init(){
  document.addEventListener("contextmenu",e=>e.preventDefault());
  document.addEventListener("selectstart",e=>e.preventDefault());
  document.addEventListener("dragstart",e=>e.preventDefault());
  $("#eventBtn").onclick=openEvents;
  $("#accountBtn").onclick=accountModal;
  $("#playBtn").onclick=()=>startLevel(state.level);
  $("#homeMap").onclick=()=>{buildMap();show("map");requestAnimationFrame(()=>$("#mapPath").scrollTop=0)};
  $("#mapBtn").onclick=()=>{buildMap();show("map");requestAnimationFrame(()=>$("#mapPath").scrollTop=0)};
  $("#mapHome").onclick=()=>{clearInterval(timerId);show("home");buildPreview();updateCoins()};
  $("#mapBackTop").onclick=()=>{clearInterval(timerId);show("home");buildPreview();updateCoins()};
  $("#gameHome").onclick=()=>{clearInterval(timerId);gameOver=true;pickBusy=false;updateCoins();buildPreview();show("home")};
  $("#homeShop").onclick=openShop;
  $("#shopTop").onclick=openShop;
  $("#dailyBtn").onclick=claimDaily;
  $("#settingsBtn").onclick=settings;
  $("#closeModal").onclick=closeModal;
  [["#undo",useUndo],["#shuffle",useShuffle],["#hint",useHint]].forEach(([sel,fn])=>{
    const b=$(sel); if(!b)return;
    b.onclick=null;
    b.addEventListener("pointerup",e=>{e.preventDefault();e.stopPropagation();fn();},{passive:false});
    b.addEventListener("click",e=>{e.preventDefault();e.stopPropagation();},{passive:false});
  });
  // Delegated pointer/click handling keeps taps reliable after every board re-render.
  const handleTileTap=e=>{
    const tile=e.target.closest?.(".tile");
    if(!tile||e.button===2)return;
    e.preventDefault();
    pick(Number(tile.dataset.id));
  };
  $("#board").addEventListener("pointerup",handleTileTap,{passive:false});
  $("#board").addEventListener("click",handleTileTap);
  updateCoins(); buildPreview(); buildMap(); updatePowerups();
  setTimeout(()=>{updateCoins();buildPreview();buildMap();updatePowerups()},300);
}
window.addEventListener("error",e=>console.error("Tile Trails error:",e.error||e.message));
if(document.readyState==="loading") document.addEventListener("DOMContentLoaded",init,{once:true}); else init();
