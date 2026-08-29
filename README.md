<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Logic Gate Sandbox</title>
  <script src="https://www.gstatic.com/antigravity/web/dev/tailwindcss.min.js"></script>
  <style>
    /* Design tokens: the markup references these everywhere but nothing defined
       them, so standalone the page rendered as invisible-on-white. */
    :root {
      --background: #ffffff;
      --foreground: #0b0f19;
      --card: #f6f7f9;
      --border: #d8dce3;
      --primary: #10b981;
      --primary-foreground: #04170f;
      --muted-foreground: #6b7280;
      /* Canvas-only. --border is deliberately faint for UI chrome, which made the
         schematic wash out; gates and wires get their own stronger ink. */
      --wire: #4d5771;
      --gate-fill: #eaeef5;
      --gate-ink: #1b2233;
      /* Energised gates/wires. On white a brighter stroke LOSES contrast, so the
         outline alone can't carry the signal -- the lit body tint and the glow do. */
      --live: #059669;
      --live-fill: #c7f2df;
      --live-ink: #04372a;
      --row-live: #e4ebfb;
      --row-live-edge: #3b5bdb;
      color-scheme: light;
    }

    /* Three theme states. No data-theme = follow the OS; data-theme pins it. The
       media query is guarded so an explicit "light" still wins on a dark OS. */
    @media (prefers-color-scheme: dark) {
      :root:not([data-theme="light"]) {
      --background: #0b0f19;
      --foreground: #e7eaf0;
      --card: #151b28;
      --border: #2b3444;
      --primary: #34d399;
      --primary-foreground: #04170f;
      --muted-foreground: #8b94a5;
      --wire: #97a6bd;
      --gate-fill: #1e2739;
      --gate-ink: #f2f5fa;
      --live: #4ade80;
      --live-fill: #10402c;
      --live-ink: #dcfce7;
      --row-live: #1b2542;
      --row-live-edge: #7f9cff;
      color-scheme: dark;
      }
    }

    :root[data-theme="dark"] {
      --background: #0b0f19;
      --foreground: #e7eaf0;
      --card: #151b28;
      --border: #2b3444;
      --primary: #34d399;
      --primary-foreground: #04170f;
      --muted-foreground: #8b94a5;
      --wire: #97a6bd;
      --gate-fill: #1e2739;
      --gate-ink: #f2f5fa;
      --live: #4ade80;
      --live-fill: #10402c;
      --live-ink: #dcfce7;
      --row-live: #1b2542;
      --row-live-edge: #7f9cff;
      color-scheme: dark;
    }
    html, body {
      margin: 0;
      padding: 0;
      width: 100vw;
      height: 100vh;
      overflow: hidden;
    }
    canvas {
      touch-action: none;
    }
  </style>
