// ============================================================================
// 3ICHI AUTO-TRADING DASHBOARD (ENTERPRISE SECURE VERSION)
// ============================================================================

const CONFIG = {
  DEFAULT_SYMBOL: 'BTCUSDT',
  TIMEFRAME: '15m',
  TRADE_QUANTITY: 0.001,
  CACHE_TTL: 60
};

// --- MODULE 1: SECURITY & CRYPTOGRAPHY ---
const Security = {
  async getCryptoKey(env) {
    const enc = new TextEncoder();
    const keyMaterial = await crypto.subtle.importKey('raw', enc.encode(env.ENCRYPTION_SECRET), { name: 'PBKDF2' }, false, ['deriveKey']);
    return await crypto.subtle.deriveKey({ name: 'PBKDF2', salt: enc.encode('salt-for-bot'), iterations: 100000, hash: 'SHA-256' }, keyMaterial, { name: 'AES-GCM', length: 256 }, false, ['encrypt', 'decrypt']);
  },
  async encrypt(env, text) {
    const key = await this.getCryptoKey(env);
    const iv = crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, new TextEncoder().encode(text));
    const combined = new Uint8Array(iv.length + encrypted.byteLength);
    combined.set(iv, 0); combined.set(new Uint8Array(encrypted), iv.length);
    return btoa(String.fromCharCode(...combined));
  },
  async decrypt(env, base64Data) {
    const key = await this.getCryptoKey(env);
    const combined = Uint8Array.from(atob(base64Data), c => c.charCodeAt(0));
    const iv = combined.slice(0, 12); const data = combined.slice(12);
    const decrypted = await crypto.subtle.decrypt({ name: 'AES-GCM', iv }, key, data);
    return new TextDecoder().decode(decrypted);
  },
  async signHMAC(secret, str) {
    const key = await crypto.subtle.importKey('raw', new TextEncoder().encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
    const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(str));
    return [...new Uint8Array(sig)].map(b => b.toString(16).padStart(2, '0')).join('');
  }
};

// --- MODULE 2: 3ICHI STRATEGY ---
const Strategy = {
  getHighest: (arr, len) => { let max = -Infinity; for (let i = arr.length - len; i < arr.length; i++) if (arr[i] > max) max = arr[i]; return max; },
  getLowest: (arr, len) => { let min = Infinity; for (let i = arr.length - len; i < arr.length; i++) if (arr[i] < min) min = arr[i]; return min; },
  getSMA: (arr, len) => { let sum = 0; for (let i = arr.length - len; i < arr.length; i++) sum += arr[i]; return sum / len; },
  getStdDev: (arr) => { const mean = arr.reduce((a, b) => a + b, 0) / arr.length; return Math.sqrt(arr.map(v => Math.pow(v - mean, 2)).reduce((a, b) => a + b, 0) / arr.length); },
  calcATR: (highs, lows, closes, len) => {
    let sum = 0;
    for (let i = highs.length - len; i < highs.length; i++) {
      if (i === 0) { sum += highs[i] - lows[i]; continue; }
      sum += Math.max(highs[i] - lows[i], Math.abs(highs[i] - closes[i-1]), Math.abs(lows[i] - closes[i-1]));
    }
    return sum / len;
  },
  calculate: (data) => {
    if (data.length < 250) return { error: 'NOT_ENOUGH_DATA' };
    const highs = data.map(d => d.high), lows = data.map(d => d.low), closes = data.map(d => d.close), opens = data.map(d => d.open);
    const t18 = (Strategy.getHighest(highs, 18) + Strategy.getLowest(lows, 18)) / 2;
    const t29 = (Strategy.getHighest(highs, 29) + Strategy.getLowest(lows, 29)) / 2;
    const t47 = (Strategy.getHighest(highs, 47) + Strategy.getLowest(lows, 47)) / 2;
    const atr = Strategy.calcATR(highs, lows, closes, 14);
    let t18Series = [];
    for (let i = 0; i < 10; i++) { const endIdx = highs.length - i; t18Series.push((Strategy.getHighest(highs.slice(0, endIdx), 18) + Strategy.getLowest(lows.slice(0, endIdx), 18)) / 2); }
    const isFlat = Strategy.getStdDev(t18Series) <= (atr * 0.03);
    const bodyAvg = Strategy.getSMA(opens.map((o, i) => Math.abs(closes[i] - o)), 10);
    const flatLevel = (t18 + t29 + t47) / 3;
    let trend = 0;
    if (t29 > (flatLevel + bodyAvg / 2)) trend = 1;
    else if (t29 < (flatLevel - bodyAvg / 2)) trend = -1;
    return { trend, t29: t29.toFixed(4), price: closes[closes.length - 1] };
  }
};

