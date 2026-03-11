// --- STATE ---
let userLocation = null;
let userData = { 
    element: 'water', 
    name: '', 
    instruments: [], 
    availPills: [], 
    schedule: {}, 
    avatar: null,
    masteryUnlocked: false,
    bio: "",        // NEW: Origin Story
    socials: {}     // NEW: Signal Uplinks
};
let allMarkers = [];

// 1. RICH JAM DATA (The Database)
const JAM_DATA = [];

const DAYS = ['MON','TUE','WED','THU','FRI','SAT','SUN'];
const TIMES = ['Morn','Day','Night'];
let isCreating = false;
let tempCoords = null;

const elementDefs = {
    water: "<span class='highlight-word'>WATER</span><br>Adaptable. The flow.",
    fire: "<span class='highlight-word'>FIRE</span><br>High energy. The spark.",
    earth: "<span class='highlight-word'>EARTH</span><br>Grounded. The foundation.",
    air: "<span class='highlight-word'>AIR</span><br>Atmospheric. The space."
};

// --- INIT ---
window.onload = function() {
  // Check Identity
  if(window.jamblrAuth) {
      window.jamblrAuth.init().then(hasId => {
          if(hasId) console.log("Identity Loaded");
      });
  }

  const saved = localStorage.getItem("jamblr_v3");
  if(saved) {
    userData = JSON.parse(saved);
    
    if (userData.masteryUnlocked) {
         const btn = document.querySelector('.btn-aether');
         if(btn) btn.classList.remove('locked');
    }
    
    morph(userData.element);
    enterApp();
  } else {
    document.getElementById('the-door').classList.remove('vanish');
  }
};

// --- ONBOARDING ---
function capitalize(s) { return s.charAt(0).toUpperCase() + s.slice(1); }

window.selectElement = function(el, btnWrapper) {
  document.querySelectorAll('.element-item').forEach(item => {
    item.classList.remove('active');
    item.querySelector('.el-option').style.cssText = "background-image: url('Jamblr Images/" + capitalize(el) + "-Pin.png')";
    item.querySelector('.el-label').style.cssText = "";
  });
  btnWrapper.classList.add('active');
  const root = document.documentElement;
  const glow = getComputedStyle(root).getPropertyValue('--glow-' + el).trim();
  const color = getComputedStyle(root).getPropertyValue('--color-' + el).trim();
  
  const opt = btnWrapper.querySelector('.el-option');
  opt.style.cssText = "background-image: url('Jamblr Images/" + capitalize(el) + "-Pin.png'); filter: grayscale(0%) opacity(1) drop-shadow(0 0 15px " + glow + "); transform: scale(1.2) translateY(-5px);";
  const lbl = btnWrapper.querySelector('.el-label');
  lbl.style.cssText = `opacity: 1; font-weight: bold; color: ${color};`;
  
  document.getElementById('element-desc-box').innerHTML = elementDefs[el];
  userData.element = el;
  morph(el);
};

window.togglePill = function(btn) { btn.classList.toggle('selected'); };

window.handleAvatar = function(input) {
  if(input.files && input.files[0]) {
    const reader = new FileReader();
    reader.onload = function(e) {
      document.getElementById('avatar-preview').src = e.target.result;
      document.getElementById('avatar-preview').style.display = 'block';
      userData.avatar = e.target.result;
    };
    reader.readAsDataURL(input.files[0]);
  }
};

window.nextStep = function(n) {
  if(n === 3) userData.name = document.getElementById('input-name-onboarding').value || "Traveler";
  document.querySelectorAll('.step-container').forEach(el => el.classList.remove('active'));
  document.getElementById('step-' + n).classList.add('active');
};

