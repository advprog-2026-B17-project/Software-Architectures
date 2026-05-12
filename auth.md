# Auth Module — Component & Code Architecture

---

## PART 1: Component Diagrams

### Component Overview

> High-level view of the auth module as interconnected subsystems.

```mermaid
graph TB
    subgraph Controllers["🖥️ Controllers"]
        AC["🔐 AuthController<br/>/auth/register<br/>/auth/login<br/>/auth/me GET<br/>/auth/me PUT"]
        GOC["🌐 GoogleOAuthController<br/>/auth/oauth/google<br/>/auth/oauth/google/callback"]
    end

    subgraph Security["🛡️ Security Layer"]
        JAF["🔑 JwtAuthFilter<br/>(OncePerRequestFilter)"]
        SC["⚙️ SecurityConfig<br/>(SecurityFilterChain)"]
        PE["🔒 PasswordEncoder<br/>(BCrypt)"]
    end

    subgraph Utils["⚡ Utils"]
        JU["🎯 JwtUtils<br/>(Component)"]
    end

    subgraph Repository["📦 Repository"]
        UR["📂 UserRepository<br/>(JpaRepository)"]
    end

    subgraph Entities["📋 Entity"]
        U["👤 User<br/>(JPA Entity)"]
    end

    subgraph DTOs["📤 DTOs"]
        DTO1["📝 LoginRequest"]
        DTO2["📝 RegisterRequest"]
        DTO3["📝 UpdateProfileRequest"]
        DTO4["📤 AuthResponse"]
        DTO5["📤 UserResponse"]
        DTO6["📤 GoogleAuthResponse"]
    end

    Controllers --> |"depends on"| Repository
    Controllers --> |"uses"| Utils
    Controllers --> |"encodes"| Security
    Security --> |"validates|encodes"| Utils
    Security --> |"queries"| Repository
    Repository --> |"persists|queries"| Entities
    Controllers --> |"returns"| DTOs
```

### Security Configuration — Endpoint Access Map

> Which endpoints are public vs authenticated.

```mermaid
graph LR
    A["🔓 Public Endpoints"] --> B["POST /auth/register"]
    A --> C["POST /auth/login"]
    A --> D["GET /auth/oauth/google"]
    A --> E["GET /auth/oauth/google/callback"]
    A --> F["GET /api/texts/**"]
    A --> G["GET /api/quizzes/**"]
    A --> H["POST /api/events/quiz-completed"]
    A --> I["POST /api/events/quiz_completed"]

    J["🔒 Authenticated Endpoints"] --> K["POST /api/quizzes/*/start"]
    J --> L["POST /api/quizzes/attempts/**"]
    J --> M["GET /api/quizzes/attempts/**"]
    J --> N["GET /auth/me"]
    J --> O["PUT /auth/me"]
    J --> P["* (any other)"]
```

---

## PART 2: Code Diagrams

### Class Diagram

> Structural relationships between all auth module classes and their members.

```mermaid
classDiagram
    class User {
        +Long id
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String googleId
        +String password
        +LocalDateTime createdAt
        +String role
        +List~Text~ texts
        +List~QuizAttempt~ quizAttempts
    }

    class AuthController {
        +UserRepository userRepository
        +PasswordEncoder passwordEncoder
        +JwtUtils jwtUtils
        +register(RegisterRequest) ResponseEntity
        +login(LoginRequest) ResponseEntity
        +getCurrentUser() ResponseEntity
        +updateProfile(UpdateProfileRequest) ResponseEntity
    }

    class GoogleOAuthController {
        +String googleClientId
        +String googleClientSecret
        +String redirectUri
        +UserRepository userRepository
        +JwtUtils jwtUtils
        +RestTemplate restTemplate
        +initiateGoogleOAuth() ResponseEntity
        +handleGoogleCallback(code, state) ResponseEntity
        -exchangeCodeForToken(code) Map
        -fetchGoogleUser(accessToken) Map
    }

    class JwtAuthFilter {
        +JwtUtils jwtUtils
        +UserRepository userRepository
        +doFilterInternal(request, response, chain)
        -isPublicPath(path) boolean
    }

    class SecurityConfig {
        +JwtAuthFilter jwtAuthFilter
        +securityFilterChain(HttpSecurity) SecurityFilterChain
        +passwordEncoder() PasswordEncoder
        +authenticationManager(AuthenticationConfiguration) AuthenticationManager
    }

    class JwtUtils {
        +String secretKey
        +long jwtExpiration
        +generateToken(username) String
        +getUsernameFromToken(token) String
        +validateToken(token) boolean
        -getSignInKey() Key
    }

    class UserRepository {
        <<interface>>
        +findByUsername(username) Optional~User~
        +findByPhoneNumber(phone) Optional~User~
        +findByEmail(email) Optional~User~
        +findByEmailOrPhoneNumber(identifier) Optional~User~
        +findByGoogleId(googleId) Optional~User~
        +existsByUsername(username) Boolean
        +existsByEmail(email) Boolean
        +existsByPhoneNumber(phone) Boolean
        +existsByGoogleId(googleId) Boolean
    }

    class LoginRequest {
        +String identifier
        +String password
    }

    class RegisterRequest {
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String password
        +String role
    }

    class UpdateProfileRequest {
        +String displayName
        +String email
        +String phoneNumber
        +String password
    }

    class AuthResponse {
        +String token
        +String message
    }

    class UserResponse {
        +Long id
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String role
    }

    class GoogleAuthResponse {
        +String token
        +boolean isNewUser
    }

    UserRepository --|> JpaRepository
    AuthController --> User
    AuthController --> LoginRequest
    AuthController --> RegisterRequest
    AuthController --> UpdateProfileRequest
    AuthController --> AuthResponse
    AuthController --> UserResponse
    AuthController --> UserRepository
    AuthController --> JwtUtils
    GoogleOAuthController --> User
    GoogleOAuthController --> UserRepository
    GoogleOAuthController --> JwtUtils
    GoogleOAuthController --> GoogleAuthResponse
    JwtAuthFilter --> JwtUtils
    JwtAuthFilter --> UserRepository
    SecurityConfig --> JwtAuthFilter
```

