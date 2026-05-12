# Software Architectures Yomu B17

---

## System Context Diagram

```mermaid
graph TB
    User([👤 User]) --> Vercel["🌐 Yomu Frontend<br/>Vercel<br/>https://yomu-frontend-alpha.vercel.app"]
    Vercel --> |"/proxy/:path*"| EC2["⚙️ Yomu Backend<br/>AWS EC2<br/>http://52.6.39.43:8080"]
    EC2 --> RDS["🗄️ Amazon RDS<br/>PostgreSQL<br/>Port 5432"]
    Google["🔐 Google OAuth"] <--> EC2
    Vercel -.-> |"OAuth 2.0"| Google
```

| Component | Description |
|-----------|-------------|
| **User** | End user accessing Yomu via web browser |
| **Yomu Frontend** | Next.js app deployed on Vercel, handles UI and proxies API calls |
| **Yomu Backend** | Spring Boot REST API deployed on AWS EC2 (Docker container), handles auth, reads, quizzes, achievements |
| **Amazon RDS** | PostgreSQL database, endpoint stored in GitHub Secrets |
| **Google OAuth** | External identity provider for SSO login |

---

## Container Diagram

```mermaid
graph LR
    subgraph Frontend["🖥️ Frontend (Next.js)"]
        FE1["📄 Pages<br/>login, register, texts, quizzes, admin"]
        FE2["🔌 API Routes<br/>/api/auth/google/*"]
        FE3["📦 lib/axios<br/>HTTP client"]
        FE4["🔄 Next.js Rewrites<br/>/proxy/:path* → :path*"]
    end

    subgraph Backend["⚙️ Backend (Spring Boot)"]
        BE1["🔐 Auth Module<br/>AuthController, GoogleOAuthController, JWT"]
        BE2["📖 Read/Quiz Module<br/>TextController, QuizController, AdminQuizController"]
        BE3["🏆 Achievements Module<br/>AchievementController, DailyMissionController"]
        BE4["📡 Event Module<br/>EventPublisher, QuizCompletedEvent"]
    end

    subgraph Storage["🗄️ Amazon RDS (PostgreSQL)"]
        DB1["👤 users"]
        DB2["📖 texts, questions, quizzes, quiz_attempts"]
        DB3["🏆 achievements, daily_missions, user_achievements"]
    end

    FE1 & FE2 & FE3 & FE4 --> BE1 & BE2 & BE3
    BE1 & BE2 & BE3 --> Storage
```

| Module | Responsibilities | Key Classes |
|--------|------------------|-------------|
| **Auth Module** | Login, register, JWT generation, Google OAuth flow | `AuthController`, `GoogleOAuthController`, `JwtUtils`, `JwtAuthFilter`, `UserRepository`, `User` entity |
| **Read/Quiz Module** | Reading texts, quiz attempts, grading, admin question management | `TextController`, `QuizController`, `AdminQuizController`, `QuizServiceImpl`, `Text`, `Quiz`, `Question` models |
| **Achievements Module** | Achievement tracking, daily missions, user progress | `AchievementController`, `DailyMissionController`, `AchievementServiceImpl`, `UserAchievement`, `DailyMission` entities |
| **Event Module** | Publishes `QuizCompletedEvent` when quiz is graded (Spring `ApplicationEventPublisher`, no external broker) | `EventPublisher`, `QuizCompletedEvent` |
| **API Routes (FE)** | Google OAuth callback handler | `/api/auth/google/route.ts`, `/api/auth/google/callback/route.ts` |
| **Next.js Rewrites** | Proxies `/proxy/:path*` → `http://52.6.39.43:8080/:path*` to hide backend URL | `next.config.ts` rewrites |
| **HTTP Client (FE)** | Axios instance configured with base URL `/proxy` | `lib/axios` |

---

## Deployment Diagram

```mermaid
graph TD
    subgraph VercelCloud["🌐 Vercel"]
        FE["🎯 Next.js Frontend<br/>yomu-frontend-alpha.vercel.app"]
    end

    subgraph AWS["☁️ AWS"]
        subgraph EC2["🖥️ AWS EC2 (Ubuntu)"]
            Docker["🐳 Docker Container<br/>yomu-backend:latest<br/>Port 8080"]
        end
        subgraph RDS["🗄️ Amazon RDS"]
            Postgres["🍐 PostgreSQL<br/>yomu_db<br/>Port 5432"]
        end
    end

    subgraph External["🔐 External Services"]
        Google_API["👑 Google OAuth API"]
    end

    FE --> |"Rewrite: /proxy/:path*"| Docker
    Docker --> |"JDBC"| Postgres
    Docker --> |"OAuth 2.0"| Google_API
    FE -.-> |"OAuth 2.0"| Google_API
```

