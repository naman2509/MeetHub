# MeetHub Video Call

A full-stack, Zoom/Google-Meet-style video conferencing app. Users register/log in, start or join a meeting with a shareable ID, and get live multi-party video, audio, screen sharing, and in-call chat — all peer-to-peer over WebRTC, with a small Node/Express + Socket.IO backend acting purely as the signaling server and a MongoDB store for accounts and meeting history.

Built as a learning/portfolio project (based on the "apnacollege" video-call tutorial lineage) and since restyled with Framer Motion + GSAP animations and a MUI dark theme.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [REST API](#rest-api)
- [Socket.IO Events](#socketio-events)
- [Design Decisions & Tradeoffs](#design-decisions--tradeoffs)
- [Known Limitations](#known-limitations)
- [Possible Improvements](#possible-improvements)

---

## Features

- **Auth** — register/login with hashed passwords (bcrypt), opaque token issued on login and stored in `localStorage`.
- **Instant or scheduled-style meetings** — generate a Google-Meet-style meeting ID (`abc-defg-hij`) or join an existing one by code.
- **Multi-party video calls** — full-mesh WebRTC, so every participant streams directly to every other participant (no media server).
- **Audio/video toggling** — mute mic / stop camera without renegotiating the peer connection (just disables the track).
- **Screen sharing** — swaps the outgoing video track on every active `RTCPeerConnection` via `replaceTrack`, and automatically reverts to the camera when the browser's "stop sharing" control is used.
- **In-call chat** — text messages broadcast to everyone in the room, backed by a per-room, in-memory message log so latecomers see history from that server's uptime.
- **Meeting history** — every meeting a logged-in user starts/joins is recorded and viewable on a History page.
- **Guest access** — `Join as Guest` on the landing page skips auth and drops straight into a fixed demo room.

## Tech Stack

**Frontend** (`/frontend`)
| Library | Purpose |
|---|---|
| React 18 + React Router 6 | SPA shell & routing |
| Socket.IO client | Signaling transport |
| Native WebRTC (`RTCPeerConnection`) | Media transport |
| MUI 5 (`@mui/material`) | Component library / dark theme |
| Framer Motion + GSAP | Page/element animation |
| Axios | REST calls to the backend |

**Backend** (`/backend`)
| Library | Purpose |
|---|---|
| Express 4 | HTTP API |
| Socket.IO | WebRTC signaling (offer/answer/ICE relay), chat relay, presence |
| Mongoose / MongoDB | User accounts & meeting history |
| bcrypt | Password hashing |
| Node `crypto` | Opaque auth token generation |

## Architecture

```
┌────────────┐        REST (/api/v1/users)         ┌──────────────┐
│  React SPA │ ───────────────────────────────────▶ │   Express    │
│ (frontend) │ ◀─────────────────────────────────── │   (backend)  │
└─────┬──────┘        auth / meeting history         └──────┬───────┘
      │                                                      │
      │  Socket.IO (join-call, signal, chat-message)         │ Mongoose
      │◀────────────────────────────────────────────────────▶│
      │                                                       ▼
      │                                                 ┌──────────┐
      │                                                 │ MongoDB  │
      │                                                 └──────────┘
      │
      │  WebRTC media (SDP/ICE negotiated via the signaling
      │  socket above, then flows directly peer-to-peer)
      ▼
┌────────────┐   direct P2P audio/video/screen   ┌────────────┐
│ Participant │◀──────────────────────────────────▶│ Participant│
│      A      │                                     │      B    │
└────────────┘                                     └────────────┘
```

- The backend **never touches media** — `socketManager.js` only relays `signal` events (SDP offers/answers and ICE candidates) between socket IDs and tracks room membership in an in-memory `connections` map keyed by room path.
- Each browser opens **one `RTCPeerConnection` per other participant** (full mesh). When a peer joins, it's told about every existing socket in the room and initiates an offer to each one.
- STUN-only ICE (`stun:stun.l.google.com:19302`) — there is no TURN relay, so calls between peers on restrictive/symmetric NATs can fail to connect (see [Tradeoffs](#design-decisions--tradeoffs)).
- Chat messages are appended to an in-memory `messages[roomPath]` array and replayed to a socket when it joins — this is why chat history exists only for the current server process uptime and only for participants currently online.

## Project Structure

```
Zoom/
├── backend/
│   ├── src/
│   │   ├── app.js                     # Express app bootstrap, Mongo connect, HTTP+Socket.IO server
│   │   ├── controllers/
│   │   │   ├── socketManager.js       # Signaling: join-call, signal, chat-message, disconnect
│   │   │   └── user.controller.js     # login, register, addToHistory, getUserHistory
│   │   ├── models/
│   │   │   ├── user.model.js          # { name, username, password (hashed), token }
│   │   │   └── meeting.model.js       # { user_id, meetingCode, date }
│   │   └── routes/
│   │       └── users.routes.js        # /api/v1/users/*
│   └── package.json
└── frontend/
    ├── src/
    │   ├── contexts/AuthContext.jsx   # axios client + auth/history API calls, token in localStorage
    │   ├── pages/
    │   │   ├── landing.jsx            # Marketing landing page
    │   │   ├── authentication.jsx     # Login / register form
    │   │   ├── home.jsx               # Start/join meeting, protected via withAuth
    │   │   ├── history.jsx            # Past meetings list
    │   │   └── VideoMeet.jsx          # Lobby + in-call room: getUserMedia, RTCPeerConnection mesh, chat, screen share
    │   ├── utils/
    │   │   ├── generateMeetingId.js   # Random "abc-defg-hij" style ID generator
    │   │   └── withAuth.jsx           # HOC redirecting to /auth if no token
    │   └── environment.js             # Backend base URL (see Environment Variables)
    └── package.json
```

## Getting Started

### Prerequisites

- Node.js 20.x (pinned via `frontend/package.json` `engines`)
- A MongoDB instance (local or [Atlas](https://www.mongodb.com/atlas))

### 1. Backend

```bash
cd backend
npm install
```

Create `backend/.env`:

```env
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>/<db>
PORT=8000
```

```bash
npm run dev     # nodemon, auto-restarts on change
# or
npm start       # plain node
```

The API + Socket.IO server starts on `http://localhost:8000`.

### 2. Frontend

```bash
cd frontend
npm install
npm start
```

Opens `http://localhost:3000`.

> **Note:** `frontend/src/environment.js` currently hardcodes the backend URL behind an `IS_PROD` boolean rather than reading a build-time env var:
> ```js
> let IS_PROD = true;
> const server = IS_PROD ? "meet-hub-steel.vercel.app" : "http://localhost:8000";
> ```
> To run the frontend against your local backend, flip `IS_PROD` to `false` (or replace this with a `process.env.REACT_APP_SERVER_URL` read — see [Tradeoffs](#design-decisions--tradeoffs)).

## Environment Variables

| Variable | Where | Description |
|---|---|---|
| `MONGO_URI` | `backend/.env` | MongoDB connection string |
| `PORT` | `backend/.env` | Backend HTTP/Socket.IO port (defaults to `8000`) |

The frontend has no `.env` — its backend URL is a hardcoded literal in `environment.js` (see note above).

## REST API

Base path: `/api/v1/users`

| Method | Endpoint | Body / Query | Description |
|---|---|---|---|
| `POST` | `/register` | `{ name, username, password }` | Creates a user; password is bcrypt-hashed before storage |
| `POST` | `/login` | `{ username, password }` | Verifies password, generates a new random hex token, saves it on the user document, returns it |
| `POST` | `/add_to_activity` | `{ token, meeting_code }` | Records a meeting in the caller's history |
| `GET` | `/get_all_activity` | `?token=` | Returns all meetings recorded for the token's user |

Auth is **not** JWT-based — `token` is a random 40-character hex string (`crypto.randomBytes(20).toString('hex')`) persisted on the user document and passed back on every history request as a query/body param, not an `Authorization` header.

## Socket.IO Events

| Event | Direction | Payload | Purpose |
|---|---|---|---|
| `join-call` | client → server | `roomPath` (full URL used as room key) | Joins/creates a room, triggers `user-joined` broadcast, replays stored chat history to the joiner |
| `user-joined` | server → clients | `(socketId, allSocketIdsInRoom)` | Tells every client in the room who's present so mesh connections can be created |
| `signal` | both directions | `(toSocketId, JSON.stringify({sdp} \| {ice}))` | Relays SDP offers/answers and ICE candidates between two specific peers |
| `chat-message` | both directions | `(data, sender)` in, `(data, sender, senderSocketId, timestamp)` out | Broadcasts a chat message to everyone in the sender's room |
| `user-left` | server → clients | `socketId` | Tells clients to tear down the `RTCPeerConnection` for a disconnected peer |
| `disconnect` | client → server (implicit) | — | Removes the socket from its room, notifies remaining peers |

## Design Decisions & Tradeoffs

This section documents deliberate (and inherited) tradeoffs so future work can weigh them consciously rather than "fixing" something that was a considered choice.

**Full-mesh WebRTC (no SFU/MCU media server)**
Every participant opens a direct `RTCPeerConnection` to every other participant. This is simple to build, needs zero media-server infrastructure/cost, and keeps media off the backend entirely (lower latency, more private). The cost is that it scales quadratically — bandwidth and CPU per client grow with `O(n)` peers, and total connections in a room are `O(n²)`. Good for small calls (roughly ≤4–6 participants); a real multi-party product would front the media with an SFU (e.g. mediasoup, LiveKit, Janus) at the cost of running and paying for that infrastructure.

**In-memory signaling state on a single Node process**
`connections`, `messages`, and `timeOnline` in `socketManager.js` are plain JS objects, not Redis or a DB. This is fast and needs no extra service, but it means: (a) all state is lost on server restart, and (b) the backend **cannot be horizontally scaled** past one instance without adding the Socket.IO Redis adapter, since two server processes wouldn't share room membership.

**STUN-only ICE, no TURN server**
Only `stun:stun.l.google.com:19302` is configured. STUN is free and enough for most home/office NATs, but calls between peers behind symmetric NATs or restrictive corporate firewalls will fail to establish a direct path with no TURN relay to fall back to. Adding a TURN server (e.g. coturn, Twilio, Metered) fixes this but adds ongoing relay bandwidth cost.

**Opaque random token instead of JWT**
Login issues a random hex string stored on the user document rather than a signed, expiring JWT. This is simple and trivially revocable (never expires unless the account row changes), but there's no built-in expiry, no scopes/claims, and every history request does a DB lookup by token rather than a stateless signature check. "Logout" is just deleting the token from `localStorage` client-side — the token itself stays valid server-side until the user logs in again (which overwrites it).

**No room access control**
Anyone who has (or guesses) a meeting ID can join — there's no per-meeting password, waiting room/lobby approval, or host role. Meeting IDs are reasonably high-entropy (`26³ × 26⁴ × 26³ ≈ 1.6×10¹⁷` combinations) which makes blind guessing impractical, but there is no expiry or revocation once a code has been shared.

**CORS wide open (`origin: "*"`)**
Both the Express `cors()` middleware and the Socket.IO CORS config accept any origin. Fine for a demo/personal project, but it means any website could make authenticated-looking requests to the API using a leaked token. Should be locked to the actual frontend origin(s) before any real deployment.

**Hardcoded backend URL in the frontend (`environment.js`)**
The frontend picks its API base URL via a hardcoded `IS_PROD` boolean and a literal Vercel URL, instead of a `REACT_APP_*` build-time environment variable. Switching environments currently requires a code edit + rebuild rather than a config change, and the hardcoded URL is also missing its `https://` scheme.

**Mute/camera-off toggles the MediaStreamTrack, not renegotiation**
`handleVideo`/`handleAudio` just set `track.enabled = false` rather than removing/re-adding tracks or renegotiating. This avoids expensive SDP renegotiation and connection hiccups, at the cost of still sending (silent/black) RTP packets — slightly more bandwidth than a "hard" mute, but a much smoother UX with no visible reconnect.

**Plain JavaScript, not TypeScript**
Faster to iterate on for a small solo/tutorial-derived project, but there's no compile-time protection against payload/shape mismatches between the REST controller responses, Socket.IO event payloads, and the components consuming them.

## Known Limitations

- No password-reset / forgot-password flow.
- No moderation controls (kick/mute-others, waiting room, host transfer).
- No server-side call recording or transcript.
- No automated test suite beyond the default Create React App smoke test (`App.test.js`).
- No reconnect/backoff handling if the Socket.IO connection drops mid-call — the peer connections are not re-established automatically.
- Guest access on the landing page ("Join as Guest") always routes to the same fixed demo room path.

## Possible Improvements

- Swap full-mesh for an SFU once typical room size grows past a handful of participants.
- Add a Socket.IO Redis adapter + move `connections`/`messages` state to Redis for horizontal scaling and restart-safety.
- Replace the opaque token with short-lived JWTs + refresh tokens, and move the token out of `localStorage` (e.g. httpOnly cookie) to reduce XSS token-theft risk.
- Add a TURN server for reliable connectivity across restrictive NATs.
- Lock CORS to known origins and move `environment.js` to a real `REACT_APP_*` env var.
- Add per-meeting passwords/host controls and a lobby/waiting-room approval step.
