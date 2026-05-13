# Unidad 7

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
<img width="1919" height="1009" alt="image" src="https://github.com/user-attachments/assets/cbb03223-af75-4ea5-8e44-a161d383651a" />

### Códigos:
OSCAdapter.js
```.js
const dgram = require("dgram");
const BaseAdapter = require("./BaseAdapter");

class OpenStageControlAdapter extends BaseAdapter {
  constructor({ port = 9000, verbose = false } = {}) {
    super();

    this.portNumber = port;
    this.verbose = verbose;
    this.udp = null;
  }

  async connect() {
    if (this.connected) return;

    this.udp = dgram.createSocket("udp4");

    this.udp.on("message", (buffer) => {
      this._handleMessage(buffer);
    });

    this.udp.on("error", (err) => {
      this._fail(err);
    });

    this.udp.bind(this.portNumber, () => {
      this.connected = true;
      this.onConnected?.(`osc udp server open ${this.portNumber}`);

      if (this.verbose) {
        console.log(`Open Stage Control listening on UDP ${this.portNumber}`);
      }
    });
  }

  async disconnect() {
    if (!this.connected) return;

    this.connected = false;

    if (this.udp) {
      this.udp.close();
    }

    this.udp = null;
    this.onDisconnected?.("osc udp server closed");
  }

  getConnectionDetail() {
    return `osc udp server ${this.portNumber}`;
  }

  _handleMessage(buffer) {
    const oscMessage = this._parseOSCMessage(buffer);

    if (!oscMessage) return;

    const normalized = {
      type: "osc",
      timestamp: Date.now(),
      payload: {
        address: oscMessage.address,
        args: oscMessage.args
      }
    };

    if (this.verbose) {
      console.log(`[OSC] Mensaje recibido: ${oscMessage.address} args:`, oscMessage.args);
      console.log("[OSC] Normalizado:", JSON.stringify(normalized));
    }

    this.onData?.(normalized);
  }

  _parseOSCMessage(buffer) {
    try {
      let offset = 0;

      const addressData = this._readOSCString(buffer, offset);
      const address = addressData.value;
      offset = addressData.nextOffset;

      const typesData = this._readOSCString(buffer, offset);
      const types = typesData.value;
      offset = typesData.nextOffset;

      const args = [];

      for (let i = 1; i < types.length; i++) {
        const type = types[i];

        if (type === "f") {
          args.push(buffer.readFloatBE(offset));
          offset += 4;
        } else if (type === "i") {
          args.push(buffer.readInt32BE(offset));
          offset += 4;
        } else if (type === "s") {
          const stringData = this._readOSCString(buffer, offset);
          args.push(stringData.value);
          offset = stringData.nextOffset;
        }
      }

      return { address, args };
    } catch (err) {
      this._fail(err);
      return null;
    }
  }

  _readOSCString(buffer, offset) {
    let end = offset;

    while (end < buffer.length && buffer[end] !== 0) {
      end++;
    }

    const value = buffer.toString("utf8", offset, end);

    let nextOffset = end + 1;

    while (nextOffset % 4 !== 0) {
      nextOffset++;
    }

    return { value, nextOffset };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
  }

  async handleCommand(_cmd) {
    return;
  }
}

module.exports = OpenStageControlAdapter;
```