```mermaid
graph TB
    subgraph Controllers["🖥️ Controllers"]
        AC["🔐 AuthController<br/>/auth/register<br/>/auth/login<br/>/auth/me GET<br/>/auth/me PUT"]
        GOC["🌐 GoogleOAuthController<br/>/auth/oauth/google<br/>/auth/oauth/google/callback"]
    end

    subgraph Security["🛡️ Security Layer"]
        JAF["🔑 JwtAuthFilter<br/>(OncePerRequestFilter)"]
        SC["⚙️ SecurityConfig<br/>(SecurityFilterChain)"]
        PE["🔒 PasswordEncoder<br/>(BCrypt)"]
    end

    subgraph Utils["⚡ Utils"]
        JU["🎯 JwtUtils<br/>(Component)"]
    end

    subgraph Repository["📦 Repository"]
        UR["📂 UserRepository<br/>(JpaRepository)"]
    end

    subgraph Entities["📋 Entity"]
        U["👤 User<br/>(JPA Entity)"]
    end

    subgraph DTOs["📤 DTOs"]
        DTO1["📝 LoginRequest"]
        DTO2["📝 RegisterRequest"]
        DTO3["📝 UpdateProfileRequest"]
        DTO4["📤 AuthResponse"]
        DTO5["📤 UserResponse"]
        DTO6["📤 GoogleAuthResponse"]
    end

    Controllers --> |"depends on"| Repository
    Controllers --> |"uses"| Utils
    Controllers --> |"encodes"| Security
    Security --> |"validates|encodes"| Utils
    Security --> |"queries"| Repository
    Repository --> |"persists|queries"| Entities
    Controllers --> |"returns"| DTOs
```

---

## Class Diagram

```mermaid
classDiagram
    class User {
        +Long id
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String googleId
        +String password
        +LocalDateTime createdAt
        +String role
        +List~Text~ texts
        +List~QuizAttempt~ quizAttempts
    }

    class AuthController {
        +UserRepository userRepository
        +PasswordEncoder passwordEncoder
        +JwtUtils jwtUtils
        +register(RegisterRequest) ResponseEntity
        +login(LoginRequest) ResponseEntity
        +getCurrentUser() ResponseEntity
        +updateProfile(UpdateProfileRequest) ResponseEntity
    }

    class GoogleOAuthController {
        +String googleClientId
        +String googleClientSecret
        +String redirectUri
        +UserRepository userRepository
        +JwtUtils jwtUtils
        +RestTemplate restTemplate
        +initiateGoogleOAuth() ResponseEntity
        +handleGoogleCallback(code, state) ResponseEntity
        -exchangeCodeForToken(code) Map
        -fetchGoogleUser(accessToken) Map
    }

    class JwtAuthFilter {
        +JwtUtils jwtUtils
        +UserRepository userRepository
        +doFilterInternal(request, response, chain)
        -isPublicPath(path) boolean
    }

    class SecurityConfig {
        +JwtAuthFilter jwtAuthFilter
        +securityFilterChain(HttpSecurity) SecurityFilterChain
        +passwordEncoder() PasswordEncoder
        +authenticationManager(AuthenticationConfiguration) AuthenticationManager
    }

    class JwtUtils {
        +String secretKey
        +long jwtExpiration
        +generateToken(username) String
        +getUsernameFromToken(token) String
        +validateToken(token) boolean
        -getSignInKey() Key
    }

    class UserRepository {
        <<interface>>
        +findByUsername(username) Optional~User~
        +findByPhoneNumber(phone) Optional~User~
        +findByEmail(email) Optional~User~
        +findByEmailOrPhoneNumber(identifier) Optional~User~
        +findByGoogleId(googleId) Optional~User~
        +existsByUsername(username) Boolean
        +existsByEmail(email) Boolean
        +existsByPhoneNumber(phone) Boolean
        +existsByGoogleId(googleId) Boolean
    }

    class LoginRequest {
        +String identifier
        +String password
    }

    class RegisterRequest {
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String password
        +String role
    }

    class UpdateProfileRequest {
        +String displayName
        +String email
        +String phoneNumber
        +String password
    }

    class AuthResponse {
        +String token
        +String message
    }

    class UserResponse {
        +Long id
        +String username
        +String displayName
        +String email
        +String phoneNumber
        +String role
    }

    class GoogleAuthResponse {
        +String token
        +boolean isNewUser
    }

    UserRepository --|> JpaRepository
    AuthController --> User
    AuthController --> LoginRequest
    AuthController --> RegisterRequest
    AuthController --> UpdateProfileRequest
    AuthController --> AuthResponse
    AuthController --> UserResponse
    AuthController --> UserRepository
    AuthController --> JwtUtils
    GoogleOAuthController --> User
    GoogleOAuthController --> UserRepository
    GoogleOAuthController --> JwtUtils
    GoogleOAuthController --> GoogleAuthResponse
    JwtAuthFilter --> JwtUtils
    JwtAuthFilter --> UserRepository
    SecurityConfig --> JwtAuthFilter
```

---
