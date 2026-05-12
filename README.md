# Software Architectures Yomu B17

> Documentation Date: 2026-05-12
> Current State (reflects actual codebase and production deployment)

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