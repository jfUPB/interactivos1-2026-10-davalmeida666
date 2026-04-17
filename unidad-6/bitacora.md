# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
<img width="1893" height="943" alt="image" src="https://github.com/user-attachments/assets/ec50a68f-bfeb-4a3a-8cd6-2422d51ccf9d" />
<img width="1042" height="718" alt="image" src="https://github.com/user-attachments/assets/c12469c4-9701-43f0-8c38-52fd47c49ef5" />
<img width="610" height="281" alt="image" src="https://github.com/user-attachments/assets/d2f910f3-1646-4988-bfa6-29e058acd2a1" />

### Código

### index_strudel.html

``` .html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Strudel Visual</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #000; overflow: hidden; }
    #connectBtn {
      position: fixed;
      top: 10px;
      left: 10px;
      z-index: 100;
      padding: 8px 20px;
      font-size: 16px;
      cursor: pointer;
      background: #222;
      color: #fff;
      border: 1px solid #555;
      border-radius: 6px;
    }
    #connectBtn:hover { background: #444; }
  </style>
</head>
<body>
  <button id="connectBtn">Connect</button>

  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="fsm.js"></script>
  <script src="sketch_strudel.js"></script>
</body>
</html>
```
## skecth_strudel.js
```.js
const EVENTS = {
  CONNECT:    "CONNECT",
  DISCONNECT: "DISCONNECT",
  DATA:       "DATA",
};

class StrudelTask extends FSMTask {
  constructor() {
    super();

    // Cola temporal
    this.eventQueue   = [];
    this.currentSound = null;
    this.currentDelta = 0;
    this.flashAlpha   = 0;

    // Historial de flashes para efecto trail
    this.history = [];

    this.transitionTo(this.estado_esperando);
  }

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      console.log("[FSM] Esperando conexión...");
    } else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      console.log("[FSM] Conectado — escuchando Strudel");
      this.eventQueue = [];
      this.history    = [];
    }

    else if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
    }

    else if (ev.type === EVENTS.DATA) {
      this.updateLogic(ev.payload);
    }
  };

  updateLogic(data) {
    if (data.type !== "strudel") return;

    // Guardar en cola y ordenar por timestamp
    this.eventQueue.push(data);
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);

    const now = Date.now();

    while (this.eventQueue.length > 0 &&
           this.eventQueue[0].timestamp <= now) {

      const evt = this.eventQueue.shift();

      this.currentSound = evt.payload.s;
      this.currentDelta = evt.payload.delta ?? 0.5;
      this.flashAlpha   = 255;

      // Guardar en historial para trail
      this.history.push({
        sound: this.currentSound,
        delta: this.currentDelta,
        alpha: 255,
        x: random(width),
        y: random(height),
      });

      // Máximo 12 en historial
      if (this.history.length > 12) this.history.shift();

      console.log("[STRUDEL] exec:", this.currentSound,
                  "| delta:", this.currentDelta,
                  "| lag:", now - evt.timestamp, "ms");
    }
  }
}
```
## StrudelAdapter.js
```.js
const { WebSocketServer } = require("ws");
const BaseAdapter = require("./BaseAdapter");

class StrudelAdapter extends BaseAdapter {
constructor({ port = 8080 } = {}) {
    super();
    this.port = port;
    this.wss  = null;
  }

  async connect() {
    if (this.connected) return;

    return new Promise((resolve) => {
      this.wss = new WebSocketServer({ port: this.port });

      this.wss.on("listening", () => {
        this.connected = true;
        this.onConnected?.(`strudel server escuchando en ws://localhost:${this.port}`);
        console.log(`[StrudelAdapter] Servidor WS abierto en puerto ${this.port}`);
        resolve();
      });

      this.wss.on("connection", (ws) => {
        console.log("[StrudelAdapter] Strudel conectado");

        ws.on("message", (data) => {
          let msg;
          try {
            msg = JSON.parse(data.toString());
          } catch (e) {
            console.warn("[StrudelAdapter] JSON inválido:", data.toString());
            return;
          }
          this._handleMessage(msg);
        });

        ws.on("close", () => {
          console.log("[StrudelAdapter] Strudel desconectado");
        });
      });

      this.wss.on("error", (e) => this._fail(e));
    });
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;
    this.wss?.close();
    this.wss = null;
    this.onDisconnected?.("strudel server cerrado");
  }

  getConnectionDetail() {
    return `strudel ws://localhost:${this.port}`;
  }

  _handleMessage(msg) {
    if (msg.address !== "/dirt/play") return;

    const parsed = this._parseArgs(msg.args ?? []);
    if (!parsed.s) return;

    this.onData?.({
      type: "strudel",
      timestamp: msg.timestamp ?? Date.now(),
      payload: {
        s:     parsed.s,
        delta: parsed.delta ?? 0.5,
        cps:   parsed.cps   ?? 0.5,
      }
    });
  }

  _parseArgs(args) {
    const obj = {};
    for (let i = 0; i < args.length - 1; i += 2) {
      obj[args[i]] = args[i + 1];
    }
    return obj;
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }
}

module.exports = StrudelAdapter;
```
### BridgeServer.js
```.js
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200
//     node bridgeServer.js --device microbit2
//     node bridgeServer.js --device microbitbinary
//     node bridgeServer.js --device strudel

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//        {type:"strudel", timestamp:ms, payload:{s, delta, cps}}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitASCIIAdapter2 = require("./adapters/MicrobitASCIIAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");

const log = {
  info:  (...args) => console.log(`[${new Date().toISOString()}] [INFO]`,  ...args),
  warn:  (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`,  ...args),
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

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);
  } catch (e) {
    log.warn("Failed to parse JSON:", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE      = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT     = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD        = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ      = parseInt(getArg("hz", "30"), 10);
const VERBOSE     = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit2 found at ${path}`);
    return new MicrobitASCIIAdapter2({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bitbinary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "strudel") {
    log.info("Iniciando StrudelAdapter en puerto 8082...");
    return new StrudelAdapter({ port: 8080 });
  }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

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
    if (d.type === "strudel") {
      broadcast(wss, {
        type:      "strudel",
        timestamp: d.timestamp,
        payload:   d.payload,
        t:         nowMs(),
      });
      return;
    }

    broadcast(wss, {
      type: "microbit",
      x:    d.x,
      y:    d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t:    nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Client connected from ${req.socket.remoteAddress}. Total: ${wss.clients.size}`);

    const state  = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }

        try {
          await adapter.connect();
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
          log.info(`[HW-POLICY] Adapter kept open. Shared with ${wss.clients.size - 1} other client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }

        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ${detail}`);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Client disconnected. Total left: ${wss.clients.size}`);
      if (wss.clients.size === 0 && DEVICE !== "strudel") {
        log.info("[HW-POLICY] No more clients. Auto-disconnecting adapter...");
        adapter.disconnect();
      }
    });
  });

  // Arranque automático para sim y strudel
  if (DEVICE === "sim" || DEVICE === "strudel") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
## Bitácora de reflexión
