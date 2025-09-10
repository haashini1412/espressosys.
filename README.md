<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Espresso World Map — Interactive</title>
<style>
  :root{
    --bg:#fff6f0;
    --card-bg:#3e210f; /* espresso dark brown header */
    --accent:#d88b57;  /* espresso warm accent */
    --map-frame-bg: #0f0f0f; /* inner map background */
    --pin-bg: #c66b29;
    --pin-stroke: rgba(0,0,0,0.25);
    font-family: Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
  }

  html,body{height:100%;margin:0;background:var(--bg);color:#222}
  .container{
    max-width:1100px;
    margin:36px auto;
    padding:18px;
  }

  .card{
    background: linear-gradient(180deg, var(--card-bg), #401f0d);
    border-radius:14px;
    box-shadow:0 10px 30px rgba(0,0,0,0.25);
    padding:18px;
    color:var(--accent);
  }

  .header{
    display:flex;
    align-items:center;
    gap:18px;
    padding:6px 8px 14px 8px;
  }

  .logo {
    height:44px;
    width:44px;
    display:flex;
    align-items:center;
    justify-content:center;
    background:transparent;
  }
  .title{
    font-size:28px;
    font-weight:600;
    color:var(--accent);
  }
  .subtitle{
    color:#f6e8de;
    margin-left:6px;
    font-weight:500;
    font-size:18px;
  }

  /* Map area */
  .map-wrapper{
    margin-top:12px;
    background:var(--map-frame-bg);
    border-radius:12px;
    padding:14px;
  }

  .map{
    position:relative;
    width:100%;
    /* 16:9-ish ratio */
    aspect-ratio: 16/9;
    border-radius:8px;
    overflow:hidden;
    background-size:cover;
    background-position:center center;
    box-shadow: inset 0 0 0 6px rgba(0,0,0,0.35);
  }

  /* Pins are positioned with left/top % */
  .pin{
    position:absolute;
    transform: translate(-50%,-100%) scale(1);
    cursor:pointer;
    transition: transform .12s ease, box-shadow .12s ease;
    display:flex;
    align-items:center;
    justify-content:center;
    width:34px;
    height:34px;
  }

  .pin .dot{
    width:20px;height:20px;border-radius:50%;
    background:var(--pin-bg);
    border:3px solid #fff6f0;
    box-shadow: 0 6px 14px var(--pin-stroke);
  }

  .pin:hover{ transform: translate(-50%,-100%) scale(1.08); }

  .tooltip{
    position:absolute;
    background: #24160f;
    color: #ffdcb8;
    padding:8px 10px;
    font-size:13px;
    border-radius:8px;
    pointer-events:none;
    transform: translate(-50%, -120%);
    white-space:nowrap;
    box-shadow: 0 6px 16px rgba(0,0,0,0.45);
    opacity:0;
    transition:opacity .12s ease;
  }
  .pin:hover + .tooltip, .tooltip.show { opacity:1; }

  /* Controls area */
  .controls{
    display:flex;
    gap:12px;
    margin-top:12px;
    flex-wrap:wrap;
  }

  .controls button, .controls input, .controls select {
    padding:8px 10px;
    border-radius:8px;
    border:1px solid rgba(0,0,0,0.12);
    background:#fff;
    font-size:14px;
  }

  .small { font-size:13px; padding:6px 8px; }

  .right{
    margin-left:auto;
    display:flex;
    gap:8px;
    align-items:center;
  }

  /* Simple responsive */
  @media (max-width:640px){
    .title{font-size:20px}
    .subtitle{font-size:14px}
    .pin{width:28px;height:28px}
    .pin .dot{width:14px;height:14px}
  }
</style>
</head>
<body>
  <div class="container">
    <div class="card">
      <div class="header">
        <div class="logo" aria-hidden>
          <!-- Replace with the Espresso logo SVG or image if you have it -->
          <svg width="44" height="44" viewBox="0 0 44 44" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Espresso logo">
            <circle cx="22" cy="22" r="20" fill="none" stroke="none"></circle>
            <!-- simplified sun motif -->
            <g transform="translate(2,2)" fill="#d88b57">
              <path d="M20 0v6h4v4h6v4h-6v4h-4v6h-4v-6h-4v-4H6V6h6V2h4V0z" opacity="0.12"/>
            </g>
          </svg>
        </div>
        <div>
          <div class="title">Espresso <span class="subtitle">World Map</span></div>
          <div style="font-size:12px;color:#f1d7c2;margin-top:6px">Interactive — hover markers to see details — exportable</div>
        </div>
        <div class="right">
          <button id="exportBtn" class="small">Export markers</button>
          <button id="resetBtn" class="small">Reset</button>
        </div>
      </div>

      <div class="map-wrapper">
        <!-- Change IMAGE_URL to your generated map image path or URL -->
        <div id="map" class="map" style="background-image:url('IMAGE_URL');">
          <!-- pins will be injected here -->
        </div>
      </div>

      <div class="controls" style="margin-top:14px">
        <label>
          Add marker title: <input id="markerTitle" placeholder="e.g. Espresso NYC" />
        </label>
        <label>
          X%: <input id="posX" type="number" min="0" max="100" value="30" style="width:72px"/>
        </label>
        <label>
          Y%: <input id="posY" type="number" min="0" max="100" value="45" style="width:72px"/>
        </label>
        <button id="addBtn">Add marker</button>

        <div style="margin-left:auto">
          <small style="color:#555">Tip: set IMAGE_URL to your PNG, then add markers. You can drag markers after adding by holding and moving.</small>
        </div>
      </div>
    </div>
  </div>

<script>
/*
 Simple interactive marker manager.
 Markers are positioned with left/top percent values.
 - You can add markers with title and coordinates
 - Hover shows tooltip
 - Export returns JSON to the console and downloads a file
 - Markers are draggable (mouse + touch)
*/

const mapEl = document.getElementById('map');
const addBtn = document.getElementById('addBtn');
const markerTitle = document.getElementById('markerTitle');
const posX = document.getElementById('posX');
const posY = document.getElementById('posY');
const exportBtn = document.getElementById('exportBtn');
const resetBtn = document.getElementById('resetBtn');

let markers = [
  // default sample markers (approx percent positions based on provided mock image)
  {id: cryptoRandomId(), title:'North America', x:30, y:45},
  {id: cryptoRandomId(), title:'South America', x:28, y:72},
  {id: cryptoRandomId(), title:'Africa', x:52, y:60},
  {id: cryptoRandomId(), title:'Asia', x:68, y:38},
  {id: cryptoRandomId(), title:'Australia', x:80, y:78},
];

function cryptoRandomId(){ return 'm_'+Math.random().toString(36).slice(2,9); }

function renderMarkers(){
  mapEl.innerHTML = ''; // clear
  markers.forEach(m=>{
    const pin = document.createElement('div');
    pin.className = 'pin';
    pin.style.left = m.x + '%';
    pin.style.top = m.y + '%';
    pin.dataset.id = m.id;

    // inner dot
    const dot = document.createElement('div');
    dot.className = 'dot';
    pin.appendChild(dot);

    // tooltip (sibling inside map so it follows)
    const tooltip = document.createElement('div');
    tooltip.className = 'tooltip';
    tooltip.textContent = m.title || '(untitled)';

    // click to remove
    pin.addEventListener('contextmenu', (ev)=>{
      ev.preventDefault();
      if(confirm('Delete marker "'+m.title+'"?')) {
        markers = markers.filter(x=>x.id!==m.id);
        renderMarkers();
      }
    });

    // drag support
    makeDraggable(pin, (clientX, clientY) => {
      // compute percent relative to mapEl
      const rect = mapEl.getBoundingClientRect();
      const px = Math.max(0, Math.min(1, (clientX - rect.left) / rect.width));
      const py = Math.max(0, Math.min(1, (clientY - rect.top) / rect.height));
      m.x = Math.round(px * 1000)/10;
      m.y = Math.round(py * 1000)/10;
      pin.style.left = m.x + '%';
      pin.style.top = m.y + '%';
    });

    // show tooltip on hover
    pin.addEventListener('mouseenter', () => tooltip.classList.add('show'));
    pin.addEventListener('mouseleave', () => tooltip.classList.remove('show'));

    mapEl.appendChild(pin);
    mapEl.appendChild(tooltip);

    // position tooltip above the pin (update on each render)
    const rect = mapEl.getBoundingClientRect();
    const left = (m.x/100) * rect.width + rect.left;
    const top = (m.y/100) * rect.height + rect.top;
    // but because we appended tooltip to map (not body), position using percent offsets:
    tooltip.style.left = m.x + '%';
    tooltip.style.top = (m.y - 5) + '%';
  });
}

// Very small draggable helper: pointer events
function makeDraggable(el, onMove){
  let dragging=false;
  let pointerId=null;

  el.addEventListener('pointerdown', (e)=>{
    e.preventDefault();
    dragging=true;
    pointerId = e.pointerId;
    el.setPointerCapture(pointerId);
    el.style.transition = 'none';
  });

  el.addEventListener('pointermove', (e)=>{
    if(!dragging || e.pointerId !== pointerId) return;
    onMove(e.clientX, e.clientY);
  });

  el.addEventListener('pointerup', (e)=>{
    if(e.pointerId !== pointerId) return;
    dragging=false;
    try{ el.releasePointerCapture(pointerId); }catch(e){}
    el.style.transition = '';
  });

  el.addEventListener('pointercancel', ()=>{ dragging=false; });
}

// UI actions
addBtn.addEventListener('click', ()=>{
  const title = markerTitle.value.trim() || 'Untitled';
  const x = Math.min(100, Math.max(0, Number(posX.value) || 50));
  const y = Math.min(100, Math.max(0, Number(posY.value) || 50));
  markers.push({id: cryptoRandomId(), title, x, y});
  renderMarkers();
  markerTitle.value=''; posX.value='50'; posY.value='50';
});

exportBtn.addEventListener('click', ()=>{
  const dataStr = JSON.stringify(markers, null, 2);
  console.log('Markers export:', markers);
  downloadFile('espresso-markers.json', dataStr);
});

resetBtn.addEventListener('click', ()=>{
  if(!confirm('Reset to default sample markers?')) return;
  markers = [
    {id: cryptoRandomId(), title:'North America', x:30, y:45},
    {id: cryptoRandomId(), title:'South America', x:28, y:72},
    {id: cryptoRandomId(), title:'Africa', x:52, y:60},
    {id: cryptoRandomId(), title:'Asia', x:68, y:38},
    {id: cryptoRandomId(), title:'Australia', x:80, y:78},
  ];
  renderMarkers();
});

// helper download
function downloadFile(filename, text) {
  const blob = new Blob([text], {type: 'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  setTimeout(()=>URL.revokeObjectURL(url), 60000);
}

// initial render
renderMarkers();

</script>
</body>
</html>