| Environment | Component | URL | Notes |
|-------------|----------|-----|-------|
| **Production** | Next.js Frontend (Vercel) | `https://yomu-frontend-alpha.vercel.app` | Auto-deploys on push to main |
| **Production** | Spring Boot Backend (EC2) | `http://52.6.39.43:8080` | Docker container, direct public IP |
| **Production** | Amazon RDS | RDS endpoint via GitHub Secrets | PostgreSQL 5432, credentials in GitHub Secrets |
| **External** | Google OAuth API | `accounts.google.com` | SSO login, credentials in GitHub Secrets |

---

## Future Software Architecture

### Future Container Diagram

```mermaid
graph LR
    subgraph Frontend["🖥️ Frontend (Next.js) - Vercel"]
        FE1["📄 Pages<br/>login, register, texts, quizzes, admin"]
        FE2["🔌 API Routes<br/>/api/auth/google/*"]
        FE3["📦 lib/axios<br/>HTTP client"]
        FE4["🔄 Next.js Rewrites<br/>/proxy/:path* → :path*"]
    end

    subgraph Gateway["🚪 yomu-gateway (Railway) - Spring Cloud Gateway"]
        GW1["🔐 JWT Auth Filter<br/>Validate token, inject X-User-Id headers"]
        GW2["🛣️ Route Config<br/>/api/auth → core-api<br/>/api/** → core-api<br/>/admin/** → core-api"]
        GW3["🌐 CORS Config<br/>Allow Vercel origin only"]
    end

    subgraph CoreAPI["⚙️ yomu-core-api (Railway) - Spring Boot"]
        CBE1["🔐 Auth Module<br/>Login, register, Google OAuth"]
        CBE2["📖 Read/Quiz Module<br/>TextController, QuizController, AdminQuizController"]
        CBE3["🏆 Achievements Module<br/>Admin CRUD achievements, missions"]
        CBE4["📡 Event Publisher<br/>publishQuizCompleted()<br/>publishMissionProgress()"]
    end

    subgraph GamEngine["🎮 yomu-gamification-engine (Railway) - Rust"]
        GE1["🏆 Achievement Handler<br/>handle_achievement_unlocked()<br/>XP & milestone tracking"]
        GE2["📊 Leaderboard Handler<br/>handle_quiz_completed()<br/>Clan score calculation"]
        GE3["⚡ Buff/Debuff Manager<br/>Auto-activate based on conditions<br/>Tier-strategy scoring"]
        GE4["📅 Season Manager<br/>handle_season_ended()<br/>Promotion/demotion logic"]
        GE5["🗓️ Mission Tracker<br/>handle_mission_progress()<br/>Daily mission state"]
    end

    subgraph Broker["🐇 RabbitMQ - CloudAMQP"]
        EX["📬 Exchange: yomu.events<br/>(topic)"]
    end

    subgraph Storage["🗄️ PostgreSQL - Railway Managed"]
        DB1["📦 auth schema<br/>users, sessions"]
        DB2["📦 quiz schema<br/>readings, questions,<br/>quiz_attempts, completed_readings"]
        DB3["📦 gamification schema<br/>clans, achievements,<br/>daily_missions, buffs, seasons"]
    end

    FE1 & FE2 & FE3 & FE4 --> Gateway
    Gateway --> CoreAPI
    CoreAPI --> |"quiz.completed<br/>mission.progress"| Broker
    Broker --> GamEngine
    GamEngine --> |"achievement.unlocked<br/>mission.progress"| Broker
    CoreAPI --> Storage
    GamEngine --> Storage
```

### Event Flow Summary

```mermaid
graph TD
    Q[Student completes quiz] --> |"quiz.completed event"| MQ[RabbitMQ<br/>yomu.events]
    MQ --> |"consume"| GE[🎮 yomu-gamification-engine]
    GE --> |"update XP,<br/>check achievements"| DB_g["🗄️ gamification schema"]
    GE --> |"achievement.unlocked<br/>(optional)"| MQ

    M[Student claims<br/>daily mission] --> |"mission.progress<br/>(with xpReward)"| MQ2[RabbitMQ<br/>yomu.events]
    MQ2 --> GE2[🎮 yomu-gamification-engine]
    GE2 --> |"UPDATE clan<br/>total_score"| DB_clan["🗄️ clans table"]
    GE2 --> |"check daily mission<br/>completion"| DB_mission["🗄️ daily_missions"]

    S[Admin ends season] --> |"season.ended event"| MQ3[RabbitMQ<br/>yomu.events]
    MQ3 --> GE3[🎮 yomu-gamification-engine]
    GE3 --> |"promote/demote<br/>clans by ranking"| DB_tier["🗄️ clans.tier"]
```

### RabbitMQ Events Contract