bridgeClient.js
```.js
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData = null;
    this._onConnect = null;
    this._onDisconnect = null;
    this._onStatus = null;
  }

  get isOpen() {
    return this._isOpen;
  }

  onData(callback) { this._onData = callback; }
  onConnect(callback) { this._onConnect = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback) { this._onStatus = callback; }

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) {
      this.close();
    }

    console.log(`[BridgeClient] Intentando conectar a ${this._url}`);
    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      console.log(`[BridgeClient] WebSocket abierto`);
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      let msg;

      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          console.log("[BridgeClient] Estado: CONECTADO");
          this._onConnect?.();
        }

        if (
          msg.state === "disconnected" ||
          msg.state === "error" ||
          msg.state === "ready"
        ) {
          this._isOpen = false;
          console.log(`[BridgeClient] Estado: ${msg.state.toUpperCase()}`);
          this._onDisconnect?.();

          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }
        }

        return;
      }

      if (
        msg.type === "microbit" ||
        msg.type === "strudel" ||
        msg.type === "osc"
      ) {
        this._onData?.(msg);
        return;
      }

      console.warn("Unknown message type from bridge:", msg);
    };

    this._ws.onerror = (err) => {
      console.error("[BridgeClient] ❌ WebSocket error:", err);
      console.error("⚠️  Verifica que el servidor esté corriendo:");
      console.error("   node bridgeServer.js --device strudelosc --strudelPort 8080 --oscPort 9000 --wsPort 8081");
    };

    this._ws.onclose = () => {
      console.warn("[BridgeClient] WebSocket cerrado");
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;

    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}
```
bridgeServer.js
```.js
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbit2
//     node bridgeServer.js --device microbitbinary
//     node bridgeServer.js --device strudel --strudelPort 8080 --wsPort 8081
//     node bridgeServer.js --device osc --oscPort 9000 --wsPort 8081
//     node bridgeServer.js --device strudelosc --strudelPort 8080 --oscPort 9000 --wsPort 8081

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");

const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitASCIIAdapter2 = require("./adapters/MicrobitASCIIAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OpenStageControlAdapter = require("./adapters/OSCAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};

function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() {
  return Date.now();
}

function safeJsonParse(s) {
  try {
    return JSON.parse(s);
  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);

  for (const client of wss.clients) {
    if (client.readyState === 1) {
      client.send(text);
    }
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
const OSC_PORT = parseInt(getArg("oscPort", "9000"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();

  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );

  return microbit?.path ?? null;
}

async function createAdapters() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bit found at ${path}`);

    return [
      new MicrobitAsciiAdapter({
        path,
        baud: BAUD,
        verbose: VERBOSE
      })
    ];
  }

  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bit2 found at ${path}`);

    return [
      new MicrobitASCIIAdapter2({
        path,
        baud: BAUD,
        verbose: VERBOSE
      })
    ];
  }

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }

    log.info(`micro:bitbinary found at ${path}`);

    return [
      new MicrobitBinaryAdapter({
        path,
        baud: BAUD,
        verbose: VERBOSE
      })
    ];
  }

  if (DEVICE === "strudel") {
    log.info(`Strudel adapter will listen on ws://127.0.0.1:${STRUDEL_PORT}`);

    return [
      new StrudelAdapter({
        port: STRUDEL_PORT,
        verbose: VERBOSE
      })
    ];
  }

  if (DEVICE === "osc") {
    log.info(`OSC adapter will listen on udp://127.0.0.1:${OSC_PORT}`);

    return [
      new OpenStageControlAdapter({
        port: OSC_PORT,
        verbose: VERBOSE
      })
    ];
  }

  if (DEVICE === "strudelosc") {
    log.info(`Strudel adapter will listen on ws://127.0.0.1:${STRUDEL_PORT}`);
    log.info(`OSC adapter will listen on udp://127.0.0.1:${OSC_PORT}`);

    return [
      new StrudelAdapter({
        port: STRUDEL_PORT,
        verbose: VERBOSE
      }),

      new OpenStageControlAdapter({
        port: OSC_PORT,
        verbose: VERBOSE
      })
    ];
  }

  return [
    new SimAdapter({
      hz: SIM_HZ
    })
  ];
}

function normalizeData(d) {
  if (d?.type === "strudel") {
    return d;
  }

  if (d?.type === "osc") {
    return d;
  }

  return {
    type: "microbit",
    x: d.x,
    y: d.y,
    btnA: !!d.btnA,
    btnB: !!d.btnB,
    t: nowMs()
  };
}

