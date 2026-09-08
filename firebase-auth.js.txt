(function(){
  'use strict';
  const $ = id => document.getElementById(id);
  let confirmationResult = null;
  let verifier = null;
  let currentUser = null;
  let authReady = false;
  let applyingCloud = false;

  function setStatus(text, bad){
    const el = $('authStatus');
    if(!el) return;
    el.textContent = text || '';
    el.className = 'auth-status' + (bad ? ' bad' : '');
  }
  function esc(v){
    return String(v||'').replace(/[&<>'"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;',"'":'&#39;','"':'&quot;'}[c]));
  }
  function showAuth(){ $('authModal')?.classList.remove('hidden'); setStatus(''); updateAuthView(); }
  function hideAuth(){ $('authModal')?.classList.add('hidden'); setStatus(''); }
  function updateAuthView(){
    const signed = !!currentUser;
    $('authSignedOut')?.classList.toggle('hidden', signed);
    $('authSignedIn')?.classList.toggle('hidden', !signed);
    const label = $('accountLabel');
    if(label) label.textContent = signed ? (currentUser.email || currentUser.phoneNumber || 'ACCOUNT') : 'LOGIN';
    const userName = $('accountName');
    if(userName) userName.textContent = currentUser ? (currentUser.displayName || currentUser.email || currentUser.phoneNumber || 'Player') : '';
    const verified = $('verifyNote');
    if(verified && currentUser) verified.textContent = currentUser.email ? (currentUser.emailVerified ? '✓ Email verified' : '⚠ Email not verified') : 'Phone account';
  }
  function friendly(err){
    const code = err && err.code || '';
    const map = {
      'auth/email-already-in-use':'This email is already registered. Please log in.',
      'auth/invalid-email':'Please enter a valid email address.',
      'auth/weak-password':'Password must be at least 6 characters.',
      'auth/invalid-credential':'Email or password is incorrect.',
      'auth/user-not-found':'Account not found. Please sign up first.',
      'auth/wrong-password':'Email or password is incorrect.',
      'auth/too-many-requests':'Too many attempts. Please wait and try again.',
      'auth/invalid-phone-number':'Enter phone number with country code, e.g. +91XXXXXXXXXX.',
      'auth/quota-exceeded':'SMS quota exceeded. Try again later.',
      'auth/code-expired':'OTP expired. Please request a new OTP.',
      'auth/invalid-verification-code':'Incorrect OTP. Please try again.',
      'auth/captcha-check-failed':'reCAPTCHA failed. Please retry.',
      'auth/operation-not-allowed':'Enable this sign-in method in Firebase Authentication.',
      'auth/network-request-failed':'Network error. Check your internet connection.'
    };
    return map[code] || (err && err.message) || 'Something went wrong. Please try again.';
  }
  async function emailSignup(){
    const email = $('signupEmail').value.trim();
    const pass = $('signupPassword').value;
    if(!email || !pass){setStatus('Enter email and password.', true);return;}
    try{
      setStatus('Creating account...');
      const cred = await firebase.auth().createUserWithEmailAndPassword(email, pass);
      await cred.user.sendEmailVerification();
      setStatus('Account created. Verification email sent.');
    }catch(e){setStatus(friendly(e), true);}
  }
  async function emailLogin(){
    const email = $('loginEmail').value.trim();
    const pass = $('loginPassword').value;
    if(!email || !pass){setStatus('Enter email and password.', true);return;}
    try{setStatus('Logging in...'); await firebase.auth().signInWithEmailAndPassword(email, pass); hideAuth();}
    catch(e){setStatus(friendly(e), true);}
  }
  async function resendVerification(){
    if(!currentUser || !currentUser.email){return;}
    try{await currentUser.sendEmailVerification();setStatus('Verification email sent again.');}
    catch(e){setStatus(friendly(e), true);}
  }
  async function phoneSend(){
    const phone = $('phoneNumber').value.trim();
    if(!phone){setStatus('Enter your phone number with country code.', true);return;}
    try{
      if(!verifier){
        verifier = new firebase.auth.RecaptchaVerifier('recaptcha-container', {size:'normal'});
        await verifier.render();
      }
      setStatus('Sending OTP...');
      confirmationResult = await firebase.auth().signInWithPhoneNumber(phone, verifier);
      $('otpRow')?.classList.remove('hidden');
      setStatus('OTP sent. Enter the 6-digit code.');
    }catch(e){
      if(verifier){try{verifier.clear();}catch(_){} verifier=null;}
      setStatus(friendly(e), true);
    }
  }
  async function phoneVerify(){
    const code = $('otpCode').value.trim();
    if(!confirmationResult || !code){setStatus('Request an OTP and enter the code.', true);return;}
    try{setStatus('Verifying OTP...');await confirmationResult.confirm(code);hideAuth();}
    catch(e){setStatus(friendly(e), true);}
  }
  async function logout(){
    try{await firebase.auth().signOut();hideAuth();}catch(e){setStatus(friendly(e), true);}
  }
  async function saveCloud(){
    if(!authReady || !currentUser || applyingCloud || !window.TileTrailsGame) return;
    try{
      const s=window.TileTrailsGame.state;
      await firebase.firestore().collection('users').doc(currentUser.uid).set({
        level:Number(s.level)||1,
        maxUnlocked:Number(s.maxUnlocked)||1,
        coins:Number(s.coins)||0,
        streak:Number(s.streak)||0,
        lastDaily:String(s.lastDaily||''),
        powerups:s.pu||{undo:3,shuffle:3,hint:3},
        sound:!!s.sound,
        music:!!s.music,
        updatedAt:firebase.firestore.FieldValue.serverTimestamp()
      }, {merge:true});
    }catch(e){console.warn('Cloud save failed:',e);}
  }
  async function loadCloud(){
    if(!currentUser || !window.TileTrailsGame) return;
    try{
      const snap=await firebase.firestore().collection('users').doc(currentUser.uid).get();
      if(!snap.exists) { await saveCloud(); return; }
      const d=snap.data()||{}, s=window.TileTrailsGame.state;
      applyingCloud=true;
      if(Number.isFinite(Number(d.level))) s.level=Math.min(3000,Math.max(1,Number(d.level)));
      if(Number.isFinite(Number(d.maxUnlocked))) s.maxUnlocked=Math.min(3000,Math.max(s.level,Number(d.maxUnlocked)));
      else s.maxUnlocked=Math.max(s.level, Number(localStorage.getItem('tt_max_unlocked'))||1);
      if(Number.isFinite(Number(d.coins))) s.coins=Math.max(0,Number(d.coins));
      if(Number.isFinite(Number(d.streak))) s.streak=Math.max(0,Number(d.streak));
      if(typeof d.lastDaily==='string') s.lastDaily=d.lastDaily;
      if(d.powerups && typeof d.powerups==='object') s.pu={undo:Math.max(0,Number(d.powerups.undo)||0),shuffle:Math.max(0,Number(d.powerups.shuffle)||0),hint:Math.max(0,Number(d.powerups.hint)||0)};
      if(typeof d.sound==='boolean') s.sound=d.sound;
      if(typeof d.music==='boolean') s.music=d.music;
      window.TileTrailsGame.saveLocal();
      window.TileTrailsGame.refreshUI();
    }catch(e){console.warn('Cloud load failed:',e);}
    finally{applyingCloud=false;}
  }
  function init(){
    if(!window.firebase){setStatus('Firebase SDK failed to load.', true);return;}
    try{firebase.initializeApp(window.firebaseConfig);}catch(e){console.error(e);setStatus('Firebase configuration error.', true);return;}
    const auth=firebase.auth();
    auth.onAuthStateChanged(async user=>{
      currentUser=user; authReady=true; updateAuthView();
      if(user){ await loadCloud(); }
    });
    $('accountBtn')?.addEventListener('click',showAuth);
    $('authClose')?.addEventListener('click',hideAuth);
    $('signupBtn')?.addEventListener('click',emailSignup);
    $('loginBtn')?.addEventListener('click',emailLogin);
    $('resendBtn')?.addEventListener('click',resendVerification);
    $('phoneSendBtn')?.addEventListener('click',phoneSend);
    $('phoneVerifyBtn')?.addEventListener('click',phoneVerify);
    $('logoutBtn')?.addEventListener('click',logout);
    $('authModal')?.addEventListener('click',e=>{if(e.target.id==='authModal')hideAuth();});
    updateAuthView();
  }
  window.TileTrailsAuth={getUser:()=>currentUser,saveCloud,showAuth};
  window.addEventListener('tiletrails-save',()=>{if(authReady&&!applyingCloud) saveCloud();});
  if(document.readyState==='loading') document.addEventListener('DOMContentLoaded',init,{once:true}); else init();
})();
