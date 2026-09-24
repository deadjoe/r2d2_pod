# r2d2_pod

A phone-sized launcher for [R2D2 // Listening Room](https://github.com/deadjoe/r2d2) on RunPod.
One tap creates the cheapest in-stock GPU pod that fits, runs the prebuilt image
`ghcr.io/deadjoe/r2d2`, waits until the weights are verified and the recogniser is loaded,
and hands back a link that opens the app. A time limit deletes the pod when you forget to.

It is a single Cloudflare Worker: static page + JSON API, a **Workflow** that runs the
deployment steps durably (it keeps going after you close the phone), a **Durable Object**
(SQLite) that holds session state, and a **cron** cost guard. Everything fits in the Workers
free plan; RunPod is the only thing that costs money.

```
phone ──► worker (/api/launch) ──► Workflow: select gpu → create pod → wait running → wait ready
                                        │                                   ▲
                                        │  R2D2_ACCESS_KEY, R2D2_PROGRESS_URL/TOKEN
                                        ▼                                   │
                                   RunPod pod (r2d2-start) ── POST /api/progress/<id> ─┘
                                        │
                        https://<pod>-8765.proxy.runpod.net/?k=<access key>  ◄── you
```

## Setup (once)

```sh
npm install
npx wrangler login
npx wrangler secret put RUNPOD_API_KEY     # a RunPod API key with pod create/delete rights
npx wrangler secret put LAUNCH_KEY         # a long random string; the page then needs /?k=<key> once
npx wrangler secret put NOTIFY_URL         # optional: an ntfy.sh topic URL, e.g. https://ntfy.sh/<random-topic>
npx wrangler deploy
```

**Set `LAUNCH_KEY`.** Without it anyone who finds the `workers.dev` hostname can start pods
on your RunPod account. With it, the page and the API answer 401 until you open
`https://<host>/?k=<key>` once on that device (a cookie keeps you in for a year; a home-screen
web app can paste the key into the lock page). The pod's progress route is exempt and is
protected by a per-session bearer token instead. Cloudflare Access in front of the hostname
works too; give `/api/progress/*` a Bypass policy.

## Use

Open the page, pick the time limit / memory / price ceiling, tap **Deploy**. The timeline
fills in from the pod itself (`gpu`, `weights`, `start`, `ready`). When the state turns
**ready**, tap *OPEN R2D2*: the link carries the session's access key, which the app trades
for a cookie. Each session gets a new key, so an old link stops working with its pod.
Closing the page changes nothing; the Workflow finishes on its own and, if `NOTIFY_URL` is
set, sends a push notification (with the pod's address, never the key).

**Stop & delete pod** ends the session immediately. Otherwise the pod is deleted when the
time limit is reached. Independently of both, the cron runs every 15 minutes and deletes any
`r2d2-pod-*` pod on the account whose session is over, expired, or unknown for more than
six hours.

## Configuration

`wrangler.jsonc` `vars` (defaults, all overridable per launch from the page):

| var | default | meaning |
|---|---|---|
| `IMAGE` | `ghcr.io/deadjoe/r2d2:latest` | image to run; must be public on GHCR |
| `MIN_GPU_GB` | `16` | smallest card considered (the app uses about 5 GB) |
| `MAX_PRICE_PER_HR` | `0.60` | price ceiling, USD/h |
| `CLOUD` | `SECURE` | `SECURE` or `COMMUNITY` |
| `DEFAULT_TTL_HOURS` / `MAX_TTL_HOURS` | `2` / `8` | time limit and its cap |
| `CONTAINER_DISK_GB` | `20` | container disk; weights (3.3 GB) live in `/data` inside it |

Pods are created with `allowedCudaVersions` 12.8+ (the image needs an R570+ driver). Cards
older than Turing (V100, P-series) and MIG slices are skipped: the image's llama-server is
built for sm_75 and newer.

## API

| route | |
|---|---|
| `GET /api/gpus?min_gb&max_price&cloud` | in-stock candidates, cheapest first |
| `POST /api/launch` `{ttl_hours, min_gpu_gb, max_price, cloud}` | start a session (409 if one is live) |
| `GET /api/sessions`, `GET /api/sessions/:id` | state, events, link, cost estimate |
| `POST /api/sessions/:id/stop` | delete the pod, end the session |
| `POST /api/progress/:id` (bearer token) | called by the pod's `r2d2-start` |

## Development

```sh
npx wrangler types && npx tsc --noEmit
npx wrangler dev          # needs RUNPOD_API_KEY in .dev.vars
```