function getAdaptersDetail(adapters) {
  return adapters
    .map(adapter => adapter.getConnectionDetail?.() ?? "adapter connected")
    .join(" | ");
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });

  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapters = await createAdapters();

  for (const adapter of adapters) {
    adapter.onConnected = (detail) => {
      log.info(`[ADAPTER] Device Connected: ${detail}`);
      status(wss, "connected", detail);
    };

    adapter.onDisconnected = (detail) => {
      log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
      status(wss, "disconnected", detail);
    };

    adapter.onError = (detail) => {
      log.error(`[ADAPTER] Device Error: ${detail}`);
      status(wss, "error", detail);
    };

    adapter.onData = (d) => {
      broadcast(wss, normalizeData(d));
    };
  }

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const anyConnected = adapters.some(adapter => adapter.connected);

    const state = anyConnected ? "connected" : "ready";

    const detail = anyConnected
      ? getAdaptersDetail(adapters)
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({
      type: "status",
      state,
      detail,
      t: nowMs()
    }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        try {
          for (const adapter of adapters) {
            if (!adapter.connected) {
              await adapter.connect();
            }
          }

          status(wss, "connected", getAdaptersDetail(adapters));
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);

        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapter kept open. Shared with ${wss.clients.size - 1} other active client(s).`);

          ws.send(JSON.stringify({
            type: "status",
            state: "disconnected",
            detail: "logical disconnect only",
            t: nowMs()
          }));

          return;
        }

        try {
          for (const adapter of adapters) {
            await adapter.disconnect();
          }
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }

      if (msg.cmd === "setSimHz") {
        const simAdapter = adapters.find(adapter => adapter instanceof SimAdapter);

        if (simAdapter) {
          log.info(`Setting Sim Hz to ${msg.hz}`);
          await simAdapter.handleCommand(msg);
          status(wss, "connected", `sim hz=${simAdapter.hz}`);
        }

        return;
      }

      if (msg.cmd === "setLed") {
        try {
          for (const adapter of adapters) {
            await adapter.handleCommand?.(msg);
          }
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }

        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);

      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");

        for (const adapter of adapters) {
          adapter.disconnect();
        }
      }
    });
  });

  if (
    DEVICE === "sim" ||
    DEVICE === "strudel" ||
    DEVICE === "osc" ||
    DEVICE === "strudelosc"
  ) {
    for (const adapter of adapters) {
      await adapter.connect();
    }
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
indexStrudelOSC.html
```.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Painter – Strudel + OSC Visuals</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="fsm.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketchStrudel.js"></script>
</head>
<body>
  <button id="connectBtn" style="position: fixed; top: 10px; left: 10px; z-index: 999;">Connect</button>
</body>
</html>
```
sketchStrudel.js
```.html
const EVENTS = {
  CONNECT: "CONNECT",
  DISCONNECT: "DISCONNECT",
  DATA: "DATA",
  OSC_CONTROL: "OSC_CONTROL", // ✅ nuevo evento
};

class StrudelTask extends FSMTask {
  constructor() {
    super();

    // Cola temporal
    this.eventQueue = [];
    this.currentSound = null;
    this.currentDelta = 0;
    this.flashAlpha = 0;

    // Historial visual
    this.history = [];

    // ✅ Estado persistente OSC
    this.controls = {
      rgb: [255, 120, 30],
      sizeMultiplier: 1,
      trail: true
    };

    this.transitionTo(this.estado_esperando);
  }

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      console.log("[FSM] Esperando conexión...");
    } 
    
    else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      console.log("[FSM] Conectado — escuchando Strudel");
      this.eventQueue = [];
      this.history = [];
    }

    else if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
    }

    else if (ev.type === EVENTS.DATA) {
      this.updateLogic(ev.payload);
    }

    // ✅ integración OSC
    else if (ev.type === EVENTS.OSC_CONTROL) {
      this.updateControls(ev.payload);
    }
  };

  // ─────────────────────────────
  // STRUDEL LOGIC
  // ─────────────────────────────
  updateLogic(data) {
    if (data.type !== "strudel") return;

    this.eventQueue.push(data);
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);

    const now = Date.now();

    while (
      this.eventQueue.length > 0 &&
      this.eventQueue[0].timestamp <= now
    ) {
      const evt = this.eventQueue.shift();

      this.currentSound = evt.payload.s;
      this.currentDelta = evt.payload.delta ?? 0.5;
      this.flashAlpha = 255;

      this.history.push({
        sound: this.currentSound,
        delta: this.currentDelta,
        alpha: 255,
        x: random(width),
        y: random(height),
      });

      if (this.history.length > 12) this.history.shift();

      console.log(
        "[STRUDEL] exec:",
        this.currentSound,
        "| delta:",
        this.currentDelta,
        "| lag:",
        now - evt.timestamp,
        "ms"
      );
    }
  }

  // ─────────────────────────────
  // OSC CONTROLS
  // ─────────────────────────────
  updateControls(data) {
    console.log("[OSC UPDATE] Datos recibidos:", data);
    
    const payload = data.payload;
    console.log("[OSC UPDATE] Payload:", payload);
    
    if (!payload) {
      console.error("[OSC ERROR] No hay payload");
      return;
    }

    const { address, args } = payload;
    console.log(`[OSC UPDATE] Address: ${address}, Args:`, args);

    if (address === "/rgb_1") {
      if (!args || args.length < 3) {
        console.warn("[OSC WARN] /rgb_1 necesita 3 argumentos, recibió:", args?.length || 0);
        return;
      }
      this.controls.rgb = [args[0], args[1], args[2]];
      console.log("[OSC UPDATE] ✅ RGB actualizado a:", this.controls.rgb);
    } 
    
    else if (address === "/visual_size") {
      if (!args || args.length < 1) {
        console.warn("[OSC WARN] /visual_size necesita 1 argumento");
        return;
      }
      this.controls.sizeMultiplier = args[0];
      console.log("[OSC UPDATE] ✅ Size actualizado a:", this.controls.sizeMultiplier);
    } 
    
    else if (address === "/trail") {
      if (!args || args.length < 1) {
        console.warn("[OSC WARN] /trail necesita 1 argumento");
        return;
      }
      this.controls.trail = Boolean(args[0]);
      console.log("[OSC UPDATE] ✅ Trail actualizado a:", this.controls.trail);
    } 
    
    else {
      console.log(`[OSC] Dirección no reconocida: ${address}`);
    }
  }
}