| Event | Routing Key | Publisher | Consumer | Action |
|-------|-------------|-----------|----------|---------|
| `quiz.completed` | `quiz.completed` | yomu-core-api | yomu-gamification-engine | Update XP, check achievements, update clan score |
| `mission.progress` | `mission.progress` | yomu-core-api | yomu-gamification-engine | Update daily mission state, add XP to clan if member |
| `achievement.unlocked` | `achievement.unlocked` | yomu-gamification-engine | yomu-core-api (optional) | Store notification |
| `season.ended` | `season.ended` | yomu-core-api (admin) | yomu-gamification-engine | Promote/demote clans per ranking |

### Planned Repositories

| Repo | Tech Stack | Hosting | Responsibility |
|------|-----------|---------|----------------|
| `yomu-infra` | Docker Compose + SQL scripts | Local dev only | RabbitMQ bootstrap, DB schema init scripts |
| `yomu-gateway` | Java 17, Spring Cloud Gateway | Railway | JWT auth, CORS, routing to core-api |
| `yomu-core-api` | Java 21, Spring Boot | Railway | Auth, readings, quizzes, achievements admin, event publishing |
| `yomu-gamification-engine` | Rust, SQLx | Railway | Achievement unlocks, leaderboard, clan scores, buffs/debuffs, missions, seasons |
| `yomu-web` | Next.js | Vercel | Existing frontend (no change) |

### Environment Variables (Future)

| Variable | Service | Notes |
|----------|---------|-------|
| `JWT_SECRET` | yomu-gateway | Min 32 chars, HS256 key |
| `CLOUDAMQP_URL` | yomu-gateway, yomu-core-api, yomu-gamification-engine | RabbitMQ connection string |
| `YOMU_WEB_URL` | yomu-gateway | Frontend URL for CORS |
| `CORE_API_URL` | yomu-gateway | Internal URL to yomu-core-api |
| `DATABASE_URL` | yomu-core-api, yomu-gamification-engine | PostgreSQL connection (Railway managed) |
| `GOOGLE_CLIENT_ID` | yomu-core-api | Google OAuth app client ID |
| `GOOGLE_CLIENT_SECRET` | yomu-core-api | Google OAuth app secret |
| `GOOGLE_REDIRECT_URI` | yomu-core-api | `https://yomu-gateway-url/api/auth/google/callback` |

### Module Responsibilities (Future)

| Module | Service | Responsibilities |
|--------|---------|-----------------|
| **Auth Module** | yomu-core-api | Login, register, Google OAuth, JWT generation |
| **Read/Quiz Module** | yomu-core-api | Reading CRUD, quiz submission, grading, publish `quiz.completed` event |
| **Achievements Module** | yomu-core-api | Admin CRUD achievements/missions, claim mission reward, publish `mission.progress` event |
| **Gamification Engine** | yomu-gamification-engine | All gamification logic: XP, leaderboard, clan scores, buffs/debuffs, achievement unlocks, season management |
| **Gateway** | yomu-gateway | JWT validation, forward user claims via `X-User-Id`, `X-User-Role` headers, CORS |

### Implementation Order

1. **yomu-infra** — RabbitMQ compose + SQL schemas + events contract
2. **yomu-gateway** — Spring Cloud Gateway with JWT filter and routing
3. **yomu-core-api** — Refactor existing backend into core-api service, add event publishing
4. **yomu-gamification-engine** — Rust service for gamification logic
5. **Vercel proxy update** — Change rewrite destination from EC2 IP to Railway gateway URL

### Architecture Modification Justification

This section explains why the current architecture is being changed and the reasoning behind key design decisions.

#### Why Split Into Microservices?

**Current State:** Single Spring Boot monolith (`yomu-backend`) handles all modules: auth, readings, quizzes, achievements, gamification logic, and event publishing.

**Problem (per `yomu.md` line 165-169):**
> "setiap modul harus berinteraksi dengan modul lain melalui mekanisme komunikasi yang terdefinisi dengan jelas, tanpa berbagi state atau pemanggilan langsung antar komponen"

The current monolith violates this — achievement and gamification logic are entangled in the same codebase, making independent iteration difficult and violating the separation of concerns requirement.

**Solution:** Split into two services:
- `yomu-core-api` — handles domain logic, publishes events
- `yomu-gamification-engine` (Rust) — handles all gamification logic (XP, achievements, leaderboards, buffs, seasons)

#### Why yomu-gateway?

**Problem:** Current backend is directly exposed to the internet with JWT handling in the same service. Adding CORS, rate limiting, or changing routing requires modifying the core service.

**Solution:** `yomu-gateway` (Spring Cloud Gateway) acts as single entry point:
- Strips `Authorization` header, injects `X-User-Id`, `X-User-Role`, `X-Username` headers
- Frontend never sees core-api URL directly (security through obscurity)
- Future: can add rate limiting, circuit breaking without touching core-api