window.finishOnboarding = async function() {
    userData.availPills = [];
    document.querySelectorAll('#avail-grid .selected')
        .forEach(el => userData.availPills.push(el.innerText));
    userData.instruments = [];
    document.querySelectorAll('#instrument-grid .selected')
        .forEach(el => userData.instruments.push(el.innerText));
    
    userData.schedule = {};
    DAYS.forEach(d => {
        TIMES.forEach(t => {
            userData.schedule[`${d}-${t}`] = false;
        });
    });
    
    const pills = userData.availPills;
    if (pills.includes('Anytime')) {
        for (let k in userData.schedule) userData.schedule[k] = true;
    } else {
        if (pills.includes('Weeknights'))
            ['MON', 'TUE', 'WED', 'THU', 'FRI']
            .forEach(d => userData.schedule[`${d}-Night`] = true);
        if (pills.includes('Weekends'))
            ['SAT', 'SUN'].forEach(d => {
                userData.schedule[`${d}-Day`] = true;
                userData.schedule[`${d}-Night`] = true;
            });
        if (pills.includes('Mornings'))
            DAYS.forEach(d => userData.schedule[`${d}-Morn`] = true);
    }
    
    // ── SOVEREIGN IDENTITY GENERATION ──
    // Show a brief message while keys are forged
    const enterBtn = document.querySelector('#step-4 .btn-action');
    if (enterBtn) {
        enterBtn.innerText = "FORGING IDENTITY...";
        enterBtn.disabled = true;
    }
    
    try {
        const soul = await generateSovereignSoul();
        
        // Store keys
        localStorage.setItem('jamblr_public_key', soul.fullPublicKey);
        localStorage.setItem('jamblr_secret_key', soul.privateSecret);
        localStorage.setItem('jamblr_soul_id', soul.soulID);
        
        // Attach public identity to userData
        userData.soulID = soul.soulID;
        userData.publicKey = soul.fullPublicKey;
        
        saveData();
        
        showToast("⚡ Identity Forged. Welcome to the Galaxy.");
        
        setTimeout(() => {
            enterApp();
        }, 1500);
        
    } catch (err) {
        console.error("Key generation failed:", err);
        // Fallback — enter app anyway with a random ID
        userData.soulID = 'jb_' + Math.random().toString(36).substring(2, 10);
        saveData();
        enterApp();
    }
};

window.saveData = function() { localStorage.setItem("jamblr_v3", JSON.stringify(userData)); };

window.enterApp = function() {
  document.getElementById('the-door').classList.add('vanish');
  document.getElementById('main-app').classList.add('visible');
  initMap();
  renderCalendar();
  loadProfileData(); // UPDATED: Calls the new Deep Profile engine
};

// --- NAVIGATION ---
window.switchView = function(viewName, navItem) {
  document.querySelectorAll('.nav-item').forEach(el => el.classList.remove('active'));
  navItem.classList.add('active');
  
  document.querySelectorAll('.view-section').forEach(el => el.classList.remove('active'));
  document.getElementById('view-' + viewName).classList.add('active');
  
  if(viewName === 'map') setTimeout(() => { window.mapObj.resize(); }, 100);
  
  if(viewName !== 'map') {
      isCreating = false;
      document.getElementById('center-btn').classList.remove('creating');
  }
};

// --- MAP ENGINE ---
window.initMap = function() {
  if(window.mapObj) return;
  window.mapObj = new maplibregl.Map({
    container: 'map', 
    style: 'https://basemaps.cartocdn.com/gl/dark-matter-gl-style/style.json',
    center: [-78.636, 42.833], zoom: 12, attributionControl: false
  });

  const geolocate = new maplibregl.GeolocateControl({
      positionOptions: { enableHighAccuracy: true },
      trackUserLocation: true,
      showUserHeading: true
  });
  window.mapObj.addControl(geolocate, 'top-right');

  let hasLocked = false; 

  geolocate.on('geolocate', (e) => {
      userLocation = { lat: e.coords.latitude, lng: e.coords.longitude };
      
      if (!hasLocked) {
           console.log("User located at:", userLocation);
           showToast("Location Locked. Updating Distances...");
           hasLocked = true; 
           
           if(allMarkers) {
               allMarkers.forEach(m => m.remove());
               allMarkers = [];
           }
           JAM_DATA.forEach(jam => addPinToMap(jam));
      }
  });


  geolocate.on('error', (e) => {
      console.error("Geolocation Error:", e);
      if (e.code === 1) showToast("⚠️ Location Denied. Check Device Settings.");
      else if (e.code === 3) showToast("⚠️ Location Timeout. Try again.");
      else showToast("⚠️ GPS Error: " + e.message);
  });

  window.mapObj.on('load', () => {
const savedJams = localStorage.getItem('jamblr_jams');
if(savedJams) JSON.parse(savedJams).forEach(jam => { JAM_DATA.push(jam); addPinToMap(jam); });
     
     window.mapObj.on('click', (e) => {
        if(isCreating) {
            tempCoords = e.lngLat;
            document.getElementById('modal-overlay').classList.add('active');
        }
     });
  });
};