// ─────────────────────────────────────────────
// GLOBALS
// ─────────────────────────────────────────────
let task;
let bridge;
const renderer = new Map();

// ─────────────────────────────────────────────
// COLOR
// ─────────────────────────────────────────────
function soundColor(s, alpha, rgb) {
  // ✅ SIEMPRE usar el color OSC (rgb) como base
  // Variar solo la saturación/brillo según el sonido
  
  const [r, g, b] = rgb;
  
  if (!s) {
    return color(r, g, b, alpha);
  }

  // Variar brillo según el tipo de sonido
  if (s.includes("bd")) {
    // Kick drum - más brillante
    return color(min(r * 1.3, 255), min(g * 1.3, 255), min(b * 1.3, 255), alpha);
  }
  
  if (s.includes("sd") || s.includes("cp")) {
    // Snare/clap - normal
    return color(r, g, b, alpha);
  }
  
  if (s.includes("hh") || s.includes("oh")) {
    // Hi-hat/open hat - más oscuro
    return color(r * 0.7, g * 0.7, b * 0.7, alpha);
  }

  // Default - usar color OSC
  return color(r, g, b, alpha);
}

// ─────────────────────────────────────────────
// SETUP
// ─────────────────────────────────────────────
function setup() {
  createCanvas(windowWidth, windowHeight);
  colorMode(RGB);
  noStroke();

  task = new StrudelTask();
  bridge = new BridgeClient();

  console.log("🎨 Painter + Strudel + OSC Initialized");
  console.log("Intentando conectar a ws://127.0.0.1:8081");

  bridge.onConnect(() => {
    console.log("✅ [BRIDGE] Conectado al servidor");
    const btn = document.getElementById("connectBtn");
    if (btn) btn.innerText = "Disconnect";

    task.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    console.log("❌ [BRIDGE] Desconectado");
    const btn = document.getElementById("connectBtn");
    if (btn) btn.innerText = "Connect";

    task.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => {
    console.log(`📡 [BRIDGE] ${s.state}${s.detail ? ": " + s.detail : ""}`);
  });

  // ✅ STRUDEL + OSC
  bridge.onData((data) => {
    if (data.type === "strudel") {
      console.log(`🎵 [STRUDEL] Evento recibido:`, data.payload);
      task.postEvent({ type: EVENTS.DATA, payload: data });
    } 
    
    else if (data.type === "osc") {
      console.log(`🎚️ [OSC] Control recibido: ${data.payload.address}`, data.payload.args);
      task.postEvent({ type: EVENTS.OSC_CONTROL, payload: data });
    }
  });

  // botón seguro
  const btn = document.getElementById("connectBtn");
  if (btn) {
    btn.addEventListener("click", () => {
      if (bridge.isOpen) {
        console.log("Desconectando...");
        bridge.close();
      } else {
        console.log("Conectando a ws://127.0.0.1:8081");
        bridge.open();
      }
    });
    console.log("✓ Botón 'Connect' inicializado");
  } else {
    console.error("❌ No se encontró el botón 'connectBtn'");
  }

  renderer.set(task.estado_corriendo, drawRunning);
  renderer.set(task.estado_esperando, drawWaiting);
}

// ─────────────────────────────────────────────
function draw() {
  task.update();
  renderer.get(task.state)?.();
}

// ─────────────────────────────────────────────
function drawWaiting() {
  background(0);
  fill(255, 255, 255, 120);
  textAlign(CENTER, CENTER);
  textSize(18);
  text("Conecta para comenzar", width / 2, height / 2);
}

// ─────────────────────────────────────────────
function drawRunning() {

  // ✅ control trail OSC
  if (task.controls.trail) {
    background(0, 30);
  } else {
    background(0);
  }

  // ── TRAIL
  for (let i = 0; i < task.history.length; i++) {
    const h = task.history[i];

    const size =
      constrain(h.delta * 400, 20, 600) *
      task.controls.sizeMultiplier;

    fill(soundColor(h.sound, h.alpha, task.controls.rgb));
    circle(h.x, h.y, size);

    task.history[i].alpha = max(0, h.alpha - 8);
  }

  // ── FLASH CENTRAL
  if (task.flashAlpha > 0) {
    const size =
      constrain(task.currentDelta * 500, 40, 700) *
      task.controls.sizeMultiplier;

    fill(
      soundColor(
        task.currentSound,
        task.flashAlpha,
        task.controls.rgb
      )
    );

    circle(width / 2, height / 2, size);

    task.flashAlpha = max(0, task.flashAlpha - 20);
  }
}

// ─────────────────────────────────────────────
function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```




## Bitácora de reflexión
