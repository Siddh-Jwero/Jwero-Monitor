# Observability & Crash-Handling Integration

Note: if you are using docker container's locally don't use localhost use your private ip address as (docker doesn't have access to the whole machine) 



This document describes how to integrate the provided `index.js` observability/logging snippet into any Node.js project (local or production). It explains the files you already provided, required environment variables, how to run locally with Prometheus, how to configure production IPs, how to test crash handling, and troubleshooting tips.

> **Files you provided**
>
> - `index.js` (your Node app with the integrated snippet)
> - `prometheus-config.yml` (Prometheus scrape config)
> - `docker-compose.yml` (runs Prometheus using the provided config)

---

## Overview

This integration gives you:

- Structured JSON logs sent to **Loki** via `winston-loki` (static labels `Server="logs"` and `env`).
- Console-friendly formatted logs for local development.
- Prometheus metrics via `prom-client` served on `/metrics`.
- Push to **Pushgateway** on fatal events (uncaught exceptions, unhandled rejections, graceful shutdown) so Prometheus can record crashes after the process exits.
- Safe flush / wait logic (`flushAndExit`) that gives the Loki transport time to POST logs before `process.exit()` so your full stack traces arrive in Loki.

This README explains how to run the code locally and how to adapt it for production IPs.

---

## Prerequisites

- Node.js (tested on Node 24.x) and `npm` or `pnpm`.
- Docker (for Pushgateway and Prometheus local testing).
- Running Loki instance reachable at `LOKI_URL` (not strictly required to start testing, but needed to see logs in Grafana).
- Prometheus and Grafana for metric and log visualization (Prometheus is included in `docker-compose.yml`).

---

## Dependencies (what your project needs)

Install these packages in the project that contains the `index.js` snippet:

```bash
npm install express response-time prom-client winston winston-loki
```

(If you already have `express` and other packages installed, skip duplicates.)





Install Grafana in a Docker-container on other server/localhost (Recommended)

```
docker run -d -p 3000:3000 --name=grafana grafana/grafana-oss
```

(Grafana should only run on port 3000 due to its limitations)





Install Push-gateway in Docker-container on other server/localhost (Recommended)

```
docker run -d --name pushgateway -p 9091:9091 prom/pushgateway
```





Install Grafana-Loki in Docker-container on other server/localhost (Recommended)

```
docker run -d --name=loki -p 3100:3100 grafana/loki
```

(Grafana and Grafana-Loki is Different {Grafana is a Dashboard and loki is a logger})





Setup for Prometheus Client should be given Below ....

---

## Environment variables used (defaults shown)

- `SERVICE_NAME` — service/job name used when pushing to Pushgateway. Default: `jweroai-service`.
- `HOSTNAME` — instance name. Default: resolved via `os.hostname()`.
- `PUSHGATEWAY_URL` — Pushgateway address. Default: `http://192.168.1.99:9091` (your local setup). Change to production Pushgateway URL in production.
- `LOKI_URL` — Loki server URL used by `winston-loki`. Default used in code: `http://192.168.1.99:3100`. Change for production.
- `NODE_ENV` — environment label for logs (`dev`/`prod`). Default: `dev`.
- `PORT` (if your full app listens on a port) — ensure Prometheus `prometheus-config.yml` has the correct target port.

**Note:** The snippet uses `instance = process.env.HOSTNAME || os.hostname()` to label pushes; for Kubernetes or VMs set `HOSTNAME` or use Pod metadata.

---

## Files & configs (what to change for production)

### `index.js` (snippet)
```js
const client = require("prom-client"); // Metrics client//added for dynamic llms

const winston = require("winston");
const LokiTransport = require("winston-loki");
const { createLogger, transports, format } = winston;
const responseTime = require('response-time');
const SERVICE_NAME = process.env.SERVICE_NAME || "jweroai-service";
const INSTANCE = process.env.HOSTNAME || require("os").hostname();


const appExpress = express();


// ---------------------------------------------------------------------------
// :brain: PROMETHEUS METRICS + PUSHGATEWAY
// ---------------------------------------------------------------------------

const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ register: client.register });

const reqResTime = new client.Histogram({
  name: "http_express_req_res_time",
  help: "Duration of HTTP requests in milliseconds",
  labelNames: ["method", "route", "status_code"],
});

const totalRequests = new client.Counter({
  name: "http_total_requests",
  help: "Total number of HTTP requests",
});

// Use response-time middleware with proper callback
appExpress.use(responseTime(function (req, res, time) {
  //if (req.path) 
    {  // Only record metrics for routes that exist
    //totalRequests.inc();
    reqResTime.labels({
      method: req.method,
      route: req.path,  // Use route.path for better grouping
      status_code: res.statusCode,
    }).observe(time);
  }
}));


appExpress.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});

const PUSHGATEWAY_URL = process.env.PUSHGATEWAY_URL || "http://192.168.1.99:9091";
const pushgateway = new client.Pushgateway(PUSHGATEWAY_URL);

const crashCounter = new client.Counter({
  name: "app_crash_total",
  help: "Total app crashes or fatal errors",
  labelNames: ["reason", "service", "instance"],
});

const lastExitTs = new client.Gauge({
  name: "app_last_exit_timestamp_seconds",
  help: "Last exit timestamp in seconds",
  labelNames: ["service", "instance", "reason"],
});

async function pushMetrics(reason = "unknown") {
  try {
    crashCounter.inc({ reason, service: SERVICE_NAME, instance: INSTANCE });
    lastExitTs.set(
      { service: SERVICE_NAME, instance: INSTANCE, reason },
      Date.now() / 1000
    );
    await pushgateway.pushAdd({
      jobName: SERVICE_NAME,
      groupings: { instance: INSTANCE },
      registry: client.register,
      critical: true,
    });
    logger.info(":white_check_mark: Pushed metrics to Pushgateway", { meta: { reason } });
  } catch (err) {
    console.error(":x: Failed to push metrics to Pushgateway:", err);
  }
}
//setTimeout(() => { throw new Error('Test crash'); }, 2000);
//testing crash

// ---------------------------------------------------------------------------
// :scroll: LOKI + WINSTON LOGGER SETUP
// ---------------------------------------------------------------------------

// 1) create the loki transport with static labels only
const lokiTransport = new LokiTransport({
  host: "http://192.168.1.99:3100",
  labels: {
    Server: "logs",
    env: process.env.NODE_ENV || "dev"
  },
  json: true
});

// 2) transform: move dynamic fields into meta and delete top-level keys
const normalizeForLoki = format((info) => {
  // keep message and timestamp, move problematic fields into meta
  info.meta = info.meta || {};

  // fields we want to *not* be top-level (so they don't become stream labels)
  const fieldsToMove = ['key', 'service_name', 'app', 'detected_level', 'level', 'userId', 'route'];

  for (const f of fieldsToMove) {
    if (Object.prototype.hasOwnProperty.call(info, f)) {
      // don't overwrite existing meta keys
      if (!Object.prototype.hasOwnProperty.call(info.meta, f)) {
        info.meta[f] = info[f];
      }
      delete info[f];
    }
  }

  // if you want level kept for console formatting but not promoted to Loki stream:
  // keep `info.level` (above we deleted it) — Console will still show level if you include it via format.

  return info;
});

// 3) build logger with normalize transform BEFORE json formatting
const logger = createLogger({
  level: 'debug',
  format: format.combine(
    normalizeForLoki(),
    format.timestamp(),
    format.json()
  ),
  transports: [
    new transports.Console(), // nice readable console output
    lokiTransport
  ],
});

// -------------------- FLUSH & EXIT HELPER --------------------
// Place this immediately after the `logger` creation above.
async function flushAndExit(exitCode = 1) {
  try {
    // Attempt to flush transports that expose a flush() method
    if (logger && Array.isArray(logger.transports)) {
      for (const transport of logger.transports) {
        if (typeof transport.flush === 'function') {
          // transport.flush accepts a callback in some implementations
          await new Promise((resolve) => transport.flush(resolve));
        }
      }
    }
  } catch (err) {
    console.error('Error while flushing transports:', err && err.stack || err);
  }

  // Give Node a short moment for any last network packets to be sent.
  // Increase this if your Loki/Pushgateway is slow or on a remote network.
  await new Promise((r) => setTimeout(r, 500));

  process.exit(exitCode);
}


// 4) use logger normally; do NOT pass top-level 'labels' or keys that you want as stream labels
// Instead put contextual data under meta so it travels in the JSON body:

function logErrorObject(err, message = "Error occurred", extraMeta = {}) {
  if (!err) {
    logger.error(message, { meta: extraMeta });
    return;
  }
  const meta = Object.assign({}, extraMeta, {
    errorMessage: err.message,
    errorName: err.name,
    stack: err.stack,
  });
  logger.error(message, { meta });
}

// ---------------------------------------------------------------------------
// :warning: GLOBAL ERROR + SHUTDOWN HANDLERS
// ---------------------------------------------------------------------------

// -------------------- FATAL HANDLERS (use flushAndExit) --------------------
process.on('uncaughtException', async (err) => {
  try {
    logErrorObject(err instanceof Error ? err : new Error(String(err)), 'Uncaught Exception', { critical: true });
    // Push crash metrics
    try {
      await pushMetrics('uncaughtException');
      logger.info('Pushed crash metrics (uncaughtException)', { meta: { reason: 'uncaughtException' }});
    } catch (pushErr) {
      console.error('Pushgateway push failed during uncaughtException:', pushErr && pushErr.stack || pushErr);
    }
  } catch (e) {
    console.error('Error while handling uncaughtException:', e && e.stack || e);
  } finally {
    // ensure logs/metrics flush before exit
    await flushAndExit(1);
  }
});

process.on('unhandledRejection', async (reason) => {
  try {
    const err = reason instanceof Error ? reason : new Error(String(reason));
    logErrorObject(err, 'Unhandled Rejection', { critical: true });
    try {
      await pushMetrics('unhandledRejection');
      logger.info('Pushed crash metrics (unhandledRejection)', { meta: { reason: 'unhandledRejection' }});
    } catch (pushErr) {
      console.error('Pushgateway push failed during unhandledRejection:', pushErr && pushErr.stack || pushErr);
    }
  } catch (e) {
    console.error('Error while handling unhandledRejection:', e && e.stack || e);
  } finally {
    await flushAndExit(1);
  }
});


async function gracefulShutdown(signal) {
  try {
    logger.info(`Received ${signal}, shutting down gracefully`, {
      meta: { service_name: SERVICE_NAME },
    });
    await pushMetrics(signal);
    logger.info("Metrics pushed before shutdown", {
      meta: { signal, service_name: SERVICE_NAME },
    });
  } catch (e) {
    logErrorObject(e, "Error during shutdown", { signal });
  } finally {
    // Flush logs and exit cleanly (0 for success)
    await flushAndExit(0);
  }
}


process.on("SIGINT", () => gracefulShutdown("SIGINT"));
process.on("SIGTERM", () => gracefulShutdown("SIGTERM"));
```


No extra files are required. To adapt for production:

- Set `PUSHGATEWAY_URL` to your production Pushgateway (e.g. `http://{ Public/Private Ip }:9091`).
- Set `LOKI_URL` to your Loki cluster endpoint (e.g. `http://{ Public/Private Ip }:3100`).
- Ensure your process listens on the port that Prometheus will scrape (update `PORT` or `appExpress.listen` accordingly).
- Make sure the `logger` creation runs early in your app so `exceptionHandlers` and `rejectionHandlers` are registered before other modules run.

### `prometheus-config.yml`

You provided:

```yaml
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: node-metrics-server
    static_configs:
      - targets: ["Public/Private Ip:3001"]
  - job_name: pushgateway
    static_configs:
      - targets: ["Public/Private Ip:9091"]
```

Change `Public/Private Ip::3001` to the hostname\:port where your Node app exposes `/metrics` in that environment. For production replace with the production host or use service discovery.

### `docker-compose.yml:`
```yml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus-config.yml:/etc/prometheus/prometheus.yml
```

You provided a compose that mounts `prometheus-config.yml` and runs Prometheus. To run locally:

```bash
docker-compose up -d
```

**Note:** Prometheus config uses static targets. In production you will typically use service discovery (Kubernetes, Consul) or update the static target to your production host/IP.

---

## Running locally (quickstart)

1. Start Pushgateway (local test run):

```bash
docker run -d --name pushgateway -p 9091:9091 prom/pushgateway
```

2. Start Prometheus (uses your `docker-compose.yml`):

```bash
docker-compose up -d
```

3. Ensure Loki is reachable at the `LOKI_URL` you set in the environment (if you want to view logs in Grafana/Loki). If you don’t have Loki locally, logs will still go to Console and Pushgateway metrics will still work.

4. Install node deps and start the app in the same project directory:

```bash
npm install
node index.js
```

5. Verify:

- Metrics: `curl http:// Public/Private Ip:3001/metrics` (replace port with your app port)
- Pushgateway UI: `http://Public/Private Ip :9091/` — after crash events you should see `job="jweroai-service"` entries
- Loki (Grafana Explore using Loki datasource): query `{Server="logs"}` or `{Server="logs"} |= "Uncaught Exception"`

---

## How to test crash handling (safe)

**Test trigger (temporary):** In `index.js` you can add a single-line test (already present during your tests):

```js
setTimeout(() => { throw new Error('Test crash'); }, 2000);
```

Start the app, and within a few seconds you should see:

- Console: an `Uncaught Exception` log with full stack trace.
- Loki: the JSON log entry (query `{Server="logs"} |= "Uncaught Exception" | json`).
- Pushgateway: `app_crash_total{reason="uncaughtException", service="jweroai-service", instance="..."}`

**Real-life testing:** Remove the test trigger and simulate a failure by throwing inside a route handler or by rejecting an unhandled Promise — the same handlers will catch and report it.

---

## Grafana / Loki queries (use in Explore)

- Show all app logs (stream):

```
{Server="logs"}
```

- Find uncaught exceptions and parse JSON:

```
{Server="logs"} |= "Uncaught Exception" | json
```

- Custom formatted line (timestamp + message + meta):

```
{Server="logs"} |= "Uncaught Exception" | json | line_format "{{.timestamp}} {{.message}} {{.meta | toJson}}"
```

---

## Prometheus queries and alert ideas

- Crashes in last 5 minutes:

```
increase(app_crash_total{service="jweroai-service"}[5m]) > 0
```

- Example alert rule (Prometheus):

```yaml
- alert: JweroaiServiceCrashed
  expr: increase(app_crash_total{service="jweroai-service"}[5m]) > 0
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "jweroai-service crashed within last 5m on {{ $labels.instance }}"
```

- Monitor blackbox/health (optional): set up `blackbox_exporter` to probe `/metrics` or a `/healthz` endpoint.

---

## Troubleshooting

- **I see metrics in Pushgateway but no logs in Loki:**
  - Make sure Grafana Explore uses **Loki** datasource when querying logs.
  - Confirm `LOKI_URL` is reachable from your app and `winston-loki` transport has correct host.
  - Increase the `flushAndExit` timeout if your network is slow (increase `setTimeout` to 1500–3000 ms).
  - Add temporary debug wrapper after creating `lokiTransport` to print `info` objects being sent:

```js
const origLog = lokiTransport.log && lokiTransport.log.bind(lokiTransport);
if (origLog) {
  lokiTransport.log = function(info, cb) {
    try { console.log('WINSTON->LOKI INFO:', JSON.stringify(info, null, 2)); } catch(e) { console.log('WINSTON->LOKI INFO (u):', info); }
    return origLog(info, cb);
  }
}
```

- **Push to Pushgateway fails:** check `PUSHGATEWAY_URL`, network/firewall, and ensure Pushgateway is running.

- **Labels fragmenting logs into many streams:** ensure contextual fields are sent under `meta` and not as top-level keys (the `normalizeForLoki` transform in your snippet already handles common keys).

---

## Production checklist (quick)

1. Set `PUSHGATEWAY_URL` and `LOKI_URL` to internal production endpoints. Do not expose Pushgateway/Loki publicly.
2. Update `prometheus-config.yml` targets to point to production app host/port.
3. Run Pushgateway & Prometheus behind your internal network or Kubernetes. Use service discovery in Prometheus where possible.
4. Consider RBAC/auth for access to logs/metrics in production Grafana/Prometheus.
5. Remove any test `setTimeout` crash triggers before deploying.

---

## Still any Questions?

@Siddh - {[https://tanikatech.slack.com/archives/D064B4DAG21](https://tanikatech.slack.com/archives/D064B4DAG21)}