#### Why Railway for All Backend Services?

**Current:** AWS EC2 (backend) + Amazon RDS (database)

| Factor | AWS EC2/RDS | Railway |
|--------|-------------|---------|
| Setup complexity | High — manual SSH, Docker, security groups | Low — GitHub repo connect, auto-deploy |
| Scaling | Manual or via ASG + ALB (complex) | Auto-scaling built-in |
| Cost (dev) | EC2 ~$10/mo + RDS ~$25/mo = $35/mo | ~$5/mo (Railway starter) |
| Database managed | Separate RDS setup | Included in same platform |
| Docker support | Yes, but manual | Native Docker container deploy |
| GitOps | Manual SSH script via GitHub Actions | Connected repo auto-deploys |

**Rationale:** Railway reduces ops overhead significantly. All backend services (gateway, core-api, gamification-engine) can be managed from one platform.

#### Why RabbitMQ Instead of Spring ApplicationEventPublisher?

**Current:** Spring `ApplicationEventPublisher` — events are in-process only, not persisted.

**Problem:** If `yomu-core-api` restarts during quiz completion, events in-flight are lost. No replay capability.

**Solution:** RabbitMQ (via CloudAMQP) provides:
- **Durability** — messages persisted to disk, survive broker restart
- **Replay** — gamification-engine can re-consume events if it restarts
- **Decoupling** — services don't need to know each other's network addresses, only the exchange name

#### Why Rust for Gamification Engine?

From `yomu.md` line 14-15: "mahasiswa diharapkan dapat menciptakan solusi digital yang mampu meningkatkan standar literasi informasi di Indonesia."

The gamification engine handles:
- Heavy leaderboard calculations (sorted clans per tier)
- Dynamic buff/debuff multipliers per strategy pattern
- High-frequency mission progress updates

**Rationale:** Rust's `sqlx` + async runtime handles these efficiently with minimal memory footprint. The architecture design document (`yomu-infra-gateway-design.md`) explicitly specifies Rust for this service.

#### Why 3 PostgreSQL Schemas?

**Current:** Single `public` schema with all tables mixed.

**New:** Three logical schemas: `auth`, `quiz`, `gamification`

**Rationale:** Aligns with microservice boundaries — each service owns its schema. `yomu-core-api` writes to `auth` and `quiz` schemas. `yomu-gamification-engine` writes to `gamification` schema. Clear ownership per `yomu.md` constraint.

---

### Risk Analysis

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **RabbitMQ downtime** — broker unavailable causes event loss or service failure | Medium | High | CloudAMQP free tier has uptime ~99.9%. Add retry with dead-letter queue if event publish fails. |
| **Gamification engine crash** — quiz events pile up in RabbitMQ queue | Medium | Medium | Monitor queue depth via RabbitMQ management UI. Implement consumer health checks. |
| **JWT secret mismatch** — gateway and core-api use different secrets, causing auth failures | Low | Critical | Both must share same `JWT_SECRET` env var. Document in shared secrets manager. |
| **Gateway routes to wrong core-api** — wrong `CORE_API_URL` env var | Low | Critical | Railway environment variable linking between services. Verify routing in staging before production. |
| **Data migration** — existing EC2 data must migrate to Railway PostgreSQL without downtime | Medium | High | Use read replica sync, then switchover. Test migration script in staging first. |
| **Google OAuth redirect URI mismatch** — URI still points to EC2 IP after migration | Medium | High | Update `GOOGLE_REDIRECT_URI` in Google Cloud Console to new gateway URL before cutting over. |
| **CORS blocking** — Vercel origin not in gateway allowed-origins list | Low | High | Set `YOMU_WEB_URL` env var to exact Vercel URL. Test in staging. |
| **Schema ownership conflict** — core-api and gamification-engine write to same schema | Low | High | Enforce schema isolation by DB role permissions: `core_api_user` can only write to `auth` and `quiz`. `gamification_user` can only write to `gamification`. |
| **Rust service cold start** — Railway hibernation causes delayed event processing | Medium | Low | Railway Hobby tier spins down after 15 min inactivity. Upgrade to usage-based plan or implement queue consumer that handles delayed start. |
| **Event schema drift** — core-api publishes event with extra field gamification-engine doesn't expect | Low | Medium | Maintain `EVENTS_CONTRACT.md` as source of truth. Add integration test to verify event shape. |

---

### Rollback Plan

If migration fails at any step:

1. **Before migration:** Tag current EC2 Docker image with `yomu-backend:v1`
2. **Step 1-2 failure:** Revert GitHub Actions deploy workflow to SSH into EC2 and run existing container
3. **Step 3-4 failure:** Keep EC2 running in parallel, point Vercel rewrite back to EC2 IP until Railway services are stable
4. **After full migration:** Terminate EC2 instance only after 1 week stability period