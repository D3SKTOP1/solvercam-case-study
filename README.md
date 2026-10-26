# SolverCam — Edge AI for physical retail (case study)

> I'm co-founder and **CTO** of SolverCam. The product code is proprietary; this repository explains **what we built, how it works and the engineering decisions behind it**. Website: [solvercam.com](https://solvercam.com)

## The problem

Stores already have cameras, but the footage is only watched after something goes wrong. Owners have no reliable numbers on how many people come in, when the peaks are, where customers stop, or whether staffing matches traffic. Off-the-shelf people counters need new hardware, and sending video to the cloud collides with privacy law (LGPD, Brazil's GDPR).

## The solution

A small NVIDIA Jetson device sits next to the store's existing DVR. It reads the cameras over the local network, detects and tracks people in real time, and turns video into **structured data only** — line crossings, zone events, occupancy. That data goes to a cloud backend and becomes dashboards, alerts and reports for the store owner.

**Images never leave the store.** The cloud only receives numbers and events.

## Architecture

```
Store cameras (RTSP / DVR)
        │  local network
        ▼
EDGE — NVIDIA Jetson Orin Nano (on-prem)
  DeepStream + TensorRT + PeopleNet  →  detection
  NvDCF tracker                      →  tracking
  stage-graph pipeline per camera    →  line crossing, zones, occupancy
  local outbox (SQLite)              →  survives network outages
        │  HTTPS, per-device token
        ▼
CLOUD BACKEND — FastAPI + PostgreSQL
  ingestion · analytics · alerts · multi-tenant auth
        │  REST + JWT
        ▼
WEB APP — Next.js            MOBILE — React Native / Expo
  dashboards, peak hours, daily flow, alerts (no pixels)

LOCAL CONFIG (on the device, over a private Tailscale network)
  the only place where a technician sees live video: draw counting lines and zones, audit counts
```

## Key engineering decisions

| Decision | Why |
|---|---|
| **Inference on the edge, not in the cloud** | Privacy by design (no images leave the store), no video bandwidth cost, works with the customer's existing cameras |
| **Store-and-forward outbox on the device** | Store internet is unreliable; events are queued locally and drained when the link is back — nothing is lost |
| **Count by boundary crossing, no cross-camera re-identification** | Re-ID across cameras is both a privacy risk and a source of error; counting at the door line is simpler, auditable and accurate enough to sell |
| **Pixel boundary enforced in the API** | Endpoints that could expose images return `410 LOCAL_PREVIEW_ONLY`; visual tools live only on the device, reachable over a private network |
| **Zero-touch device enrollment** | Devices claim a code and receive their own token, so installing a new store doesn't need an engineer on site |
| **Architecture Decision Records** | 30+ ADRs document every major choice (transport, auth, runtime, privacy, scaling to 50 stores) so the team can onboard and disagree in writing |
| **Measure before promising** | Every roadmap milestone is a question answered with measured accuracy in a real store, with the gate defined *before* writing code |

## Results in production

Measured live on two devices in a real store over 11 days:

- **28,485** line crossings and **81,612** zone events processed
- **7 cameras** analysed in real time on a single Jetson, with **~45% GPU** headroom left
- **100%** of edge events delivered to the cloud (outbox fully drained)
- **4+ days** of continuous service with zero restarts
- Image retention on the device automatically purged after 7 days

Next step: closing the accuracy loop with human audits at scale before moving up from descriptive to predictive analytics (staffing recommendations).

## Other pieces I built around the product

- **DVR Snapshot Agent** — Windows agent that pulls frames from Intelbras DVRs → [dvr-snapshot-agent](https://github.com/D3SKTOP1/dvr-snapshot-agent)
- **Mobile app** — React Native / Expo client for store owners
- **AI-assisted engineering workflow** — an operator queues tasks, coding agents pick them up and return results through GitHub, with review before merge

## My role

Co-founder and CTO: system architecture, edge pipeline and device fleet, backend and web app, infrastructure and deployment, privacy/compliance design, and leading a team of engineers.

**Stack:** Python · FastAPI · PostgreSQL · SQLite · NVIDIA DeepStream / TensorRT · Jetson · OpenCV · YOLO / RF-DETR (earlier pipeline) · Docker · Tailscale · Next.js · TypeScript · React Native (Expo) · GitHub Actions

---

**Vitor Portela** — [LinkedIn](https://www.linkedin.com/in/mvitorportela/)