// Helper: Calculate distance
function getDistance(lat1, lon1, lat2, lon2) {
    if (!lat1 || !lon1) return ""; 
    const R = 3958.8; 
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;
    const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
              Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * Math.sin(dLon/2) * Math.sin(dLon/2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return (R * c).toFixed(1) + " mi"; 
}

function addPinToMap(jam) {
     const el = document.createElement('div');
     el.className = 'jamblr-pin';
     const capType = capitalize(jam.type);
     el.style.backgroundImage = "url('Jamblr Images/" + capType + "-Pin.png')";
     
     let distText = "";
     if (typeof userLocation !== 'undefined' && userLocation) {
         distText = getDistance(userLocation.lat, userLocation.lng, jam.lat, jam.lng);
     }

     // Fix nav link format
     const navLink = `https://www.google.com/maps/dir/?api=1&destination=${jam.lat},${jam.lng}`;

     let tagsHTML = `<div class="jam-tag tag-genre">${jam.genre || 'Open Jam'}</div>`;
     if(jam.needs) {
         jam.needs.forEach(inst => {
             tagsHTML += `<div class="jam-tag tag-inst">+ ${inst}</div>`;
         });
     }
     
     const hostImg = jam.avatar || "Jamblr Images/Jamblr-Wordmark.png";

     const popupHTML = `
        <div class="jam-popup-inner type-${jam.type}">
            <div class="jam-header">
                <div class="host-avatar" style="background-image: url('${hostImg}')"></div>
                <div class="host-name">${jam.host || 'Anonymous'}</div>
                <div style="margin-left:auto; font-size:0.7rem; opacity:0.7;">${distText}</div>
            </div>

            <div class="jam-card-title">${jam.title}</div>
            
            <div class="tag-container">
                ${tagsHTML}
            </div>

            <div style="font-size:0.85rem; margin-bottom:10px; line-height:1.4;">${jam.desc}</div>
            
            <div class="jam-stat-row">
                <span>⏰ ${jam.time}</span>
                <span>📍 ${distText || "Unknown"}</span>
            </div>

            <div style="display:flex; gap:10px; margin-top:10px;">
                <button class="btn-join" onclick="showToast('Request Sent to ${jam.host}...')">JOIN SIGNAL</button>
                <a href="${navLink}" target="_blank" class="btn-join" style="width:50px; text-align:center; text-decoration:none; display:flex; align-items:center; justify-content:center; background:rgba(255,255,255,0.2);">
                    🚀
                </a>
            </div>
        </div>
     `;
     
     const popupClass = jam.type === 'aether' ? 'aether-popup' : '';
     const popup = new maplibregl.Popup({ offset: 35, closeButton: true, className: popupClass }).setHTML(popupHTML);

     const marker = new maplibregl.Marker({ element: el, anchor: 'bottom' })
        .setLngLat([jam.lng, jam.lat])
        .setPopup(popup)
        .addTo(window.mapObj);
        
     marker.jamType = jam.type;
     allMarkers.push(marker);
}

// --- CENTRAL CREATE BUTTON LOGIC ---
window.activateCreationMode = function(btn) {
    document.querySelectorAll('.view-section').forEach(el => el.classList.remove('active'));
    document.getElementById('view-map').classList.add('active');
    if(window.mapObj) window.mapObj.resize();

    isCreating = !isCreating;
    if(isCreating) {
        btn.classList.add('creating');
        showToast("Tap Map to Place Signal");
    } else {
        btn.classList.remove('creating');
    }
};
window.updateModalColor = function() {
    const type = document.getElementById('jam-type').value;
    const root = document.documentElement;
    const color = getComputedStyle(root).getPropertyValue('--color-' + type).trim();
    const glow = getComputedStyle(root).getPropertyValue('--glow-' + type).trim();
    root.style.setProperty('--active-color', color);
    root.style.setProperty('--active-glow', glow);
};
window.closeModal = function() {
    document.getElementById('modal-overlay').classList.remove('active');
    isCreating = false;
    document.getElementById('center-btn').classList.remove('creating');
    document.getElementById('jam-title').value = '';
    document.getElementById('jam-genre').value = '';
    document.getElementById('jam-time').value = '';
    document.getElementById('jam-desc').value = '';
    document.getElementById('jam-type').value = 'water';
    document.querySelectorAll('#jam-needs-grid .inst-pill.selected').forEach(p => p.classList.remove('selected'));
    updateModalColor();
};

window.submitJam = function() {
    const title = document.getElementById('jam-title').value || "New Signal";
    const type = document.getElementById('jam-type').value;
    const desc = document.getElementById('jam-desc').value || "Vibe check...";
    const time = document.getElementById('jam-time').value || "Now";
    const genre = document.getElementById('jam-genre').value;

    let needs = [];
    document.querySelectorAll('#jam-needs-grid .inst-pill.selected').forEach(el => {
        needs.push(el.innerText);
});
    const newJam = {
        lat: tempCoords.lat,
        lng: tempCoords.lng,
        title: title,
        type: type,
        desc: desc,
        time: time,
        genre: genre,
        needs: needs,
        host: userData.name || "Me",
        avatar: userData.avatar
    };

JAM_DATA.push(newJam);
localStorage.setItem('jamblr_jams', JSON.stringify(JAM_DATA));
    addPinToMap(newJam);
    closeModal();
    showToast("Broadcasting Signal...");
};

window.showToast = function(msg) {
    const t = document.getElementById('toast');
    t.innerText = msg;
    t.classList.add('show');
    setTimeout(() => t.classList.remove('show'), 3000);
};

// --- CALENDAR RENDER ---
window.renderCalendar = function() {
  const wrapper = document.getElementById('calendar-wrapper');
  wrapper.innerHTML = "";
  wrapper.innerHTML += `<div></div>`; 
  TIMES.forEach(t => wrapper.innerHTML += `<div class="cal-head">${t}</div>`);
  DAYS.forEach(day => {
    wrapper.innerHTML += `<div class="day-label">${day}</div>`;
    TIMES.forEach(time => {
      const key = `${day}-${time}`;
      const isActive = userData.schedule[key];
      const div = document.createElement('div');
      div.className = `time-slot ${isActive ? 'active' : ''}`;
      div.onclick = function() {
         userData.schedule[key] = !userData.schedule[key];
         div.classList.toggle('active');
         saveData();
      };
      wrapper.appendChild(div);
    });
  });
};

// --- DEEP PROFILE ENGINE (Replaces Old Render/Edit Logic) ---

function loadProfileData() {
    // 1. Get Data (Safeguard in case user has old data version)
    const data = userData || { 
        name: "Ghost", 
        element: "EARTH", 
        instruments: [], 
        bio: "", 
        socials: {} 
    };
    
    // 2. Fill Basic Text
    const initial = data.name ? data.name.charAt(0).toUpperCase() : "?";
    const avUrl = data.avatar ? `url(${data.avatar})` : 'none';
    
    // Update Avatar Box
    const avEl = document.getElementById('profile-avatar');
    if(avEl) {
        avEl.style.backgroundImage = avUrl;
        avEl.innerText = data.avatar ? "" : initial;
    }

    document.getElementById('display-name').innerText = data.name;
    document.getElementById('display-element').innerText = data.element.toUpperCase();
    document.getElementById('display-bio').innerText = data.bio || "\"No origin story recorded.\"";
    document.getElementById('display-instruments').innerText = (data.instruments || []).join(', ') || "No Gear Listed";
    
    // 3. Render Social Buttons (STRICT INDIE ORDER)
    const socialGrid = document.getElementById('social-grid-view');
    socialGrid.innerHTML = ""; 
    
    const links = data.socials || {};
    let hasLinks = false;

    // DEFINED ORDER: Indies First, Mainstream Last
    const platformOrder = [
        'bandcamp', 'audius', 'odysee',       // Tier 1: The Indie Trinity
        'soundcloud', 'spotify',              // Tier 2: Audio
        'instagram', 'youtube', 'tiktok'      // Tier 3: Visuals
    ];

    platformOrder.forEach(platform => {
        const url = links[platform];
        if(url && url.length > 0) {
            hasLinks = true;
            const btn = document.createElement('a');
            btn.href = url.startsWith('http') ? url : `https://${url}`;
            btn.target = "_blank"; 
            btn.className = "social-link-pill";
            
            // Icons & Labels
            let icon = "🔗";
            if(platform === 'bandcamp')   icon = "🔵 Bandcamp";
            if(platform === 'audius')     icon = "🟣 Audius";
            if(platform === 'odysee')     icon = "🚀 Odysee";
            if(platform === 'soundcloud') icon = "🟠 SoundCloud";
            if(platform === 'spotify')    icon = "🟢 Spotify";
            if(platform === 'instagram')  icon = "📸 Insta";
            if(platform === 'youtube')    icon = "🔴 YouTube";
            if(platform === 'tiktok')     icon = "🎵 TikTok";
            
            btn.innerText = icon;
            socialGrid.appendChild(btn);
        }
    });

    if(!hasLinks) {
        socialGrid.innerHTML = "<span style='font-size:0.7rem; opacity:0.5; font-style:italic;'>No signals broadcast.</span>";
    }
    
    // 4. Update Sovereign ID Display
const storedKey = localStorage.getItem('jamblr_public_key');
if (storedKey) {
    const keyDisplay = document.getElementById('full-key-display');
    if (keyDisplay) keyDisplay.innerText = storedKey;
}

    updateThemeColor(data.element);
}

window.toggleEditProfile = function() {
    const container = document.getElementById('profile-card');
    const btn = document.getElementById('btn-edit-profile');
    const isEditing = container.classList.contains('editing');
    const data = userData || {};

    if (!isEditing) {
        // --- ENTER EDIT MODE ---
        container.classList.add('editing');
        btn.innerText = "SAVE UPLINKS";
        btn.style.background = "#00ff66";
        btn.style.color = "black";
        btn.style.borderColor = "#00ff66";
        
        // Show Inputs, Hide Displays
        document.getElementById('social-grid-view').classList.add('hidden');
        document.getElementById('social-grid-edit').classList.remove('hidden');

        // Pre-fill Inputs
        document.getElementById('input-name').value = data.name || "";
        document.getElementById('input-element').value = data.element || "earth";
        document.getElementById('input-bio').value = data.bio || "";
        document.getElementById('input-instruments').value = (data.instruments || []).join(', ');

        // Pre-fill Social Inputs
        document.querySelectorAll('.social-link-input').forEach(input => {
            const platform = input.getAttribute('data-platform');
            input.value = (data.socials && data.socials[platform]) ? data.socials[platform] : "";
        });

    } else {
        // --- SAVE CHANGES ---
        
        // 1. Scrape Basic Info
        window.userData.name = document.getElementById('input-name').value || "Unknown";
        window.userData.element = document.getElementById('input-element').value;
        window.userData.bio = document.getElementById('input-bio').value;
        
        const rawInst = document.getElementById('input-instruments').value;
        window.userData.instruments = rawInst.split(',').map(s => s.trim()).filter(s => s.length > 0);

        // 2. Scrape Socials
        window.userData.socials = {};
        document.querySelectorAll('.social-link-input').forEach(input => {
            if(input.value.trim().length > 0) {
                window.userData.socials[input.getAttribute('data-platform')] = input.value.trim();
            }
        });

        // 3. Commit to Storage
        saveData();

        // 4. Reset UI
        loadProfileData(); 
        container.classList.remove('editing');
        document.getElementById('social-grid-view').classList.remove('hidden');
        document.getElementById('social-grid-edit').classList.add('hidden');

        btn.innerText = "EDIT PROFILE";
        btn.style.background = ""; 
        btn.style.color = "";
        btn.style.borderColor = "var(--active-color)";
        
        showToast("Identity Updated");
    }
};

// Helper: Dynamic Theme Coloring (Used by morph and profile)
function updateThemeColor(element) {
    // We reuse the existing morph logic mostly, but this ensures profile updates apply immediately
    morph(element); 
}

// --- NEW LOGIC: FILTERING & MASTERY ---
window.morph = function(el) {
if (window.activeFilter === el) {
    window.activeFilter = null;
    document.querySelectorAll('.deck-btn').forEach(b => { b.classList.remove('active'); b.style.filter = "grayscale(100%) opacity(0.5)"; b.style.transform = "scale(1)"; });
    if(allMarkers) allMarkers.forEach(m => m.getElement().style.display = 'block');
    return;
}
window.activeFilter = el;
  
  const root = document.documentElement;
  const color = getComputedStyle(root).getPropertyValue('--color-' + el).trim();
  const glow = getComputedStyle(root).getPropertyValue('--glow-' + el).trim();
  root.style.setProperty('--active-color', color);
  root.style.setProperty('--active-glow', glow);
  
  document.querySelectorAll('.deck-btn').forEach(b => {
      b.classList.remove('active');
      b.style.filter = "grayscale(100%) opacity(0.5)";
      b.style.transform = "scale(1)";
  });
  const btn = document.querySelector('.btn-' + el);
  if(btn) {
      btn.classList.add('active');
      if (el === 'aether') {
         btn.style.transform = "scale(1.4) translateY(-10px)";
      } else {
         btn.style.transform = "scale(1.2) translateY(-5px)";
      }
      btn.style.filter = `grayscale(0%) opacity(1) drop-shadow(0 0 10px ${glow})`;
  }
  
if (allMarkers) {
    allMarkers.forEach(marker => {
        const pinElement = marker.getElement();
        if (marker.jamType === el) {
            pinElement.style.display = 'block';
        } else {
            pinElement.style.display = 'none';
        }
    });
}
};

window.tryAether = function(btn) {
    const isMaster = userData.masteryUnlocked || false; 

    if (!isMaster) {
        btn.style.animation = "shake 0.4s ease-in-out";
        setTimeout(() => btn.style.animation = "", 400);
        showToast("🔒 Locked: Master the 4 Elements to enter the Void.");
        return;
    }
    morph('aether');
};

// --- KEYRING TOOLS ---
window.copyToClipboard = function() {
    const key = document.getElementById('full-key-display').innerText;
    navigator.clipboard.writeText(key).then(() => {
        showToast("ID Copied to Clipboard");
    });
};

window.exportKeyring = function() {
    const secret = localStorage.getItem("jamblr_secret_key");
    if(!secret) return;
    
    const confirmBackup = confirm("WARNING: You are about to reveal your PRIVATE KEY.\n\nAnyone who sees this can steal your identity.\n\nDo you want to copy it to your clipboard for safekeeping?");
    
    if(confirmBackup) {
        navigator.clipboard.writeText(secret).then(() => {
            alert("SECRET KEY COPIED.\n\nPaste this into a secure note immediately.");
        });
    }
};

// --- KONAMI CODE ---
let secretClicks = 0;
window.handleKonami = function(navItem) {
    switchView('profile', navItem);
    
    secretClicks++;
    if (secretClicks >= 5) {
        unlockGodMode();
        secretClicks = 0; 
    }
    setTimeout(() => { secretClicks = 0; }, 1000);
};

window.unlockGodMode = function() {
    userData.masteryUnlocked = true;
    saveData();
    
    const btn = document.querySelector('.btn-aether');
    if(btn) {
        btn.classList.remove('locked');
        btn.style.filter = "brightness(2) drop-shadow(0 0 20px white)";
        setTimeout(() => { 
            btn.style.filter = ""; 
            showToast("⚡ MASTERY UNLOCKED: THE VOID IS OPEN");
        }, 500);
    }
};

// --- LEGAL AIRLOCK LOGIC ---
window.checkLegalState = function() {
    const c1 = document.getElementById('legal-check-1').checked;
    const c2 = document.getElementById('legal-check-2').checked;
    const btn = document.getElementById('btn-legal-enter');

    if(c1 && c2) {
        btn.classList.remove('disabled');
        btn.disabled = false;
        btn.style.opacity = "1";
        btn.style.filter = "none";
        btn.style.boxShadow = "0 0 15px var(--active-glow)";
    } else {
        btn.classList.add('disabled');
        btn.disabled = true;
        btn.style.opacity = "0.5";
        btn.style.filter = "grayscale(100%)";
        btn.style.boxShadow = "none";
    }
};

window.acceptLegal = function() {
    localStorage.setItem("jamblr_legal_accepted", "true");
    const airlock = document.getElementById('legal-airlock');
    airlock.classList.add('unlocked');
    setTimeout(() => {
        airlock.style.display = 'none';
        showToast("Protocol Accessed. Welcome.");
    }, 500);
};

if(localStorage.getItem("jamblr_legal_accepted") === "true") {
    const airlock = document.getElementById('legal-airlock');
    if(airlock) airlock.style.display = 'none';
}

// --- ACCOUNT MANAGEMENT & SAFETY ---
window.logout = function() {
    if(confirm("Log out of current session?")) {
        location.reload();
    }
};

window.toggleBurner = function() {
    const isArmed = document.getElementById('safety-switch').checked;
    const btn = document.getElementById('btn-burn');
    
    if(isArmed) {
        btn.disabled = false;
        btn.classList.remove('disabled');
        btn.style.background = "#ef4444";
        btn.style.color = "black";
        btn.style.borderColor = "#ef4444";
        btn.style.boxShadow = "0 0 20px rgba(239, 68, 68, 0.4)";
        btn.innerText = "⚠️ EXECUTE BURN";
    } else {
        btn.disabled = true;
        btn.classList.add('disabled');
        btn.style.background = "#1e293b";
        btn.style.color = "#475569";
        btn.style.borderColor = "#334155";
        btn.style.boxShadow = "none";
        btn.innerText = "⚠️ TERMINATE IDENTITY";
    }
};

window.burnIdentity = function() {
    if(confirm("⚠️ FINAL WARNING: DELETE ACCOUNT?\n\nThis will PERMANENTLY SHRED your private keys. You will lose this handle forever.")) { 
      const burnKeys = ["jamblr_v3", "jamblr_secret_key", "jamblr_legal_accepted"];
      burnKeys.forEach(key => {
          for(let i=0; i<3; i++) {
             localStorage.setItem(key, (Math.random() + 1).toString(36).substring(7));
          }
          localStorage.removeItem(key);
      });
      document.body.style.transition = "background 0.2s";
      document.body.style.background = "#ef4444"; 
      document.body.innerHTML = ""; 
      setTimeout(() => { location.reload(); }, 1000);
  }
};
async function generateSovereignSoul() {
    // Generates a NIST P-256 (ECC 256) Key Pair
    const keyPair = await window.crypto.subtle.generateKey(
        { name: "ECDSA", namedCurve: "P-256" },
        true, // extractable (so we can back it up to Bitwarden)
        ["sign", "verify"]
    );

    // Export the Public Key to hex to show the user (The "Soul" ID)
    const exportedPublic = await window.crypto.subtle.exportKey("raw", keyPair.publicKey);
    const body = Array.from(new Uint8Array(exportedPublic))
                      .map(b => b.toString(16).padStart(2, '0'))
                      .join('');
    
    // Export Private Key (The "Secret" for Bitwarden)
    const exportedPrivate = await window.crypto.subtle.exportKey("pkcs8", keyPair.privateKey);
    const secret = btoa(String.fromCharCode(...new Uint8Array(exportedPrivate)));

    return { 
        soulID: 'jb_' + body.substring(0, 16), // Short ID for the UI
        fullPublicKey: body,
        privateSecret: secret
    };
}
