# TimeApp
Aplikacja do śledzenia czasu
/diary-timer/
├── index.html
├── style.css
├── script.js
├── manifest.json
└── sw.js
<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Pamiętnik & Stoper</title>
  <link rel="stylesheet" href="style.css" />
  <link rel="manifest" href="manifest.json" />
</head>
<body>
  <nav>
    <button id="tab-diary">Pamiętnik</button>
    <button id="tab-timer">Stoper</button>
    <button id="toggle-theme">🌙 / ☀️</button>
  </nav>

  <div id="diary-module" class="module">
    <h2>Pamiętnik</h2>
    <textarea id="diary-text" placeholder="Twój wpis na dziś..."></textarea><br>
    <button id="save-diary">Zapisz wpis</button>
  </div>

  <div id="timer-module" class="module hidden">
    <h2>Stoper</h2>
    <input type="text" id="task-desc" placeholder="Opis zadania..." />
    <div id="timer-display">00:00:00</div>
    <button id="start-timer">Start</button>
    <button id="stop-timer" disabled>Stop</button>
    <h3>Historia sesji</h3>
    <div id="session-history"></div>
    <h3>Statystyka czasu według opisu</h3>
    <div id="stats"></div>
  </div>

  <script src="script.js"></script>
</body>
</html>
:root {
  --bg: #121212;
  --fg: #eee;
  --btn-bg: #333;
}
[data-theme="light"] {
  --bg: #f4f4f4;
  --fg: #222;
  --btn-bg: #ddd;
}
body {
  background: var(--bg);
  color: var(--fg);
  font-family: sans-serif;
  margin: 0; padding: 20px;
}
nav button {
  margin-right: 5px;
  padding: 10px;
  background: var(--btn-bg);
  color: var(--fg);
  border: none; border-radius: 5px;
}
.module { margin-top: 20px; }
.hidden { display: none; }
textarea, input[type="text"] { width:100%; background: #222; color: var(--fg); border: none; padding:10px; }
[data-theme="light"] textarea, [data-theme="light"] input { background:#fff; color:#000; }
#timer-display { font-size: 2em; margin: 10px 0; }
.session-item { border-top:1px solid #444; padding:5px 0; display:flex; justify-content:space-between; }
.stat-item { padding:4px 0; }
button.delete { background:red; color:#fff; border: none; padding:2px 6px; border-radius:3px; }
//  zakładki
const tabDiary = document.getElementById('tab-diary');
const tabTimer = document.getElementById('tab-timer');
const modDiary = document.getElementById('diary-module');
const modTimer = document.getElementById('timer-module');
tabDiary.onclick = () => { modDiary.classList.remove('hidden'); modTimer.classList.add('hidden'); };
tabTimer.onclick = () => { modTimer.classList.remove('hidden'); modDiary.classList.add('hidden'); };

// motyw
const toggleTheme = document.getElementById('toggle-theme');
function setTheme(mode){
  localStorage.setItem('theme', mode);
  document.documentElement.setAttribute('data-theme', mode);
}
toggleTheme.onclick = () =>
  setTheme((localStorage.getItem('theme')==='dark')?'light':'dark');
(function initTheme(){
  setTheme(localStorage.getItem('theme') || (window.matchMedia('(prefers-color-scheme:dark)').matches?'dark':'light'));
})();

// pamiętnik z edycją
const diaryText = document.getElementById('diary-text');
const saveDiary = document.getElementById('save-diary');
const todayKey = new Date().toISOString().split('T')[0];
diaryText.value = localStorage.getItem('diary-'+todayKey) || '';
saveDiary.onclick = () => {
  localStorage.setItem('diary-'+todayKey, diaryText.value);
  alert('Wpis zapisany na ' + todayKey);
};

// stoper z historią, usuwaniem i statystykami
let timer, startTime;
const startBtn = document.getElementById('start-timer');
const stopBtn = document.getElementById('stop-timer');
const taskDesc = document.getElementById('task-desc');
const display = document.getElementById('timer-display');
const historyDiv = document.getElementById('session-history');
const statsDiv = document.getElementById('stats');

function loadHistory(){
  const arr = JSON.parse(localStorage.getItem('sessions')) || [];
  historyDiv.innerHTML = '';
  const totals = {};
  arr.forEach((s, i)=>{
    totals[s.desc] = (totals[s.desc] || 0) + toSeconds(s.duration);
    const d = document.createElement('div');
    d.className = 'session-item';
    d.innerHTML = `<span>${s.desc}: ${s.duration}</span><button data-index="${i}" class="delete">Usuń</button>`;
    historyDiv.appendChild(d);
  });
  statsDiv.innerHTML = '';
  Object.entries(totals).forEach(([desc, secs])=>{
    statsDiv.appendChild(Object.assign(document.createElement('div'), {
      className:'stat-item',
      textContent: `${desc}: łączny czas ${formatHMS(secs)}`
    }));
  });
  [...historyDiv.querySelectorAll('button.delete')].forEach(btn=>{
    btn.onclick = () => {
      const idx = btn.getAttribute('data-index');
      const arr2 = JSON.parse(localStorage.getItem('sessions')) || [];
      arr2.splice(idx,1);
      localStorage.setItem('sessions', JSON.stringify(arr2));
      loadHistory();
    };
  });
}

startBtn.onclick = () => {
  if (!taskDesc.value.trim()) return alert('Dodaj opis zadania');
  startTime = Date.now();
  timer = setInterval(updateTimer,1000);
  startBtn.disabled=true; stopBtn.disabled=false;
};
stopBtn.onclick = () => {
  clearInterval(timer);
  const diff = Math.floor((Date.now()-startTime)/1000);
  const dur = formatHMS(diff);
  const arr = JSON.parse(localStorage.getItem('sessions')) || [];
  arr.unshift({ desc: taskDesc.value.trim(), duration: dur });
  localStorage.setItem('sessions', JSON.stringify(arr));
  taskDesc.value=''; display.textContent='00:00:00';
  startBtn.disabled=false; stopBtn.disabled=true;
  loadHistory();
};

function updateTimer(){
  const diff = Math.floor((Date.now()-startTime)/1000);
  display.textContent = formatHMS(diff);
}

function formatHMS(sec){
  const h=String(Math.floor(sec/3600)).padStart(2,'0');
  const m=String(Math.floor((sec%3600)/60)).padStart(2,'0');
  const s=String(sec%60).padStart(2,'0');
  return `${h}:${m}:${s}`;
}
function toSeconds(hms){
  const [h,m,s]=hms.split(':').map(Number);
  return h*3600 + m*60 + s;
}

loadHistory();
{
  "name": "Pamiętnik & Stoper",
  "short_name": "DiaryTimer",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#121212",
  "theme_color": "#121212",
  "icons": [
    {"src":"icon-192.png","sizes":"192x192","type":"image/png"},
    {"src":"icon-512.png","sizes":"512x512","type":"image/png"}
  ]
}
self.addEventListener('install', e =>
  e.waitUntil(
    caches.open('v1').then(c=>c.addAll(['.', 'index.html', 'style.css', 'script.js', 'manifest.json']))
  )
);
self.addEventListener('fetch', e =>
  e.respondWith(caches.match(e.request).then(r=>r || fetch(e.request)))
);