// --- MODULE 3: BROKER & DATA ---
const Broker = {
  async fetchCandles(symbol) {
    const cacheUrl = new URL(`https://cache.internal/${symbol}`);
    const cache = caches.default;
    let response = await cache.match(cacheUrl);
    if (!response) {
      const res = await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=${CONFIG.TIMEFRAME}&limit=250`);
      response = new Response(res.body, res);
      response.headers.set('Cache-Control', `public, max-age=${CONFIG.CACHE_TTL}`);
      await cache.put(cacheUrl, response.clone());
    }
    const d = await response.json();
    return d.map(c => ({ time: c[0]/1000, open: +c[1], high: +c[2], low: +c[3], close: +c[4] }));
  },
  async validateKeys(env, apiKey, apiSecret) {
    const isTestnet = env.BINANCE_TESTNET === "true";
    const baseUrl = isTestnet ? 'https://testnet.binance.vision' : 'https://api.binance.com';
    const timestamp = Date.now();
    const query = `timestamp=${timestamp}`;
    const sig = await Security.signHMAC(apiSecret, query);
    const res = await fetch(`${baseUrl}/api/v3/account?${query}&signature=${sig}`, { headers: { 'X-MBX-APIKEY': apiKey } });
    return res.ok;
  },
  async executeOrder(env, keys, symbol, side, qty) {
    const isTestnet = env.BINANCE_TESTNET === "true";
    const baseUrl = isTestnet ? 'https://testnet.binance.vision' : 'https://api.binance.com';
    const timestamp = Date.now();
    const query = `symbol=${symbol}&side=${side}&type=MARKET&quantity=${qty}&timestamp=${timestamp}`;
    const sig = await Security.signHMAC(keys.apiSecret, query);
    const res = await fetch(`${baseUrl}/api/v3/order?${query}&signature=${sig}`, { method: 'POST', headers: { 'X-MBX-APIKEY': keys.apiKey } });
    return await res.json();
  }
};

// --- MODULE 4: HTML UI (PURE ENGLISH, NO RTL) ---
const HTML_PAGE = `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>3ICHI Dashboard</title>
  <script src="https://unpkg.com/lightweight-charts/dist/lightweight-charts.standalone.production.js"></script>
  <style>
    :root { --bg: #0f0f13; --card: #1a1a24; --border: #2a2a35; --text: #e0e0e0; --green: #00b050; --red: #e53935; --blue: #2962ff; }
    body { font-family: -apple-system, system-ui; background: var(--bg); color: var(--text); margin: 0; padding: 0; }
    .header { background: var(--card); padding: 15px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid var(--border); }
    .container { padding: 15px; max-width: 600px; margin: auto; }
    .card { background: var(--card); border-radius: 12px; padding: 15px; margin-bottom: 15px; border: 1px solid var(--border); }
    .card-title { font-size: 14px; color: #888; margin-bottom: 10px; font-weight: 600; }
    #chart { width: 100%; height: 300px; border-radius: 8px; margin-bottom: 10px; }
    .stat-row { display: flex; justify-content: space-between; margin-bottom: 10px; font-size: 15px; }
    input, select { width: 100%; padding: 12px; border-radius: 8px; border: 1px solid var(--border); background: var(--bg); color: var(--text); margin-bottom: 10px; box-sizing: border-box; font-size: 16px; }
    button { width: 100%; padding: 14px; border: none; border-radius: 8px; font-size: 16px; font-weight: bold; cursor: pointer; margin-bottom: 10px; color: white; }
    .btn-blue { background: var(--blue); }
    .btn-green { background: var(--green); }
    .btn-red { background: var(--red); }
    .row { display: flex; gap: 10px; }
    .row button { flex: 1; }
    .switch { position: relative; display: inline-block; width: 50px; height: 28px; }
    .switch input { opacity: 0; width: 0; height: 0; }
    .slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #555; transition: .4s; border-radius: 28px; }
    .slider:before { position: absolute; content: ""; height: 20px; width: 20px; left: 4px; bottom: 4px; background-color: white; transition: .4s; border-radius: 50%; }
    input:checked + .slider { background-color: var(--green); }
    input:checked + .slider:before { transform: translateX(22px); }
    .log-box { background: #000; border-radius: 8px; padding: 10px; font-family: monospace; font-size: 12px; height: 100px; overflow-y: auto; color: #00ff00; }
    #loginModal { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.9); display: flex; justify-content: center; align-items: center; z-index: 100; }
    .hidden { display: none !important; }
  </style>
</head>
<body>
  <!-- Login Modal -->
  <div id="loginModal">
    <div class="card" style="width: 80%; max-width: 300px;">
      <h2 style="text-align:center; margin-top:0;">Login</h2>
      <input type="password" id="pwdInput" placeholder="Dashboard Password">
      <button class="btn-blue" onclick="login()">Enter</button>
    </div>
  </div>

  <div id="dashboard" class="hidden">
    <div class="header">
      <h1 style="margin:0; font-size:18px;">3ICHI Engine</h1>
      <span id="connStatus" style="font-size:12px; color:var(--red);">Offline</span>
    </div>

    <div class="container">
      <div class="card">
        <div class="stat-row">
          <span>Market:</span>
          <select id="symbolSelect" style="width:auto; margin:0; padding:5px;">
            <option value="BTCUSDT">BTC/USDT</option>
            <option value="ETHUSDT">ETH/USDT</option>
            <option value="SOLUSDT">SOL/USDT</option>
          </select>
        </div>
        <div id="chart"></div>
        <div class="stat-row"><span>Price:</span> <b id="lblPrice" style="color:var(--blue)">-</b></div>
        <div class="stat-row"><span>Signal:</span> <b id="lblSignal">-</b></div>
      </div>

      <div class="card">
        <div class="card-title">Broker Connection</div>
        <input type="text" id="inpKey" placeholder="Binance API Key">
        <input type="password" id="inpSecret" placeholder="Binance API Secret">
        <button class="btn-blue" onclick="connectBroker()">Connect Broker</button>
      </div>

      <div class="card">
        <div class="stat-row">
          <span style="font-weight:bold;">Auto-Trade Engine</span>
          <label class="switch">
            <input type="checkbox" id="autoToggle" onchange="toggleAuto()">
            <span class="slider"></span>
          </label>
        </div>
      </div>

      <div class="card">
        <div class="row">
          <button class="btn-green" onclick="manualTrade('BUY')">BUY</button>
          <button class="btn-red" onclick="manualTrade('SELL')">SELL</button>
        </div>
        <button class="btn-blue" style="background:#444;" onclick="refreshData()">Refresh Data</button>
      </div>

      <div class="card">
        <div class="log-box" id="logBox">Ready...</div>
      </div>
    </div>
  </div>

  <script>
    let authToken = '';
    
    function log(msg) {
      const box = document.getElementById('logBox');
      box.innerHTML = \`[\${new Date().toLocaleTimeString()}] \${msg}<br>\` + box.innerHTML;
    }

    function login() {
      const pwd = document.getElementById('pwdInput').value;
      if (!pwd) return alert('Enter password');
      authToken = pwd;
      document.getElementById('loginModal').classList.add('hidden');
      document.getElementById('dashboard').classList.remove('hidden');
      initChart();
      refreshData();
      setInterval(refreshData, 60000);
    }

    async function apiFetch(endpoint, options = {}) {
      options.headers = { ...options.headers, 'X-Auth-Token': authToken };
      const res = await fetch(endpoint, options);
      if (res.status === 401) {
        document.getElementById('loginModal').classList.remove('hidden');
        document.getElementById('dashboard').classList.add('hidden');
        throw new Error('Unauthorized');
      }
      return res;
    }

    let chart, candleSeries;
    function initChart() {
      chart = LightweightCharts.createChart(document.getElementById('chart'), {
        layout: { background: { color: '#1a1a24' }, textColor: '#888' },
        grid: { vertLines: { color: '#2a2a35' }, horzLines: { color: '#2a2a35' } },
        timeScale: { timeVisible: true, secondsVisible: false },
      });
      candleSeries = chart.addCandlestickSeries({ upColor: '#00b050', downColor: '#e53935', borderVisible: false, wickUpColor: '#00b050', wickDownColor: '#e53935' });
    }

    async function refreshData() {
      try {
        const sym = document.getElementById('symbolSelect').value;
        const res = await apiFetch(\`/api/data?symbol=\${sym}\`);
        const data = await res.json();
        
        candleSeries.setData(data.candles);
        chart.timeScale().fitContent();
        document.getElementById('lblPrice').innerText = data.signal.price;
        
        const sig = data.signal.trend === 1 ? 'BUY' : data.signal.trend === -1 ? 'SELL' : 'FLAT';
        const sigEl = document.getElementById('lblSignal');
        sigEl.innerText = sig;
        sigEl.style.color = sig === 'BUY' ? 'var(--green)' : sig === 'SELL' ? 'var(--red)' : 'var(--text)';

        document.getElementById('autoToggle').checked = data.autoTrade;
        document.getElementById('connStatus').innerText = data.brokerConnected ? "Online" : "Offline";
        document.getElementById('connStatus').style.color = data.brokerConnected ? 'var(--green)' : 'var(--red)';
      } catch (e) { log("Error: " + e.message); }
    }

    async function connectBroker() {
      const k = document.getElementById('inpKey').value;
      const s = document.getElementById('inpSecret').value;
      if (!k || !s) return log("Enter Key and Secret.");
      log("Validating keys...");
      try {
        const res = await apiFetch('/api/connect', { method: 'POST', body: JSON.stringify({ apiKey: k, apiSecret: s }) });
        const data = await res.json();
        if (data.success) { log("✅ Keys secured."); document.getElementById('inpKey').value = ''; document.getElementById('inpSecret').value = ''; refreshData(); } 
        else log("❌ " + data.error);
      } catch (e) { log("Failed."); }
    }

    async function toggleAuto() {
      const isChecked = document.getElementById('autoToggle').checked;
      await apiFetch('/api/toggle-auto', { method: 'POST', body: JSON.stringify({ status: isChecked }) });
      log(isChecked ? "Auto-Trade ON" : "Auto-Trade OFF");
    }

    async function manualTrade(side) {
      const sym = document.getElementById('symbolSelect').value;
      log(\`Sending \${side} order...\`);
      try {
        const res = await apiFetch('/api/trade', { method: 'POST', body: JSON.stringify({ side, symbol: sym }) });
        const data = await res.json();
        if (data.orderId) log(\`✅ Order: \${data.orderId}\`);
        else log(\`❌ Broker Error\`);
      } catch (e) { log("Trade failed."); }
    }

    document.getElementById('symbolSelect').addEventListener('change', async (e) => {
      const sym = e.target.value;
      await apiFetch('/api/set-symbol', { method: 'POST', body: JSON.stringify({ symbol: sym }) });
      refreshData();
    });
  </script>
</body>
</html>
`;

// --- MODULE 5: ROUTER & CRON ---
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // 1. Auth Check (Except for serving initial HTML)
    if (url.pathname === '/') {
      return new Response(HTML_PAGE, { headers: { 'Content-Type': 'text/html; charset=utf-8' } });
    }

    const authToken = request.headers.get('X-Auth-Token');
    if (env.DASHBOARD_PASSWORD && authToken !== env.DASHBOARD_PASSWORD) {
      return new Response('Unauthorized', { status: 401 });
    }

    // 2. API Routes
    try {
      if (url.pathname === '/api/data') {
        const symbol = url.searchParams.get('symbol') || CONFIG.DEFAULT_SYMBOL;
        const candles = await Broker.fetchCandles(symbol);
        const signal = Strategy.calculate(candles);
        const isAuto = await env.BOT_STATE.get('auto_trade') === 'true';
        const keys = await env.BOT_STATE.get('broker_keys');
        
        return Response.json({ candles: candles.slice(-100), signal, autoTrade: isAuto, brokerConnected: !!keys });
      }

      if (url.pathname === '/api/connect' && request.method === 'POST') {
        const body = await request.json();
        const isValid = await Broker.validateKeys(env, body.apiKey, body.apiSecret);
        if (!isValid) return Response.json({ success: false, error: "Invalid Keys" });

        const encrypted = await Security.encrypt(env, JSON.stringify({ apiKey: body.apiKey, apiSecret: body.apiSecret }));
        await env.BOT_STATE.put('broker_keys', encrypted);
        return Response.json({ success: true });
      }

      if (url.pathname === '/api/toggle-auto' && request.method === 'POST') {
        const body = await request.json();
        await env.BOT_STATE.put('auto_trade', body.status.toString());
        return Response.json({ success: true });
      }

      if (url.pathname === '/api/set-symbol' && request.method === 'POST') {
        const body = await request.json();
        await env.BOT_STATE.put('active_symbol', body.symbol);
        return Response.json({ success: true });
      }

      if (url.pathname === '/api/trade' && request.method === 'POST') {
        const body = await request.json();
        const keysEnc = await env.BOT_STATE.get('broker_keys');
        if (!keysEnc) return Response.json({ error: "No broker connected." });
        
        const keys = JSON.parse(await Security.decrypt(env, keysEnc));
        const order = await Broker.executeOrder(env, keys, body.symbol, body.side, CONFIG.TRADE_QUANTITY);
        
        // Sync trend state so Cron doesn't duplicate trades
        if (order.orderId) {
          const newTrend = body.side === 'BUY' ? 1 : -1;
          await env.BOT_STATE.put('last_trend', newTrend.toString());
        }
        return Response.json(order);
      }

    } catch (e) {
      return Response.json({ error: e.message }, { status: 500 });
    }

    return new Response('Not Found', { status: 404 });
  },

  // 3. Background Bot (Cron)
  async scheduled(event, env, ctx) {
    const isAuto = await env.BOT_STATE.get('auto_trade') === 'true';
    if (!isAuto) return;

    try {
      const symbol = await env.BOT_STATE.get('active_symbol') || CONFIG.DEFAULT_SYMBOL;
      const candles = await Broker.fetchCandles(symbol);
      const result = Strategy.calculate(candles);
      const lastTrend = parseInt(await env.BOT_STATE.get('last_trend') || '0');

      if (result.trend !== 0 && result.trend !== lastTrend) {
        const keysEnc = await env.BOT_STATE.get('broker_keys');
        if (keysEnc) {
          const keys = JSON.parse(await Security.decrypt(env, keysEnc));
          const side = result.trend === 1 ? 'BUY' : 'SELL';
          
          await Broker.executeOrder(env, keys, symbol, side, CONFIG.TRADE_QUANTITY);
          await env.BOT_STATE.put('last_trend', result.trend.toString());
        }
      }
    } catch (e) {
      console.error("Cron Error:", e);
    }
  }
};