</head>
<body class="bg-[var(--background)] text-[var(--foreground)] antialiased">
  <div class="h-screen w-screen flex flex-col p-4 gap-4 box-border">
    
    <!-- Top Navbar -->
    <div class="flex flex-wrap items-center justify-between border-b border-[var(--border)] pb-3 gap-3 shrink-0">
      <div>
        <h1 class="text-[var(--foreground)] font-bold text-lg leading-tight">Logic Gate Sandbox</h1>
      </div>
      <div class="flex gap-2 flex-wrap items-center">
        <button onclick="toggleRun()" id="run-btn" class="px-3 py-1 border border-[var(--border)] text-xs rounded-lg hover:bg-[var(--card)] font-mono">Run</button>
        <button onclick="stepOnce()" class="px-3 py-1 border border-[var(--border)] text-xs rounded-lg hover:bg-[var(--card)] font-mono">Step</button>
        <button onclick="deleteSelected()" id="del-btn" class="px-3 py-1 bg-red-600 text-white rounded-lg text-xs hover:opacity-90 font-mono disabled:opacity-40 disabled:cursor-not-allowed" disabled>Delete</button>
        <button onclick="cycleTheme()" id="theme-btn" class="px-3 py-1 border border-[var(--border)] text-xs rounded-lg hover:bg-[var(--card)] font-mono">Auto</button>
        <select onchange="loadPreset(this.value)" id="preset-select" class="px-2 py-1 border border-[var(--border)] bg-[var(--background)] text-[var(--foreground)] text-xs rounded-lg font-mono cursor-pointer"></select>
        <button onclick="clearCanvas()" class="px-3 py-1 border border-[var(--border)] text-xs rounded-lg hover:bg-[var(--card)] font-mono">Clear</button>
      </div>
    </div>

    <!-- Tool Selector Toolbar (Gates Row) -->
    <div class="flex flex-wrap items-center gap-1.5 p-2 bg-[var(--card)] border border-[var(--border)] rounded-xl shrink-0">
      <span class="text-xs font-mono text-[var(--muted-foreground)] mr-2">Tools:</span>
      <button onclick="setTool('select')" id="tool-select" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono font-bold transition-all bg-[var(--primary)] text-[var(--primary-foreground)] border-[var(--primary)]">
        Pointer
      </button>
      <div class="w-[1px] h-4 bg-[var(--border)] mx-1"></div>
      <button onclick="setTool('INPUT')" id="tool-INPUT" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        IN
      </button>
      <button onclick="setTool('OUTPUT')" id="tool-OUTPUT" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        OUT
      </button>
      <div class="w-[1px] h-4 bg-[var(--border)] mx-1"></div>
      <button onclick="setTool('AND')" id="tool-AND" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        AND
      </button>
      <button onclick="setTool('OR')" id="tool-OR" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        OR
      </button>
      <button onclick="setTool('XOR')" id="tool-XOR" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        XOR
      </button>
      <button onclick="setTool('NOT')" id="tool-NOT" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        NOT
      </button>
      <button onclick="setTool('NAND')" id="tool-NAND" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        NAND
      </button>
      <button onclick="setTool('NOR')" id="tool-NOR" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        NOR
      </button>
      <button onclick="setTool('XNOR')" id="tool-XNOR" class="tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] hover:bg-[var(--card)]">
        XNOR
      </button>
    </div>

    <!-- Main Workspace Splitter -->
    <div id="workspace" class="flex flex-1 flex-col md:flex-row gap-4 min-h-0 overflow-hidden">
      <!-- Interactive Schematic Area -->
      <div class="flex-1 border border-[var(--border)] bg-[var(--card)] rounded-xl overflow-hidden relative flex flex-col min-h-0">
        <div id="canvas-header" class="bg-[var(--background)] border-b border-[var(--border)] px-3 py-2 text-xs font-semibold shrink-0 flex justify-between">
          <span>Workspace Canvas</span>
          <span class="flex items-center gap-3">
            <span id="clip-indicator" class="text-[var(--muted-foreground)] hidden font-mono">0 on clipboard</span>
            <span id="stability-indicator" class="text-[#f59e0b] hidden font-mono font-bold">Not settling &mdash; oscillating</span>
            <span id="placement-indicator" class="text-[var(--primary)] hidden font-mono font-bold">Click canvas to place gate</span>
          </span>
        </div>
        <div class="flex-1 relative overflow-hidden bg-[var(--background)]">
          <canvas id="sandbox" class="w-full h-full block"></canvas>
        </div>
      </div>

      <!-- Drag to resize; double-click to reset -->
      <div id="splitter" title="Drag to resize · double-click to reset"
           class="hidden md:block w-1.5 shrink-0 self-stretch rounded-full cursor-col-resize bg-[var(--border)] hover:bg-[var(--primary)] transition-colors"></div>

      <!-- Truth Table Sidebar -->
      <div id="sidebar" class="md:w-80 w-full flex flex-col border border-[var(--border)] bg-[var(--card)] rounded-xl p-3 shrink-0 overflow-hidden min-h-[150px] md:min-h-0">
        <div class="shrink-0 pb-2 flex items-center justify-between gap-2 flex-wrap">
          <h3 class="text-sm font-semibold">Truth Tables</h3>
          <!-- Column toggles: internal gates, and the state(t)/state(t+1) pair -->
          <div class="flex items-center gap-3">
            <label class="flex items-center gap-1.5 cursor-pointer text-[10px] font-mono text-[var(--muted-foreground)] select-none">
              <input type="checkbox" id="toggle-gates" onchange="updateTruthTable()" class="accent-[var(--primary)] bg-[var(--background)] border-[var(--border)] rounded">
              Show Gates
            </label>
            <label class="flex items-center gap-1.5 cursor-pointer text-[10px] font-mono text-[var(--muted-foreground)] select-none">
              <input type="checkbox" id="toggle-state" onchange="updateTruthTable()" class="accent-[var(--primary)] bg-[var(--background)] border-[var(--border)] rounded">
              Show (t)
            </label>
          </div>
        </div>
        <div id="tables" class="flex-1 overflow-auto flex flex-col gap-3"></div>
      </div>
    </div>
  </div>

  <script>
    const canvas = document.getElementById('sandbox');
    const ctx = canvas.getContext('2d');
    
    let nodes = [];
    let connections = []; 
    let nextId = 1;
    
    let activeTool = 'select';
    
    let dragNode = null;
    let dragOffset = { x: 0, y: 0 };
    let connectionWire = null; 
    let selectionBox = null; 
    
    let selectedNodes = []; 
    let selectedConnection = null;

    let clipboard = null;
    let pasteCount = 0;

    // Logical (CSS-pixel) canvas size. canvas.width/height are the backing-store
    // size and are devicePixelRatio times larger, so all layout math uses these.
    let viewW = 0, viewH = 0;

    function resize() {
      const rect = canvas.getBoundingClientRect();
      if (!rect.width || !rect.height) return;
      const dpr = window.devicePixelRatio || 1;
      viewW = rect.width;
      viewH = rect.height;
      canvas.width = Math.round(rect.width * dpr);
      canvas.height = Math.round(rect.height * dpr);
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
      draw();
    }
    window.addEventListener('resize', resize);
    new ResizeObserver(resize).observe(canvas);

    // ---- preset circuits -------------------------------------------------------
    // A named `nm` is pinned (fixed) so assignNames leaves it alone; unnamed gates
    // get auto letters. `w(from, to, pin)` wires an output to input pin 0 or 1.
    const TALL = ['AND', 'OR', 'XOR', 'NAND', 'NOR', 'XNOR'];
    const nd = (id, type, x, y, nm) => ({
      id, type, name: nm || '', fixed: !!nm,
      x, y, w: 70, h: TALL.includes(type) ? 44 : 36, val: false
    });
    const w = (from, to, toInput = 0) => ({ from, to, toInput });

    const PRESETS = {
      'sr-latch': {
        label: 'SR Latch (NOR)',
        // Cross-coupled: Q = NOR(R, Q'), Q' = NOR(S, Q). Each NOR feeds the other,
        // so it remembers which input was asserted last -- pulse S and Q stays high.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',  60,  70, 'R'), nd(2, 'INPUT',  60, 250, 'S'),
            nd(3, 'NOR',   250,  66),      nd(4, 'NOR',   250, 246),
            nd(5, 'OUTPUT', 470,  70, 'Q'), nd(6, 'OUTPUT', 470, 250, "Q'")
          ],
          connections: [w(1,3,0), w(4,3,1), w(3,4,0), w(2,4,1), w(3,5), w(4,6)]
        })
      },

      'd-latch': {
        label: 'D Latch (gated)',
        // E gates the input: while E=1, Q follows D; drop E and Q freezes. This is
        // the SR latch with a steering front end that makes the illegal state
        // unreachable -- D and its inverse can never both be asserted.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   40,  70, 'D'), nd(2, 'INPUT',   40, 210, 'E'),
            nd(3, 'NOT',    150, 330),
            nd(4, 'NAND',   280,  80),      nd(5, 'NAND',   280, 280),
            nd(6, 'NAND',   450, 100),      nd(7, 'NAND',   450, 260),
            nd(8, 'OUTPUT', 600, 106, 'Q'), nd(9, 'OUTPUT', 600, 266, "Q'")
          ],
          connections: [
            w(1,3,0), w(1,4,0), w(2,4,1), w(3,5,0), w(2,5,1),
            w(4,6,0), w(7,6,1), w(6,7,0), w(5,7,1), w(6,8), w(7,9)
          ]
        })
      },

      'half-adder': {
        label: 'Half Adder',
        // XOR is the sum bit, AND is the carry. The two-gate core of all addition.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   60,  90, 'A'), nd(2, 'INPUT',   60, 250, 'B'),
            nd(3, 'XOR',    280, 108),      nd(4, 'AND',    280, 248),
            nd(5, 'OUTPUT', 480, 114, 'S'), nd(6, 'OUTPUT', 480, 254, 'C')
          ],
          connections: [w(1,3,0), w(2,3,1), w(1,4,0), w(2,4,1), w(3,5), w(4,6)]
        })
      },

      'full-adder': {
        label: 'Full Adder',
        // Two half adders plus an OR on the carries. Chain these and you have an
        // n-bit adder; Ci is the carry coming in from the stage below.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   30,  50, 'A'), nd(2, 'INPUT',   30, 150, 'B'),
            nd(3, 'INPUT',   30, 310, 'Ci'),
            nd(4, 'XOR',    170,  84),      nd(5, 'XOR',    330, 150),
            nd(6, 'AND',    330, 260),      nd(7, 'AND',    170, 360),
            nd(8, 'OR',     480, 306),
            nd(9, 'OUTPUT', 480, 154, 'S'), nd(10,'OUTPUT', 630, 310, 'Co')
          ],
          connections: [
            w(1,4,0), w(2,4,1),
            w(4,5,0), w(3,5,1),
            w(4,6,0), w(3,6,1),
            w(1,7,0), w(2,7,1),
            w(6,8,0), w(7,8,1),
            w(5,9), w(8,10)
          ]
        })
      },

      'mux': {
        label: '2:1 Multiplexer',
        // Sel picks which input reaches Q: Q = (A AND NOT Sel) OR (B AND Sel).
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   40,  70, 'A'), nd(2, 'INPUT',   40, 200, 'B'),
            nd(3, 'INPUT',   40, 340, 'Sel'),
            nd(4, 'NOT',    180, 260),
            nd(5, 'AND',    330,  90),      nd(6, 'AND',    330, 240),
            nd(7, 'OR',     490, 165),
            nd(8, 'OUTPUT', 630, 171, 'Q')
          ],
          connections: [
            w(1,5,0), w(4,5,1), w(3,4,0), w(2,6,0), w(3,6,1), w(5,7,0), w(6,7,1), w(7,8)
          ]
        })
      },

      'ring-osc': {
        label: 'Ring Oscillator',
        // Three inversions in a loop can never settle: whatever G1 is, the ring
        // demands it be the opposite. EN gates it -- at EN=0 the NAND pins the ring
        // high and it rests; at EN=1 it free-runs. Press Run to watch it flip, or
        // Step to walk it one propagation pass at a time.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   50, 182, 'EN'),
            nd(2, 'NAND',   200, 178),
            nd(3, 'NOT',    350, 182),
            nd(4, 'NOT',    490, 182),
            nd(5, 'OUTPUT', 630, 182, 'Q')
          ],
          connections: [w(1,2,0), w(4,2,1), w(2,3,0), w(3,4,0), w(2,5)]
        })
      },

      'xor-nand': {
        label: 'XOR from NANDs',
        // NAND is functionally complete -- every other gate can be built from it.
        // Four of them reproduce XOR exactly; compare against the XOR tool.
        build: () => ({
          nodes: [
            nd(1, 'INPUT',   40,  90, 'A'), nd(2, 'INPUT',   40, 290, 'B'),
            nd(3, 'NAND',   190, 178),
            nd(4, 'NAND',   350,  88),      nd(5, 'NAND',   350, 268),
            nd(6, 'NAND',   510, 178),
            nd(7, 'OUTPUT', 650, 184, 'Q')
          ],
          connections: [
            w(1,3,0), w(2,3,1), w(1,4,0), w(3,4,1), w(3,5,0), w(2,5,1),
            w(4,6,0), w(5,6,1), w(6,7)
          ]
        })
      }
    };

    function loadPreset(key) {
      const preset = PRESETS[key];
      if (!preset) return;
      stopRun();
      const built = preset.build();
      nodes = built.nodes;
      connections = built.connections;
      nextId = nodes.reduce((m, n) => Math.max(m, n.id), 0) + 1;

      const picker = document.getElementById('preset-select');
      if (picker) picker.value = key;

      clipboard = null;
      pasteCount = 0;
      renderClipboardHint();
      activeTool = 'select';
      setTool('select');
      clearSelection();
      assignNames();
      computeLogic(true);
      updateTruthTable();
      draw();
    }

    function loadLatch() { loadPreset('sr-latch'); }

    const THEMES = ['auto', 'light', 'dark'];
    const THEME_LABEL = { auto: 'Auto', light: 'Light', dark: 'Dark' };
    let theme = 'auto';

    function applyTheme(next) {
      theme = THEMES.includes(next) ? next : 'auto';
      const root = document.documentElement;
      if (root) {
        if (theme === 'auto') root.removeAttribute('data-theme');
        else root.setAttribute('data-theme', theme);
      }
      const btn = document.getElementById('theme-btn');
      if (btn) btn.textContent = THEME_LABEL[theme];
      try { localStorage.setItem('lgs-theme', theme); } catch (e) { /* file:// may block storage */ }
      draw();   // canvas colours are read out of the CSS vars at paint time
    }

    function cycleTheme() {
      applyTheme(THEMES[(THEMES.indexOf(theme) + 1) % THEMES.length]);
    }

    function initTheme() {
      let saved = null;
      try { saved = localStorage.getItem('lgs-theme'); } catch (e) { /* ignore */ }
      applyTheme(saved || 'auto');

      // Following the OS means repainting the canvas when the OS flips.
      if (window.matchMedia) {
        const mq = window.matchMedia('(prefers-color-scheme: dark)');
        const onChange = () => { if (theme === 'auto') draw(); };
        if (mq.addEventListener) mq.addEventListener('change', onChange);
        else if (mq.addListener) mq.addListener(onChange);
      }
    }

    function initPresetPicker() {
      const picker = document.getElementById('preset-select');
      if (!picker) return;
      picker.innerHTML = Object.keys(PRESETS)
        .map(k => `<option value="${k}">${PRESETS[k].label}</option>`).join('');
    }

    function init() {
      initPresetPicker();
      initSplitter();
      loadPreset('sr-latch');
      initTheme();
      resize();
    }

    // Auto-letters every node, except ones flagged `fixed` (preset circuits name
    // their own pins, e.g. S/R/Q/Q') -- those keep their name and their letter is
    // withheld from the pool so nothing else collides with it.
    // Spreadsheet-column naming (bijective base 26): 0->A .. 25->Z, 26->AA, 701->ZZ,
    // 702->AAA .. 18277->ZZZ. Note it is NOT plain base 26 -- there is no zero digit,
    // so 'A' carries value 1 in the leading position and the ranges never overlap.
    const MAX_NAME_INDEX = 26 + 26 * 26 + 26 * 26 * 26 - 1;   // 18277 == 'ZZZ'

    function nameAt(index) {
      let out = '';
      for (let i = index; i >= 0; i = Math.floor(i / 26) - 1) {
        out = String.fromCharCode(65 + (i % 26)) + out;
      }
      return out;
    }

    function assignNames() {
      const taken = new Set(nodes.filter(n => n.fixed).map(n => n.name));
      const byId = (a, b) => a.id - b.id;

      const outputPool = ['Q', 'Z', 'Y', 'X', 'W', 'V', 'U', 'T', 'S', 'R', 'P', 'O', 'N', 'M', 'L', 'K', 'J', 'I', 'H', 'G', 'F', 'E', 'D', 'C'];

      // Hands out the preferred letters first (outputs like to be Q, Z, ...), then
      // falls through to the endless A..Z, AA..ZZ, AAA..ZZZ run, skipping anything
      // already claimed. Both cursors only move forward, so this stays linear.
      const assign = (list, preferred) => {
        let p = 0, q = 0;
        list.forEach(node => {
          let name = null;
          while (name === null && p < preferred.length) {
            const candidate = preferred[p++];
            if (!taken.has(candidate)) name = candidate;
          }
          while (name === null && q <= MAX_NAME_INDEX) {
            const candidate = nameAt(q++);
            if (!taken.has(candidate)) name = candidate;
          }
          // Past ZZZ (18278 nodes of one kind) fall back to something still unique
          // rather than handing out duplicates.
          node.name = name !== null ? name : '#' + node.id;
          taken.add(node.name);
        });
      };

      assign(nodes.filter(n => n.type === 'OUTPUT' && !n.fixed).sort(byId), outputPool);
      assign(nodes.filter(n => n.type === 'INPUT' && !n.fixed).sort(byId), []);
      assign(nodes.filter(n => n.type !== 'INPUT' && n.type !== 'OUTPUT' && !n.fixed).sort(byId), []);
    }

    function setTool(tool) {
      activeTool = tool;
      document.querySelectorAll('.tool-btn').forEach(btn => {
        btn.className = "tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono transition-all bg-[var(--background)] border-[var(--border)] text-[var(--foreground)] hover:bg-[var(--card)]";
      });
      const activeBtn = document.getElementById(`tool-${tool}`);
      if (activeBtn) {
        activeBtn.className = "tool-btn px-2.5 py-1 text-xs rounded-lg border font-mono font-bold transition-all bg-[var(--primary)] text-[var(--primary-foreground)] border-[var(--primary)]";
      }

      const indicator = document.getElementById('placement-indicator');
      if (tool === 'select') {
        indicator.classList.add('hidden');
      } else {
        indicator.classList.remove('hidden');
        clearSelection();
      }
    }

    function placeGate(type, x, y) {
      const w = 70;
      const h = ['AND', 'OR', 'XOR', 'NAND', 'NOR', 'XNOR'].includes(type) ? 44 : 36;
      const node = {
        id: nextId++,
        type: type,
        name: '',
        x: x - w / 2,
        y: y - h / 2,
        w: w,
        h: h,
        val: false
      };
      nodes.push(node);
      assignNames();
      evaluate();
      setTool('select');
      draw();
    }

    function selectNode(node, append = false) {
      if (append) {
        if (!selectedNodes.includes(node)) {
          selectedNodes.push(node);
        }
      } else {
        selectedNodes = [node];
      }
      selectedConnection = null;
      document.getElementById('del-btn').disabled = false;
      draw();
    }

    function selectConnection(conn) {
      selectedConnection = conn;
      selectedNodes = [];
      document.getElementById('del-btn').disabled = false;
      draw();
    }

    function clearSelection() {
      selectedNodes = [];
      selectedConnection = null;
      document.getElementById('del-btn').disabled = true;
      draw();
    }

    // Copies the selected gates plus any wire whose BOTH ends are in the selection --
    // a wire with one end outside has nothing to reattach to in the copy.
    function copySelection() {
      if (!selectedNodes.length) return;
      const ids = new Set(selectedNodes.map(n => n.id));
      clipboard = {
        nodes: selectedNodes.map(n => ({
          srcId: n.id, type: n.type, x: n.x, y: n.y, w: n.w, h: n.h, val: n.val
        })),
        connections: connections
          .filter(c => ids.has(c.from) && ids.has(c.to))
          .map(c => ({ from: c.from, to: c.to, toInput: c.toInput }))
      };
      pasteCount = 0;
      renderClipboardHint();
    }

    function pasteClipboard() {
      if (!clipboard || !clipboard.nodes.length) return;
      pasteCount++;
      const shift = 26 * pasteCount;

      // Old id -> new id, so the copied wiring reattaches to the copies rather
      // than back to the originals.
      const idMap = new Map();
      const fresh = clipboard.nodes.map(src => {
        const node = {
          id: nextId++, type: src.type, name: '', w: src.w, h: src.h, val: src.val,
          x: Math.max(0, Math.min(Math.max(0, viewW - src.w), src.x + shift)),
          y: Math.max(0, Math.min(Math.max(0, viewH - src.h), src.y + shift))
        };
        idMap.set(src.srcId, node.id);
        return node;
      });

      nodes.push(...fresh);
      clipboard.connections.forEach(c => {
        connections.push({ from: idMap.get(c.from), to: idMap.get(c.to), toInput: c.toInput });
      });

      selectedNodes = fresh;
      selectedConnection = null;
      document.getElementById('del-btn').disabled = false;
      assignNames();
      evaluate();
      draw();
    }

    function renderClipboardHint() {
      const el = document.getElementById('clip-indicator');
      if (!el) return;
      const count = clipboard ? clipboard.nodes.length : 0;
      el.textContent = `${count} on clipboard`;
      el.classList.toggle('hidden', count === 0);
    }

    function deleteSelected() {
      if (selectedNodes.length > 0) {
        const ids = selectedNodes.map(n => n.id);
        nodes = nodes.filter(n => !ids.includes(n.id));
        connections = connections.filter(c => !ids.includes(c.from) && !ids.includes(c.to));
        clearSelection();
        assignNames();
        evaluate();
        draw();
      } else if (selectedConnection) {
        connections = connections.filter(c => c !== selectedConnection);
        clearSelection();
        evaluate();
        draw();
      }
    }

    window.addEventListener('keydown', e => {
      const t = e.target;
      if (t && (t.tagName === 'INPUT' || t.tagName === 'TEXTAREA' || t.isContentEditable)) return;
      const mod = e.metaKey || e.ctrlKey;   // Cmd on macOS, Ctrl elsewhere
      if (mod) {
        const k = e.key.toLowerCase();
        if (k === 'c') { e.preventDefault(); copySelection(); return; }
        if (k === 'v') { e.preventDefault(); pasteClipboard(); return; }
        if (k === 'x') { e.preventDefault(); copySelection(); deleteSelected(); draw(); return; }
        return;
      }

      if (e.key === 'Delete' || e.key === 'Backspace') {
        e.preventDefault(); // Backspace would otherwise navigate back
        deleteSelected();
      } else if (e.key === 'Escape') {
        setTool('select');
        clearSelection();
      }
    });

    function clearCanvas() {
      stopRun();
      nodes = [];
      connections = [];
      nextId = 1;
      clearSelection();
      evaluate();
      draw();
    }

    function getPorts(node) {
      const ports = { inputs: [], output: null };
      if (node.type !== 'INPUT') {
        const count = ['NOT', 'OUTPUT'].includes(node.type) ? 1 : 2;
        for (let i = 0; i < count; i++) {
          ports.inputs.push({
            x: node.x,
            y: node.y + (node.h / (count + 1)) * (i + 1)
          });
        }
      }
      if (node.type !== 'OUTPUT') {
        // NAND/NOR/XNOR draw their inverting bubble past the body's right edge,
        // so the port sits beyond it instead of underneath it.
        const bubble = ['NAND', 'NOR', 'XNOR'].includes(node.type) ? 7 : 0;
        ports.output = {
          x: node.x + node.w + bubble,
          y: node.y + node.h / 2
        };
      }
      return ports;
    }

    // Halo on energised gates/wires, in canvas px. Keep it subtle -- it is a hint
    // that a net is live, not a lighting effect. 0 disables it.
    const GLOW = 4;

    const MAX_ITERATIONS = 200;
    let unstable = false;

    // What this node's output should be, given whatever its sources currently hold.
    // Unconnected pins read as 0.
    function gateValue(node) {
      const incoming = connections.filter(c => c.to === node.id);
      const pinCount = ['NOT', 'OUTPUT'].includes(node.type) ? 1 : 2;
      const a = [];
      for (let i = 0; i < pinCount; i++) {
        const conn = incoming.find(c => c.toInput === i);
        const src = conn ? nodes.find(n => n.id === conn.from) : null;
        a.push(src ? src.val : false);
      }
      switch (node.type) {
        case 'OUTPUT': return a[0];
        case 'NOT':    return !a[0];
        case 'AND':    return a[0] && a[1];
        case 'OR':     return a[0] || a[1];
        case 'XOR':    return a[0] !== a[1];
        case 'NAND':   return !(a[0] && a[1]);
        case 'NOR':    return !(a[0] || a[1]);
        case 'XNOR':   return a[0] === a[1];
        default:       return false;
      }
    }

    // Relax the graph until nothing changes. Values are NOT cleared first: carrying
    // the previous pass forward is exactly what lets a feedback loop hold state
    // rather than collapsing to 0 on every recompute. Nodes in `frozen` are pinned
    // (used to explore a specific prior state). Returns false if it never settled.
    function settle(frozen, scope) {
      let changed = true;
      let iterations = 0;
      while (changed && iterations < MAX_ITERATIONS) {
        changed = false;
        iterations++;
        nodes.forEach(node => {
          if (node.type === 'INPUT') return;
          if (scope && !scope.has(node.id)) return;
          if (frozen && frozen.has(node.id)) return;
          const next = gateValue(node);
          if (node.val !== next) {
            node.val = next;
            changed = true;
          }
        });
      }
      return !changed;
    }

    function computeLogic(fromZero) {
      if (fromZero) nodes.forEach(n => { if (n.type !== 'INPUT') n.val = false; });
      unstable = !settle(null);
      const el = document.getElementById('stability-indicator');
      if (el) el.classList.toggle('hidden', !unstable);
      return !unstable;
    }

    // Nodes that a feedback edge points at -- i.e. the circuit's state bits. Plain
    // DFS over the wire graph; an edge into a node already on the stack is a loop.
    // Iterated in id order so the same node is picked run to run.
    function findStateNodes() {
      const adj = new Map(nodes.map(n => [n.id, []]));
      connections.forEach(c => { if (adj.has(c.from) && adj.has(c.to)) adj.get(c.from).push(c.to); });

      const ordered = nodes.slice().sort((a, b) => a.id - b.id);
      const color = new Map(nodes.map(n => [n.id, 0])); // 0 unseen, 1 on stack, 2 done
      const stateIds = new Set();

      const visit = start => {
        const stack = [{ id: start, next: 0 }];
        color.set(start, 1);
        while (stack.length) {
          const frame = stack[stack.length - 1];
          const kids = adj.get(frame.id);
          if (frame.next < kids.length) {
            const kid = kids[frame.next++];
            if (color.get(kid) === 1) stateIds.add(kid);
            else if (color.get(kid) === 0) { color.set(kid, 1); stack.push({ id: kid, next: 0 }); }
          } else {
            color.set(frame.id, 2);
            stack.pop();
          }
        }
      };
      ordered.forEach(n => { if (color.get(n.id) === 0) visit(n.id); });

      return ordered.filter(n => stateIds.has(n.id));
    }

    function evaluate() {
      // While the clock is running the stepper owns the board -- relaxing to a fixed
      // point here would just undo the step (and never terminate on an oscillator).
      if (!running) computeLogic();
      updateTruthTable();
    }

    // One propagation pass, all gates updated from the SAME previous instant rather
    // than in array order. That simultaneity is what makes a ring actually ring: an
    // in-place pass would race around the loop and land back where it started.
    function stepSync() {
      const next = nodes.map(n => (n.type === 'INPUT' ? n.val : gateValue(n)));
      nodes.forEach((n, i) => { n.val = next[i]; });
    }

    let running = false;
    let runTimer = null;
    let lastTableAt = 0;

    function stepOnce() {
      stopRun();
      stepSync();
      updateTruthTable();
      draw();
    }

    function tick() {
      stepSync();
      draw();
      // Re-deriving the tables is the expensive half, so cap it at ~4Hz even though
      // the board steps faster.
      const now = Date.now();
      if (now - lastTableAt > 240) { lastTableAt = now; updateTruthTable(); }
    }

    function stopRun() {
      if (!running) return;
      running = false;
      if (runTimer) { clearInterval(runTimer); runTimer = null; }
      const btn = document.getElementById('run-btn');
      if (btn) btn.textContent = 'Run';
      computeLogic();
      updateTruthTable();
      draw();
    }

    function toggleRun() {
      if (running) { stopRun(); return; }
      running = true;
      const btn = document.getElementById('run-btn');
      if (btn) btn.textContent = 'Pause';
      lastTableAt = 0;
      runTimer = setInterval(tick, 120);
    }

    function drawGateSymbol(n, activeCol, borderCol, fillCol) {
      const x = n.x;
      const y = n.y;
      const w = n.w;
      const h = n.h;
      
      ctx.beginPath();
      
      if (n.type === 'INPUT' || n.type === 'OUTPUT') {
        ctx.roundRect(x, y, w, h, 6);
      } 
      else if (n.type === 'AND' || n.type === 'NAND') {
        ctx.moveTo(x, y);
        ctx.lineTo(x + w * 0.5, y);
        ctx.arc(x + w * 0.5, y + h / 2, h / 2, -Math.PI / 2, Math.PI / 2);
        ctx.lineTo(x, y + h);
        ctx.closePath();
      } 
      else if (n.type === 'OR' || n.type === 'NOR') {
        ctx.moveTo(x, y);
        ctx.quadraticCurveTo(x + w * 0.25, y + h / 2, x, y + h);
        ctx.quadraticCurveTo(x + w * 0.6, y + h * 0.9, x + w, y + h / 2);
        ctx.quadraticCurveTo(x + w * 0.6, y + h * 0.1, x, y);
        ctx.closePath();
      } 
      else if (n.type === 'XOR' || n.type === 'XNOR') {
        ctx.moveTo(x - 5, y);
        ctx.quadraticCurveTo(x + w * 0.25 - 5, y + h / 2, x - 5, y + h);
        ctx.strokeStyle = selectedNodes.includes(n) ? '#ef4444' : (n.val ? activeCol : borderCol);
        ctx.stroke();
        
        ctx.beginPath();
        ctx.moveTo(x, y);
        ctx.quadraticCurveTo(x + w * 0.25, y + h / 2, x, y + h);
        ctx.quadraticCurveTo(x + w * 0.6, y + h * 0.9, x + w, y + h / 2);
        ctx.quadraticCurveTo(x + w * 0.6, y + h * 0.1, x, y);
        ctx.closePath();
      } 
      else if (n.type === 'NOT') {
        ctx.moveTo(x, y);
        ctx.lineTo(x + w - 10, y + h / 2);
        ctx.lineTo(x, y + h);
        ctx.closePath();
      }
      
      ctx.fill();
      ctx.stroke();

      if (['NOT', 'NAND', 'NOR', 'XNOR'].includes(n.type)) {
        ctx.beginPath();
        const r = 3.5;
        const bx = n.type === 'NOT' ? x + w - r - 3 : x + w + r;
        const by = y + h / 2;
        ctx.arc(bx, by, r, 0, Math.PI * 2);
        ctx.fillStyle = fillCol;
        ctx.fill();
        ctx.stroke();
      }
    }

    function getBezierPoint(t, p0, p1, p2, p3) {
      const cx = 3 * (p1.x - p0.x);
      const bx = 3 * (p2.x - p1.x) - cx;
      const ax = p3.x - p0.x - cx - bx;
      const cy = 3 * (p1.y - p0.y);
      const by = 3 * (p2.y - p1.y) - cy;
      const ay = p3.y - p0.y - cy - by;
      
      const xt = ax * Math.pow(t, 3) + bx * Math.pow(t, 2) + cx * t + p0.x;
      const yt = ay * Math.pow(t, 3) + by * Math.pow(t, 2) + cy * t + p0.y;
      return { x: xt, y: yt };
    }

    function getDistanceToConn(pos, c) {
      const fromNode = nodes.find(n => n.id === c.from);
      const toNode = nodes.find(n => n.id === c.to);
      if (!fromNode || !toNode) return Infinity;
      const start = getPorts(fromNode).output;
      const end = getPorts(toNode).inputs[c.toInput];
      if (!start || !end) return Infinity;
      
      const cp1 = { x: start.x + 40, y: start.y };
      const cp2 = { x: end.x - 40, y: end.y };
      
      let minDist = Infinity;
      for (let t = 0; t <= 1; t += 0.05) {
        const pt = getBezierPoint(t, start, cp1, cp2, end);
        const dx = pos.x - pt.x;
        const dy = pos.y - pt.y;
        const dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < minDist) minDist = dist;
      }
      return minDist;
    }

    function getMousePos(e) {
      const rect = canvas.getBoundingClientRect();
      const clientX = e.touches ? e.touches[0].clientX : e.clientX;
      const clientY = e.touches ? e.touches[0].clientY : e.clientY;
      return {
        x: clientX - rect.left,
        y: clientY - rect.top
      };
    }

    canvas.addEventListener('mousedown', startInteract);
    canvas.addEventListener('touchstart', startInteract);
    
    function startInteract(e) {
      const pos = getMousePos(e);

      if (activeTool !== 'select') {
        placeGate(activeTool, pos.x, pos.y);
        return;
      }
      
      // Output ports connection
      for (let n of nodes) {
        const ports = getPorts(n);
        if (ports.output) {
          const dx = pos.x - ports.output.x;
          const dy = pos.y - ports.output.y;
          if (dx*dx + dy*dy < 80) {
            connectionWire = { from: n.id, startX: ports.output.x, startY: ports.output.y, curX: pos.x, curY: pos.y };
            return;
          }
        }
      }

      // Node selection/dragging
      for (let i = nodes.length - 1; i >= 0; i--) {
        const n = nodes[i];
        if (pos.x >= n.x && pos.x <= n.x + n.w && pos.y >= n.y && pos.y <= n.y + n.h) {
          dragNode = n;
          dragOffset.x = pos.x - n.x;
          dragOffset.y = pos.y - n.y;
          if (!selectedNodes.includes(n)) {
            selectNode(n, e.shiftKey);
          }
          nodes.push(nodes.splice(i, 1)[0]);
          return;
        }
      }

      // Connection selection
      for (let c of connections) {
        if (getDistanceToConn(pos, c) < 8) {
          selectConnection(c);
          return;
        }
      }

      selectionBox = { x1: pos.x, y1: pos.y, x2: pos.x, y2: pos.y };
      clearSelection();
    }

    canvas.addEventListener('mousemove', moveInteract);
    canvas.addEventListener('touchmove', moveInteract);

    function moveInteract(e) {
      const pos = getMousePos(e);
      if (dragNode) {
        const dx = pos.x - dragOffset.x - dragNode.x;
        const dy = pos.y - dragOffset.y - dragNode.y;
        selectedNodes.forEach(n => {
          n.x = Math.max(0, Math.min(viewW - n.w, n.x + dx));
          n.y = Math.max(0, Math.min(viewH - n.h, n.y + dy));
        });
        draw();
      } else if (connectionWire) {
        connectionWire.curX = pos.x;
        connectionWire.curY = pos.y;
        draw();
      } else if (selectionBox) {
        selectionBox.x2 = pos.x;
        selectionBox.y2 = pos.y;
        draw();
      }
    }

    window.addEventListener('mouseup', endInteract);
    window.addEventListener('touchend', endInteract);

    function endInteract(e) {
      if (connectionWire) {
        const pos = { x: connectionWire.curX, y: connectionWire.curY };
        for (let n of nodes) {
          if (n.id === connectionWire.from) continue;
          const ports = getPorts(n);
          ports.inputs.forEach((port, idx) => {
            const dx = pos.x - port.x;
            const dy = pos.y - port.y;
            if (dx*dx + dy*dy < 80) {
              connections = connections.filter(c => !(c.to === n.id && c.toInput === idx));
              connections.push({ from: connectionWire.from, to: n.id, toInput: idx });
            }
          });
        }
        connectionWire = null;
        evaluate();
        draw();
      }

      if (selectionBox) {
        const xMin = Math.min(selectionBox.x1, selectionBox.x2);
        const xMax = Math.max(selectionBox.x1, selectionBox.x2);
        const yMin = Math.min(selectionBox.y1, selectionBox.y2);
        const yMax = Math.max(selectionBox.y1, selectionBox.y2);

        nodes.forEach(n => {
          const nodeCenterX = n.x + n.w / 2;
          const nodeCenterY = n.y + n.h / 2;
          if (nodeCenterX >= xMin && nodeCenterX <= xMax && nodeCenterY >= yMin && nodeCenterY <= yMax) {
            selectNode(n, true);
          }
        });
        selectionBox = null;
        draw();
      }
      
      dragNode = null;
    }

    canvas.addEventListener('dblclick', e => {
      const pos = getMousePos(e);
      for (let n of nodes) {
        if (n.type === 'INPUT' && pos.x >= n.x && pos.x <= n.x + n.w && pos.y >= n.y && pos.y <= n.y + n.h) {
          n.val = !n.val;
          evaluate();
          draw();
          break;
        }
      }
    });

    function draw() {
      ctx.clearRect(0, 0, viewW, viewH);
      const bodyStyle = getComputedStyle(document.body);
      const borderCol = bodyStyle.getPropertyValue('--wire').trim() || '#4d5771';
      const textCol = bodyStyle.getPropertyValue('--gate-ink').trim() || '#1b2233';
      const activeCol = bodyStyle.getPropertyValue('--live').trim() || '#059669';
      const gateFill = bodyStyle.getPropertyValue('--gate-fill').trim() || '#eaeef5';
      const liveFill = bodyStyle.getPropertyValue('--live-fill').trim() || '#c7f2df';
      const liveInk = bodyStyle.getPropertyValue('--live-ink').trim() || '#04372a';

      // Draw Connections (Curved splines from previous design)
      connections.forEach(c => {
        const fromNode = nodes.find(n => n.id === c.from);
        const toNode = nodes.find(n => n.id === c.to);
        if (!fromNode || !toNode) return;

        const start = getPorts(fromNode).output;
        const end = getPorts(toNode).inputs[c.toInput];
        if (!start || !end) return;

        ctx.beginPath();
        ctx.moveTo(start.x, start.y);
        ctx.bezierCurveTo(start.x + 40, start.y, end.x - 40, end.y, end.x, end.y);
        
        if (c === selectedConnection) {
          ctx.strokeStyle = '#ef4444';
          ctx.lineWidth = 4;
        } else {
          ctx.strokeStyle = fromNode.val ? activeCol : borderCol;
          ctx.lineWidth = fromNode.val ? 3 : 2.5;
          if (fromNode.val) { ctx.shadowColor = activeCol; ctx.shadowBlur = GLOW * 0.75; }
        }
        ctx.stroke();
        ctx.shadowBlur = 0;
      });

      // Connecting Draft Wire (Curved)
      if (connectionWire) {
        ctx.beginPath();
        ctx.moveTo(connectionWire.startX, connectionWire.startY);
        ctx.bezierCurveTo(connectionWire.startX + 40, connectionWire.startY, connectionWire.curX - 40, connectionWire.curY, connectionWire.curX, connectionWire.curY);
        ctx.strokeStyle = activeCol;
        ctx.lineWidth = 2;
        ctx.setLineDash([4, 4]);
        ctx.stroke();
        ctx.setLineDash([]);
      }

      // Draw Drag Selection Box
      if (selectionBox) {
        ctx.strokeStyle = activeCol;
        ctx.fillStyle = activeCol + '1a';
        ctx.lineWidth = 1;
        ctx.setLineDash([3, 3]);
        ctx.beginPath();
        ctx.rect(selectionBox.x1, selectionBox.y1, selectionBox.x2 - selectionBox.x1, selectionBox.y2 - selectionBox.y1);
        ctx.fill();
        ctx.stroke();
        ctx.setLineDash([]);
      }

      // Draw ANSI Gate Units
      nodes.forEach(n => {
        // A lit gate is signalled three ways at once -- tinted body, heavier green
        // outline, and a glow -- because on a white canvas no single one of them
        // reads strongly enough on its own.
        const lit = n.val;
        const body = lit ? liveFill : gateFill;
        ctx.fillStyle = body;
        
        if (selectedNodes.includes(n)) {
          ctx.strokeStyle = '#ef4444'; 
          ctx.lineWidth = 3;
        } else {
          ctx.strokeStyle = lit ? activeCol : borderCol;
          ctx.lineWidth = lit ? 3 : 2.25;
          if (lit) { ctx.shadowColor = activeCol; ctx.shadowBlur = GLOW; }
        }
        
        drawGateSymbol(n, activeCol, borderCol, body);
        ctx.shadowBlur = 0;   // never let the glow bleed onto the label below

        ctx.fillStyle = lit ? liveInk : textCol;
        ctx.font = 'bold 13px ui-monospace, SFMono-Regular, Menlo, monospace';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(n.name, n.x + n.w / 2, n.y + n.h / 2);

        const ports = getPorts(n);
        ctx.fillStyle = borderCol;
        ports.inputs.forEach(p => {
          ctx.beginPath();
          ctx.arc(p.x, p.y, 4.5, 0, Math.PI * 2);
          ctx.fill();
        });
        if (ports.output) {
          ctx.fillStyle = lit ? activeCol : borderCol;
          ctx.beginPath();
          ctx.arc(ports.output.x, ports.output.y, 5, 0, Math.PI * 2);
          ctx.fill();
        }
      });
    }

    const CELL = 'px-2 py-1.5 border-r border-[var(--border)] last:border-r-0';

    // Measured on this machine: a sweep costs about 1us per (row x node), and the
    // whole thing reruns on every input toggle, so cap the work near 250ms.
    const MAX_TABLE_ROWS = 8192;
    const TABLE_WORK_BUDGET = 250000;

    function bitCell(on, extra) {
      const tone = on ? 'text-[var(--primary)] font-bold' : 'text-[var(--muted-foreground)]';
      return `<td class="${CELL} ${extra || ''} ${tone}">${on ? '1' : '0'}</td>`;
    }

    function headCell(label, tone, extra) {
      return `<th class="${CELL} ${extra || ''} ${tone}">${label}</th>`;
    }

    // Connected components of the wire graph, wires treated as undirected. Two
    // circuits that share no wire are independent, so combining them into one table
    // is not just noisy -- it multiplies the row count by 2^(other circuit's inputs)
    // for no information at all.
    function findCircuits() {
      const adj = new Map(nodes.map(n => [n.id, []]));
      connections.forEach(c => {
        if (!adj.has(c.from) || !adj.has(c.to)) return;
        adj.get(c.from).push(c.to);
        adj.get(c.to).push(c.from);
      });

      const seen = new Set();
      const groups = [];
      nodes.slice().sort((a, b) => a.id - b.id).forEach(start => {
        if (seen.has(start.id)) return;
        const ids = [];
        const stack = [start.id];
        seen.add(start.id);
        while (stack.length) {
          const id = stack.pop();
          ids.push(id);
          adj.get(id).forEach(next => {
            if (!seen.has(next)) { seen.add(next); stack.push(next); }
          });
        }
        groups.push(new Set(ids));
      });
      return groups;
    }

    // One table for one independent circuit. Returns null when the group has no
    // input or no output, i.e. there is nothing to tabulate yet.
    function renderCircuit(scope, loops, showGates, showState) {
      const byId = (a, b) => a.id - b.id;
      const members = nodes.filter(n => scope.has(n.id));
      const inputs = members.filter(n => n.type === 'INPUT').sort(byId);
      const outputs = members.filter(n => n.type === 'OUTPUT').sort(byId);
      if (!inputs.length || !outputs.length) return null;

      const stateNodes = showState ? loops.filter(n => scope.has(n.id)) : [];
      const frozen = new Set(stateNodes.map(n => n.id));
      const gates = showGates
        ? members.filter(n => n.type !== 'INPUT' && n.type !== 'OUTPUT' && !frozen.has(n.id))
                 .sort((a, b) => a.name.localeCompare(b.name))
        : [];

      const sBits = stateNodes.length;
      const bits = inputs.length + sBits;
      const rowCount = Math.pow(2, bits);
      const signature = `${inputs.map(n => n.name).join(', ')} \u2192 ${outputs.map(n => n.name).join(', ')}`;

      const shell = inner => `<div class="border border-[var(--border)] rounded bg-[var(--background)] overflow-hidden shrink-0">`
        + `<div class="px-2 py-1 text-[10px] font-mono text-[var(--muted-foreground)] bg-[var(--card)] border-b border-[var(--border)]">${signature}</div>`
        + inner + `</div>`;

      if (rowCount > MAX_TABLE_ROWS || rowCount * members.length > TABLE_WORK_BUDGET) {
        return shell(`<div class="px-3 py-4 text-[10px] leading-relaxed text-[var(--muted-foreground)] text-center">`
          + `${inputs.length} input${inputs.length === 1 ? '' : 's'}${sBits ? ` and ${sBits} state bit${sBits === 1 ? '' : 's'}` : ''}`
          + ` means <b>${rowCount.toLocaleString()} rows</b> across ${members.length} nodes &mdash; too slow to redo on every toggle.<br>`
          + `${sBits ? 'Untick &ldquo;Show (t)&rdquo;, or remove' : 'Remove'} an input.</div>`);
      }

      let headHtml = '';
      inputs.forEach(i => headHtml += headCell(i.name, 'text-[var(--muted-foreground)]'));
      stateNodes.forEach(n => headHtml += headCell(`${n.name}(t)`, 'text-[var(--foreground)]'));
      gates.forEach(g => headHtml += headCell(g.name, 'text-[var(--foreground)]'));
      outputs.forEach(o => headHtml += headCell(o.name, 'text-[var(--primary)]'));
      stateNodes.forEach((n, idx) => headHtml += headCell(
        `${n.name}(t+1)`, 'text-[var(--primary)]',
        idx === 0 ? 'border-l border-[var(--border)]' : ''));

      // The sweep walks this circuit through every combination, so snapshot every
      // node's value first -- inputs, held state, and the other circuits on the
      // board -- and put it all back afterwards.
      const snapshot = nodes.map(n => n.val);

      // Which row the board is actually sitting on. With feedback there are several
      // rows per input combination -- one per prior state -- and only this one
      // describes the circuit right now.
      let liveRow = 0;
      inputs.forEach((n, i) => { if (n.val) liveRow |= 1 << (sBits + inputs.length - 1 - i); });
      stateNodes.forEach((n, i) => { if (n.val) liveRow |= 1 << (sBits - 1 - i); });

      let bodyHtml = '';
      for (let r = 0; r < rowCount; r++) {
        // Nothing is pinned when the state columns are hidden, so rewind to the
        // board's present state first -- otherwise each row would inherit whatever
        // the row above it left behind, which is history, not logic.
        if (!showState) nodes.forEach((n, i) => { n.val = snapshot[i]; });

        inputs.forEach((n, i) => { n.val = !!((r >> (sBits + inputs.length - 1 - i)) & 1); });
        stateNodes.forEach((n, i) => { n.val = !!((r >> (sBits - 1 - i)) & 1); });

        settle(frozen, scope);
        const next = stateNodes.map(n => gateValue(n));

        const here = r === liveRow;
        let row = `<tr data-live="${here ? '1' : '0'}" class="border-b border-[var(--border)] ${here
          ? 'bg-[var(--row-live)] shadow-[inset_3px_0_0_0_var(--row-live-edge)]'
          : 'hover:bg-[var(--card)]'}">`;
        inputs.forEach(i => row += bitCell(i.val));
        stateNodes.forEach(n => row += bitCell(n.val));
        gates.forEach(g => row += bitCell(g.val));
        outputs.forEach(o => row += bitCell(o.val));
        next.forEach((v, idx) => row += bitCell(v, idx === 0 ? 'border-l border-[var(--border)]' : ''));
        bodyHtml += row + '</tr>';
      }

      nodes.forEach((n, i) => { n.val = snapshot[i]; });

      return shell(`<div class="overflow-auto max-h-[55vh]">`
        + `<table class="min-w-full text-center text-xs font-mono">`
        + `<thead class="bg-[var(--card)] border-b border-[var(--border)] sticky top-0"><tr>${headHtml}</tr></thead>`
        + `<tbody>${bodyHtml}</tbody></table></div>`);
    }

    function updateTruthTable() {
      const container = document.getElementById('tables');
      if (!container) return;

      const showGates = document.getElementById('toggle-gates').checked;
      const showState = document.getElementById('toggle-state').checked;
      const loops = findStateNodes();

      const blocks = [];
      let halfBuilt = 0;

      findCircuits().forEach(scope => {
        const html = renderCircuit(scope, loops, showGates, showState);
        if (html) { blocks.push(html); return; }
        // Worth mentioning only when one side is wired and the other is missing;
        // a loose cluster of gates with no pins at all is just work in progress.
        const members = nodes.filter(n => scope.has(n.id));
        const hasIn = members.some(n => n.type === 'INPUT');
        const hasOut = members.some(n => n.type === 'OUTPUT');
        if (hasIn !== hasOut) halfBuilt++;
      });

      if (!blocks.length) {
        container.innerHTML = `<div class="px-2 py-8 text-[var(--muted-foreground)] text-center text-xs">Place inputs and outputs to evaluate logic.</div>`;
        return;
      }

      let html = blocks.join('');
      if (halfBuilt) {
        html += `<div class="px-1 text-[10px] font-mono text-[var(--muted-foreground)] shrink-0">`
          + `${halfBuilt} more group${halfBuilt === 1 ? '' : 's'} still needs an input and an output</div>`;
      }
      container.innerHTML = html;
    }

    // ---- sidebar splitter --------------------------------------------------------
    const SIDEBAR_DEFAULT = 320, SIDEBAR_MIN = 190, CANVAS_MIN = 260;
    let splitDrag = false;

    const wideLayout = () => !window.matchMedia || window.matchMedia('(min-width: 768px)').matches;

    function setSidebarWidth(px, persist) {
      const sidebar = document.getElementById('sidebar');
      const shell = document.getElementById('workspace');
      if (!sidebar || !shell) return;
      // Below the md breakpoint the panels stack, so an inline width would fight
      // the layout rather than resize it.
      if (!wideLayout()) { sidebar.style.width = ''; return; }
      const room = shell.getBoundingClientRect().width - CANVAS_MIN;
      const width = Math.round(Math.max(SIDEBAR_MIN, Math.min(px, Math.max(SIDEBAR_MIN, room))));
      sidebar.style.width = width + 'px';
      if (persist) { try { localStorage.setItem('lgs-sidebar', String(width)); } catch (e) { /* ignore */ } }
      return width;
    }

    function initSplitter() {
      const bar = document.getElementById('splitter');
      const sidebar = document.getElementById('sidebar');
      const shell = document.getElementById('workspace');
      if (!bar || !sidebar || !shell) return;

      let saved = null;
      try { saved = localStorage.getItem('lgs-sidebar'); } catch (e) { /* ignore */ }
      if (saved) setSidebarWidth(parseInt(saved, 10), false);

      const xOf = e => (e.touches && e.touches[0] ? e.touches[0].clientX : e.clientX);

      const begin = e => {
        splitDrag = true;
        if (document.body.style) document.body.style.cursor = 'col-resize';
        e.preventDefault();
      };
      const drag = e => {
        if (!splitDrag) return;
        setSidebarWidth(shell.getBoundingClientRect().right - xOf(e), false);
        e.preventDefault();
      };
      const finish = () => {
        if (!splitDrag) return;
        splitDrag = false;
        if (document.body.style) document.body.style.cursor = '';
        const width = parseInt(sidebar.style.width, 10);
        if (width) { try { localStorage.setItem('lgs-sidebar', String(width)); } catch (e) { /* ignore */ } }
      };

      bar.addEventListener('mousedown', begin);
      bar.addEventListener('touchstart', begin);
      window.addEventListener('mousemove', drag);
      window.addEventListener('touchmove', drag);
      window.addEventListener('mouseup', finish);
      window.addEventListener('touchend', finish);
      bar.addEventListener('dblclick', () => setSidebarWidth(SIDEBAR_DEFAULT, true));
      window.addEventListener('resize', () => { if (!wideLayout()) sidebar.style.width = ''; });
    }

    init();
  </script>
</body>
</html>
