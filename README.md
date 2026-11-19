# UBoard

**Classroom games, a real-time whiteboard and live quizzes for teachers.**
Production product: **[uboard.pro](https://uboard.pro)** (Ukrainian / English). This repository is a case study: what the system does, how it is built, and which engineering decisions I made. The source code is private. I am happy to walk through it on a call.

## What it does

- **8 learning games from one list of pairs.** A teacher pastes pairs (word and translation, country and capital, term and definition) and gets Match, Quiz, Memory, Write, True/False, Letters, Hangman and Flashcards. Students open a link or scan a QR code, **no account needed**.
- **Results for the teacher.** Who played, how long it took, which answers were wrong, which pairs are hardest.
- **Online whiteboard.** Infinite canvas with pen, shapes, arrows, stickers, images and rich text. Real-time collaboration with remote cursors and presence, undo/redo, share links, guest access, friends and notifications, a template marketplace.
- **Live class game.** The teacher starts a game on the projector (6-digit PIN + QR code), students join from their phones, answer under a timer, and get points for speed. Leaderboard between questions, podium at the end.
- **Ready-made lessons.** A starter catalog of English sets (A1 to B1) and other subjects that can be played immediately.

## Architecture

```
Browser (React + Vite + TypeScript)
 ├─ REST      auth, boards, game sets, friends, notifications, admin
 ├─ Socket.IO whiteboard rooms (one room per board)
 └─ Socket.IO /live namespace (live class games, separate from the board code)
        │
Node.js + Express + TypeScript (API + WebSocket, one process)
        │
MySQL 8 (users, sessions, boards, elements, members, game sets, results, ...)
```

The client keeps an optimistic local state (Zustand stores). Every change goes to the server over Socket.IO. The server validates permissions and the payload, persists it to MySQL and broadcasts it to the room. Shared TypeScript types live in a common package used by both the client and the server.

## Engineering highlights

### Real-time whiteboard
- One Socket.IO room per board; events for join, add/update/delete element, cursor move and presence. Server-side permission checks on every message, with message validation and rate limiting.
- Automatic reconnect with a status banner, client-side image compression and a cache, path simplification for pen strokes, and a custom eraser for Bezier paths and lines.
- Performance work driven by profiling: dropped an expensive full-board checksum and moved board images from the database to files.

### Live class game (server-authoritative)
- The room is a **pure state machine** (lobby, question, reveal, leaderboard, ..., podium) with an **injected clock**, so the whole game can be tested without waiting for timers.
- The **server owns everything that matters**: questions, correct answers, timing and score. The player's phone never receives the correct answer before the reveal stage; late and duplicate answers are rejected; response time is measured on the server.
- Scoring: up to 1000 points for a correct answer, decreasing with time, plus a small streak bonus.
- **Reconnection:** a seat token lets a student return to the same seat with their points after a page reload or lost connection. A second device with the same token replaces the first.
- Limits and abuse protection: 40 players per room, one room per host, per-message rate limiting, nickname validation (no links, e-mails or phone numbers), the host can kick players. Only a verified, signed-in teacher can start a game.
- **Privacy by design:** nicknames, answers and scores of a live game exist only in memory for the duration of the game; live pages are excluded from search engines.

**Load test** (local, 5 questions, simulated players answering after 0.2 to 3 s):

| Scenario | Messages | p50 | p95 | p99 |
|---|---|---|---|---|
| 5 rooms x 40 players = 200 players, answer acknowledged | 1000 | 0.8 ms | 1.7 ms | 3.6 ms |
| 5 rooms x 40 players, question delivered to a phone | 1000 | 2.4 ms | 4.6 ms | 5.7 ms |
| 1 room x 40 players, join | 40 | 3.9 ms | 5.3 ms | 5.4 ms |

No errors or dropped connections; the 41st player correctly receives a "room full" response.

### Authentication and security
- Short-lived access tokens (JWT) plus refresh tokens stored as server-side sessions; e-mail verification codes; password reset; trusted devices and device fingerprinting.
- Role checks in middleware (user / admin), rate limiting on sensitive routes, user bans, an admin panel, and error and request logging.
- Dependency hygiene: high-severity audit findings closed by upgrading the image-processing library and raising the minimum Node version; lock files are generated with a pinned npm version so CI stays reproducible.

### CI/CD
GitHub Actions on every push and pull request, with three parallel jobs:
1. **client:** lint, tests, production build.
2. **server:** tests, production build.
3. **fresh-install:** starts a real **MySQL 8 service**, loads the schema into an empty database, builds the server on the same Node version as production, boots it in production mode and polls the health endpoint. This catches broken schema changes and startup failures before they reach users.

A **deploy job** runs only on `main` and only after all three jobs are green. It uses a concurrency lock (one production deploy at a time) and SSH deploy keys stored as repository secrets. The deploy script **waits for live games to finish** (up to 20 minutes) before restarting the server, so a release does not kill a lesson in progress; a force flag exists for emergencies.

### Operations and recovery
- Process management with PM2 behind a reverse proxy; MySQL with Docker Compose for local development.
- A **board recovery tool** that restores a whiteboard from the MySQL binlog.
- Server metrics collection, error logs, and written operations runbooks.

### Product and growth
- Two languages (Ukrainian and English) with a lightweight i18n layer.
- Public pages (landing, game catalog, policy and terms) generated by a build script for SEO; Umami analytics events for product decisions.

## Stack

| Layer | Technologies |
|---|---|
| Frontend | React, Vite, TypeScript, Zustand, Socket.IO client |
| Backend | Node.js, Express, TypeScript, Socket.IO, JWT, sharp |
| Database | MySQL 8 |
| Infra / delivery | GitHub Actions, PM2, Apache reverse proxy, Docker Compose, SSH deploy |
| Quality | Unit tests (client and server), ESLint, fresh-install smoke test in CI |
| Product | i18n (uk/en), Umami analytics, SEO page generation |

## What this shows

Taking a product from idea to production end to end: real-time systems, server-authoritative game logic, security basics, a deployment pipeline that protects users, and documentation that lets someone else pick the project up.

**Need something similar?** Real-time apps, dashboards, Telegram bots and industrial integrations: see my [profile](https://github.com/qomad13) or write to ledachkov.ilya@gmail.com.
