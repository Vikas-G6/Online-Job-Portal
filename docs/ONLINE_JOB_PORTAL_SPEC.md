# Online Job Portal: Complete Specification

Version 1.1 (open questions decided). Consolidated from `backend.md` and `frontend.md`, restructured into numbered requirements with acceptance criteria so it can drive a spec-driven build (requirements, design, tests, implementation, traceability).

## How to read this spec

- Every requirement has an ID (for example `AUTH-03`). Tests and code comments should reference these IDs.
- **[DOC]** means the requirement comes directly from your original documents.
- **[ADDED]** means this spec adds it because the original documents were silent and the app would not work or would be unsafe without it. These are listed again in section 17 so you can accept or reject each one.
- **[DECIDED]** means the original documents left a question open or contained a weakness, and this spec now states the decision (section 17 lists each one with its reason). The agent shall not guess; it follows the decision.
- "shall" means mandatory. Acceptance criteria are written as *Given / When / Then* in short form.
- Secrets in the original documents (JWT secret, internal key, DB password) are **not** reproduced here. See section 3.

---

## 1. Product overview

An online job portal with two roles.

| Role | Who | Can do |
|---|---|---|
| **ADMIN** (employer) | Companies posting jobs | Register, post/edit/delete jobs, view jobs, search job seekers by skill, message job seekers, view messages, manage users and admins, edit own profile |
| **USER** (job seeker) | People looking for jobs | Register, browse and search jobs, apply, save jobs to a basket and apply from it, message employers, view messages, edit own profile |

The backend is five independently runnable Spring Boot applications (microservices). The frontend is a React single-page app that talks only to the API gateway.

### Goals
- G1. Demonstrate a working microservices system: discovery, gateway, stateless JWT security, service-to-service calls, and circuit breakers with fallbacks.
- G2. A complete job-seeker and employer workflow in a usable UI.
- G3. Every behaviour covered by an automated test where the spec lists one.

### Non-goals
Resume file upload, email notifications, payments, real-time chat, message queues, and production deployment are out of scope.

---

## 2. Architecture

```
React SPA (localhost:3000)
        |  HTTP + JWT (Authorization: Bearer)
        v
API Gateway :8080  ----- looks up services in ----->  Eureka :8761
   |        |        |
   v        v        v
auth-service   admin-service   user-service
   :8091          :8083           :8081
 jobportaldb     admin_db     user_service_db
   ^   ^_______ Feign (X-Internal-Service-Key) _______^   ^
   |__________________ Feign ____________________________|
```

| Service | Port | Package base | Database | Registers as |
|---|---|---|---|---|
| eureka-server | 8761 | com.jobportal.eurekaserver | none | n/a |
| api-gateway | 8080 | com.jobportal.apigateway | none | API-GATEWAY |
| auth-service | 8091 | com.jobportal.authservice | jobportaldb | AUTH-SERVICE |
| admin-service | 8083 | com.ojp.admin | admin_db | ADMIN-SERVICE |
| user-service | 8081 | com.jobportal.userservice | user_service_db | USER-SERVICE |

**Startup order:** eureka-server, admin-service, auth-service, user-service, api-gateway, then the frontend.

| ID | Requirement | Acceptance |
|---|---|---|
| GEN-01 [DOC] | Each service shall be a separate Maven project that builds and starts on its own. | Given each project, when `mvn spring-boot:run` is run, then it starts without the others (Feign calls fall back, see INT section). |
| GEN-02 [DOC] | Services shall register with Eureka and be called by service name through the load balancer (`lb://`). | Given all services running, when the Eureka dashboard is opened, then ADMIN-SERVICE, AUTH-SERVICE, USER-SERVICE and API-GATEWAY are listed UP. |
| GEN-03 [DOC] | The frontend shall call only the gateway (port 8080), never a service port. | Given the frontend code, when searched for ports 8081, 8083 and 8091, then none appear. |
| GEN-04 [DOC] | Each service shall expose Swagger UI (SpringDoc 2.3.0) with a Bearer JWT scheme. Auth: "Auth Service API"; Admin: "Admin Service API"; User: "User Service API"; version "1.0". | Given a running service, when `/swagger-ui.html` is opened, then the API list loads and an Authorize button accepts a Bearer token. |

---

## 3. Technology and configuration

### 3.1 Technology (fixed)

Java 21; Spring Boot 3.2.5; Spring Cloud 2023.0.1; Spring Security + JWT (jjwt 0.11.5, HS512); Spring Data JPA + Hibernate; MySQL 8; OpenFeign; Eureka; Spring Cloud Gateway (WebFlux); Spring Cloud LoadBalancer; Resilience4j; SpringDoc OpenAPI 2.3.0; Lombok; Maven.
Frontend: React 18.2.0, React Router DOM 6.22.0, Axios 1.6.7, jwt-decode 4.0.0, react-toastify 10.0.4, react-icons 5.0.1, react-scripts 5.0.1 (Create React App).
Tests (backend): JUnit 5, Mockito, MockMvc, H2 in-memory (test scope only).

### 3.2 Environment variables [ADDED]

The original documents contain real-looking secrets and a database password in plain text. None of them shall appear in source control.

| Variable | Used by | Meaning | Example (dummy) |
|---|---|---|---|
| `DB_USERNAME` | auth, admin, user | MySQL user | `root` |
| `DB_PASSWORD` | auth, admin, user | MySQL password | `change_me` |
| `JWT_SECRET` | auth, admin, user | Base64 secret, at least 64 bytes decoded (HS512). Generate with `openssl rand -base64 64`. **Must be identical in all three services.** | `<generated>` |
| `JWT_EXPIRATION_MS` | auth | Token lifetime in ms | `86400000` |
| `INTERNAL_API_KEY` | auth, admin, user | Shared service-to-service key. **Identical in all three.** | `<generated>` |
| `EUREKA_URL` | all except eureka | Eureka service URL | `http://localhost:8761/eureka/` |
| `REACT_APP_API_URL` | frontend | Gateway base URL | `http://localhost:8080` |

| ID | Requirement | Acceptance |
|---|---|---|
| CFG-01 [ADDED] | All secrets and the DB credentials shall be read from the variables above, through `${...}` placeholders in `application.yml` or `application.properties`. | Given the repo, when `git grep` is run for the old DB password, the old internal key and the first characters of the old JWT secret from the original documents, then there are no matches. |
| CFG-02 [ADDED] | A `.env.example` with dummy values shall be committed. `.env` shall be listed in `.gitignore`. | Given a fresh clone, when `.env.example` is copied to `.env`, then the services start. |
| CFG-03 [DOC] | JWT expiry is 24 hours (86,400,000 ms). | Given a token just issued, when its `exp` and `iat` are compared, then the difference is 86,400 seconds. |
| CFG-04 [DOC] | MySQL connections use `createDatabaseIfNotExist=true`, `useSSL=false`, `allowPublicKeyRetrieval=true`. `ddl-auto=update`. | Given an empty MySQL server, when a service starts, then its database and tables are created. |
| CFG-05 [DOC] | Eureka instance settings for every client: `prefer-ip-address=false`, `hostname=localhost`, `instance-id=<name>:localhost:<port>`; `spring.cloud.loadbalancer.cache.enabled=false`. | Given the Eureka dashboard, then instance IDs follow the `NAME:localhost:PORT` pattern. |
| CFG-06 [DOC] | Feign: circuit breaker enabled; connect timeout 5000 ms; read timeout 5000 ms; logger level BASIC. | Given a Feign client, when the callee is slow beyond 5 s, then the call times out and the fallback runs. |
| CFG-07 [DOC] | Resilience4j default config: sliding window 10, failure rate threshold 50%, wait in open state 10 s, 3 calls permitted in half-open, health indicator registered. | Given 10 calls with at least 5 failures, then the breaker opens, and after 10 s it allows 3 probe calls. |

---

## 4. Data model

### 4.1 `jobportaldb` (auth-service)

`user_credential`

| Column | Type | Constraints |
|---|---|---|
| id | BIGINT | PK, auto increment |
| username | VARCHAR | unique, not null |
| password | VARCHAR | not null, BCrypt hash |
| role | ENUM('USER','ADMIN') | not null, stored as string |
| status | ENUM('ACTIVE','INACTIVE') | not null, stored as string |
| created_at | DATETIME | not null, set in `@PrePersist` |

### 4.2 `admin_db` (admin-service)

`admin_profile`: `admin_id` BIGINT PK (**not** auto-generated, equals `user_credential.id`); `name` VARCHAR not null; `email` VARCHAR unique not null; `organization_name`; `address`; `contact_no`.

`job`: `job_id` BIGINT PK auto; `title`, `location`, `company` VARCHAR not null; `description` TEXT not null; `experience`, `salary`, `notice_period`, `contact_email`, `status` VARCHAR; `admin_id` BIGINT not null.

`employer_message`: `message_id` BIGINT PK auto; `description` TEXT not null; `job_seeker_id` BIGINT not null; `employer_id` BIGINT not null; `date` DATETIME (default now).

### 4.3 `user_service_db` (user-service)

`user_profile`: `user_id` BIGINT PK (**not** auto-generated, equals `user_credential.id`); `name`; `email` unique; `phone`; `address`; `skill_set`; `resume_url`.

`job_application`: `Application_Id` BIGINT PK auto; `User_Id` BIGINT not null; `Job_Id` BIGINT not null; `application_Date` DATE. (Column names are mixed case exactly as written.)

`job_basket`: `basket_id` BIGINT PK auto; `user_id` BIGINT not null; `job_id` BIGINT not null; `added_date` DATE (default today).

`job_seeker_message`: `message_id` BIGINT PK auto; `description` TEXT not null; `job_seeker_id`, `employer_id`, `job_id` BIGINT not null; `date` DATETIME (default now).

| ID | Requirement | Acceptance |
|---|---|---|
| DB-01 [DOC] | Each service owns its own database. No service shall query another service's database. | Given the code, then no service has a datasource for another service's schema. |
| DB-02 [DOC] | `admin_profile.admin_id` and `user_profile.user_id` shall be set from the auth-service credential id, never auto-generated. | Given registration of a user with credential id 7, then `user_profile.user_id = 7`. |
| DB-03 [DOC] | Profile creation shall be idempotent: if a profile with that id or email already exists, no new row is created and no error is raised. Insert with `EntityManager.persist` and `flush`. | Given the create call is sent twice with the same body, then there is one row and both calls succeed. |
| DB-04 [DOC] | `job_basket` shall not hold two rows for the same user and job. | Given a job already in a basket, when it is added again, then the request is rejected (see USR basket requirement). |
| DB-05 [DECIDED OQ-1] | `job_application` shall have a unique constraint on (`User_Id`, `Job_Id`). `job_basket` shall have a unique constraint on (`user_id`, `job_id`). | Given two inserts for the same user and job, then the second fails at the database level as well as in the service. |
| DB-06 [DECIDED OQ-7] | `job.status` shall be a string-stored enum with values `ACTIVE`, `CLOSED`, `DRAFT`. A job created without a status is `ACTIVE`. An unknown value is rejected with 400. | Given `status: "OPEN"`, then 400. Given no status, then the saved job is `ACTIVE`. |

---

## 5. Shared security rules

| ID | Requirement | Acceptance |
|---|---|---|
| SEC-01 [DOC] | JWT: algorithm HS512; subject = username; claims `userId` (Long), `role` (string **without** the `ROLE_` prefix: `USER` or `ADMIN`); expiry per CFG-03. The signing key is the Base64-decoded `JWT_SECRET`. | Given a login, when the token is decoded, then `sub`, `userId`, `role`, `exp` are present and `role` is `USER` or `ADMIN`. |
| SEC-02 [DOC] | All services are stateless with CSRF disabled. Only auth-service creates tokens; admin-service and user-service only validate them (`JwtUtil` without `generateToken`). | Given admin-service, then `JwtUtil` has no token-generation method. |
| SEC-03 [DOC] | Authorities are `ROLE_<role>` (`ROLE_USER`, `ROLE_ADMIN`). | Given a USER token, then an ADMIN-only endpoint returns 403. |
| SEC-04 [DOC] | Internal endpoints are protected by the header `X-Internal-Service-Key`, compared to `INTERNAL_API_KEY`. | Given a request to an internal create endpoint without the header, then it is rejected (see AUTH/ADM/USR internal requirements). |
| SEC-05 [DOC] | Passwords are stored BCrypt-encoded and never returned in any response. | Given a response from any endpoint, then no `password` field exists. |
| SEC-06 [ADDED] | Error bodies are JSON `{"message": "<text>"}` (plus `"errors": {...}` for validation). | Given any 4xx/5xx from a service, then the body parses as JSON with `message`. |
| SEC-07 [ADDED] | Bean validation failures (`@Valid`) return **400**, not 500, with per-field messages. | Given a request with a blank required field, then status 400 and `errors` names the field. |
| SEC-08 [DECIDED OQ-3] | **Ownership.** A USER may only read or change data that belongs to their own `userId` (from the JWT). Wherever a USER calls an endpoint with a user id in the path or body (`userId`, `jobSeekerId`), a different id returns **403**. Employer (ADMIN) ownership rules are given per endpoint. Ownership checks live in the controllers or a small security helper, so the documented service-layer signatures and unit tests stay valid. | Given USER 7 calls `GET /api/basket/8`, then 403. Given USER 7 calls `GET /api/basket/7`, then 200. |
| SEC-09 [DECIDED OQ-2] | **Internal principal.** When a request carries a correct `X-Internal-Service-Key`, the filter shall authenticate it as principal `internal-service` with authority `ROLE_INTERNAL`. Endpoints that list INTERNAL accept it. A wrong or missing key gives no authentication. | Given admin-service calls `GET /api/users` with the key, then 200. Given the same call without the key or a token, then 401. |
| SEC-10 [DECIDED OQ-3] | **userId in user-service.** The user-service `JwtAuthFilter` keeps the principal as the username (as documented) and also stores the token's `userId` in the authentication details, so ownership checks (SEC-08) can read it. admin-service already uses `userId` (Long) as the principal. | Given a USER token, then a controller can read the caller's userId. |
| SEC-11 [DECIDED OQ-1, OQ-5] | **Conflict handling.** user-service has a `DuplicateResourceException` mapped to **409** with the standard error body. Ownership violations return **403**. | Given a duplicate application, then 409. |

---

## 6. eureka-server (port 8761)

Dependencies: `spring-boot-starter-web`, `spring-cloud-starter-netflix-eureka-server`, `spring-boot-devtools` (runtime). Main class `EurekaServerApplication` with `@SpringBootApplication` and `@EnableEurekaServer`.

| ID | Requirement | Acceptance |
|---|---|---|
| EUR-01 [DOC] | `spring.application.name=eureka-server`, `server.port=8761`. | Given startup, then the dashboard opens at `http://localhost:8761`. |
| EUR-02 [DOC] | It shall not register with itself or fetch the registry (`register-with-eureka=false`, `fetch-registry=false`); default zone `http://localhost:8761/eureka/`. | Given the dashboard, then eureka-server is not listed as a registered instance. |
| EUR-03 [ADDED] | A context-load test shall exist. | Given `mvn test`, then it passes. |

---

## 7. api-gateway (port 8080)

Dependencies: `spring-boot-starter-webflux`, `spring-cloud-starter-gateway`, `spring-cloud-starter-netflix-eureka-client`, `spring-cloud-starter-loadbalancer`, `spring-boot-starter-actuator`, `spring-boot-starter-security`. `spring.main.allow-bean-definition-overriding=true`. Actuator exposes `health,info`.

### 7.1 Routes [DOC]

| Route id | Path predicate | Target | Filter |
|---|---|---|---|
| auth-service | `/api/auth/**` | lb://AUTH-SERVICE | none |
| user-basket | `/api/basket/**` | lb://USER-SERVICE | none |
| user-messages | `/api/messages/**` | lb://USER-SERVICE | none |
| user-applications | `/api/applications/**` | lb://USER-SERVICE | none |
| user-api | `/api/users/**` | lb://USER-SERVICE | none |
| admin-api | `/api/admin/**` | lb://ADMIN-SERVICE | none |

| ID | Requirement | Acceptance |
|---|---|---|
| GW-01 [DOC][DECIDED OQ-8] | The gateway shall define exactly the routes in the table above. The documented `/user/**` and `/admin/**` routes (with StripPrefix) are **removed** because the frontend never uses them. | Given the gateway, when each path prefix is called, then the correct service answers. Given `/user/api/users`, then 404. |
| GW-02 [DOC] | Security at the gateway: permit all exchanges, CSRF disabled. **No JWT validation at the gateway.** Each service validates tokens itself. | Given a request with no token to a protected path, then the downstream service (not the gateway) answers 401. |
| GW-03 [DECIDED OQ-4] | The gateway shall reject with **404** any request whose path contains `/internal/`, using a global filter that runs before routing. Internal endpoints are reachable only service to service, directly on the service ports. | Given `POST http://localhost:8080/api/users/internal/create`, `POST http://localhost:8080/api/admin/internal/create` and `GET http://localhost:8080/internal/jobs/1`, then each returns 404. Given the same calls directly to the service ports with a valid key, then they succeed. |
| GW-04 [ADDED] | CORS shall be configured at the gateway to allow origin `http://localhost:3000`, methods GET, POST, PUT, DELETE, OPTIONS, headers `Authorization` and `Content-Type`. Without this the browser blocks the frontend. | Given a browser at localhost:3000, when it logs in via the gateway, then no CORS error occurs and preflight OPTIONS returns 200. |
| GW-05 [DOC] | Eureka client settings per CFG-05; lease renewal 30 s, lease expiration 90 s; health check enabled. | Given the dashboard, then API-GATEWAY is UP. |
| GW-06 [ADDED] | A test shall verify the route table (every route id, predicate and target). | Given `mvn test`, then it passes. |

---

## 8. auth-service (port 8091)

Dependencies: web, security, data-jpa, validation, mysql-connector-j, eureka-client, openfeign, loadbalancer, circuitbreaker-resilience4j, springdoc-openapi-starter-webmvc-ui 2.3.0, jjwt-api/impl/jackson 0.11.5, lombok. Main class has `@SpringBootApplication`, `@EnableDiscoveryClient`, `@EnableFeignClients`. Application name `AUTH-SERVICE`. Database `jobportaldb`.

### 8.1 DTOs

| DTO | Fields and validation |
|---|---|
| RegisterRequest | username (@NotBlank, size 3-50), password (@NotBlank, min 6), name, email, phone, address, skillSet |
| AdminRegisterRequest | username (@NotBlank, size 3-50), password (@NotBlank, min 6), name (@NotBlank), email (@NotBlank, @Email), organizationName, address, contactNo |
| LoginRequest | username (@NotBlank), password (@NotBlank) |
| LoginResponse | token, userId, username, role |
| RegisterResponse | id, username, role, status, message |

### 8.2 API

| ID | Endpoint | Auth | Behaviour and success | Errors | Acceptance |
|---|---|---|---|---|---|
| AUTH-01 [DOC] | `POST /api/auth/register` | public | Saves credential with role USER, status ACTIVE, BCrypt password; calls user-service create-profile (soft fail). **201** `RegisterResponse` | 400 duplicate username or invalid body | Given a new username, then 201 and a user profile exists in user-service. Given the same username again, then 400. |
| AUTH-02 [DOC] | `POST /api/auth/admin/register` | public | Same for role ADMIN; calls admin-service create-profile (soft fail). **201** | 400 | Given a new employer, then 201 and an admin profile exists. |
| AUTH-03 [DOC] | `POST /api/auth/admin/create-admin` | ROLE_ADMIN | Creates another ADMIN. **201** | 401/403 without admin token | Given a USER token, then 403. Given an ADMIN token, then 201. |
| AUTH-04 [DOC] | `POST /api/auth/login` | public | Authenticates via `AuthenticationManager`; returns **200** `LoginResponse` with a JWT carrying `userId`, `sub`, `role` | 401 bad credentials or INACTIVE user | Given correct credentials, then 200 and a decodable token. Given a wrong password, then 401. |
| AUTH-05 [DOC] | `GET /api/auth/validate?token=` | public | **200** `{"valid": true}` for a good token | `{"valid": false}` for a bad token | Given a valid token, then valid true. |
| AUTH-06 [DOC] | `GET /api/auth/role/{username}` | public | **200** `{"role": "..."}` | 400 unknown user | Given an existing username, then its role is returned. |
| AUTH-07 [DOC] | `DELETE /internal/auth/admin/{adminId}` | internal, no JWT | Deletes the credential after checking its role is ADMIN. **204** | 400 if not an ADMIN credential | Given an ADMIN id, then 204 and the row is gone. Given a USER id, then 400 and the row remains. |

### 8.3 Logic and security

| ID | Requirement | Acceptance |
|---|---|---|
| AUTH-08 [DOC] | `CustomUserDetailsService` loads by username and rejects users whose status is not ACTIVE. | Given an INACTIVE user, when logging in, then 401. |
| AUTH-09 [DOC] | `JwtUtil` provides `generateToken(userId, username, role)`, `extractClaims`, `extractUsername`, `extractRole`, `extractUserId`, `validateToken`. | Given a generated token, then each extractor returns the original value. |
| AUTH-10 [DOC] | Security config: stateless; permit `POST /api/auth/register`, `POST /api/auth/admin/register`, `POST /api/auth/login`, `GET /api/auth/validate`, `GET /api/auth/role/**`, `/internal/**`, `/v3/api-docs/**`, `/swagger-ui/**`; everything else authenticated. `DaoAuthenticationProvider` with `CustomUserDetailsService` and `BCryptPasswordEncoder`. `JwtAuthFilter` runs before `UsernamePasswordAuthenticationFilter`. | Given a protected path with no token, then 403 (empty security context). |
| AUTH-11 [DOC] | `JwtAuthFilter` reads the Bearer token, sets an authenticated token with authority `ROLE_<role>`, and silently ignores invalid tokens (context stays empty). | Given a garbage token, then the request is treated as unauthenticated. |
| AUTH-12 [DOC] | `GlobalExceptionHandler`: `IllegalArgumentException` gives 400; any other `Exception` gives 500. (See SEC-06, SEC-07.) | Given a duplicate-username registration, then 400. |
| AUTH-13 [DOC] | Feign clients: `UserServiceClient` (`USER-SERVICE`) posts to `/api/users/internal/create` with header `X-Internal-Service-Key` and body `{userId, name, email, phone, address, skillSet}`. `AdminServiceClient` (`ADMIN-SERVICE`) posts to `/api/admin/internal/create` with the header and body `{adminId, name, email, organizationName, address, contactNo}`. | Given a registration, then the callee receives the header and body. |
| AUTH-14 [DOC] | Both fallbacks log the error and **do not throw** (soft fail). | Given user-service is down, when a job seeker registers, then 201 is still returned and the credential exists. |
| AUTH-15 [ADDED] | Tests: register success; register duplicate username; login success; login wrong password; token claims contain userId, sub, role; both fallbacks do not throw. | Given `mvn test`, then all pass. |

---

## 9. admin-service (port 8083)

Application name `ADMIN-SERVICE`; package base `com.ojp.admin`; database `admin_db`. Dependencies as auth-service minus token generation. Main class has `@EnableDiscoveryClient` and `@EnableFeignClients`.

### 9.1 DTOs

`AdminDto.Request` (name @NotBlank, email @NotBlank @Email, organizationName, address, contactNo); `AdminDto.Response` (adminId, name, email, organizationName, address, contactNo).
`JobDto.Request` (title, location, description, company all @NotBlank; experience, salary, noticePeriod; contactEmail @Email; status); `JobDto.Response` (jobId, title, location, description, company, experience, salary, noticePeriod, contactEmail, status, adminId).
`EmployerMessageDto.Request` (description @NotBlank, jobSeekerId @NotNull, employerId @NotNull); `Response` (messageId, description, jobSeekerId, employerId, date).

### 9.2 Security

| ID | Requirement | Acceptance |
|---|---|---|
| ADM-01 [DOC] | Stateless, CSRF off, no form login, no HTTP basic. Permit `/v3/api-docs/**`, `/swagger-ui/**`, `/swagger-ui.html`, `/api/admin/internal/**`, `/internal/**`; all else authenticated. | Given no token on `GET /api/admin/jobs`, then 401. |
| ADM-02 [DOC] | `JwtAuthFilter` first checks `X-Internal-Service-Key`; if it matches, the request passes through. Otherwise it validates the Bearer token. The authentication **principal is the `userId` (Long)**, authority `ROLE_<role>`. Missing, expired or invalid token returns **401**. `shouldNotFilter` covers `/v3/api-docs`, `/swagger-ui`, `/api/admin/internal`, `/internal`. | Given an expired token, then 401. |
| ADM-03 [DOC] | No `UserDetailsService` is needed (JWT only). | Given the config, then no user lookup occurs. |

### 9.3 Endpoints

`AdminController` at `/api/admin`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| ADM-10 [DOC] | `GET /` | ADMIN | All admins | Given an ADMIN token, then a list; given a USER token, then 403. |
| ADM-11 [DOC] | `GET /{id}` | ADMIN | One admin; 404 if missing | Missing id gives 404. |
| ADM-12 [DOC] | `PUT /{id}` (@Valid) | ADMIN | Updates profile fields | Given a valid body, then fields change. |
| ADM-13 [DOC] | `DELETE /{id}` | ADMIN | Calls auth-service delete credentials via Feign, then deletes the profile | Given auth-service up, then credential and profile are both gone. Given auth-service down, then the error surfaces (see ADM-43). |
| ADM-14 [DOC] | `POST /jobs` (@Valid) | ADMIN | Creates a job; `adminId` is taken from the token principal `(Long) auth.getPrincipal()`. **201** | Given an ADMIN with userId 5, then the job has adminId 5. |
| ADM-15 [DOC][DECIDED OQ-2] | `GET /seekers/search?skillSet=` | ADMIN | Calls user-service search via Feign; the Feign call sends `X-Internal-Service-Key` (SEC-09) | Given user-service down, then an empty list. Given user-service up, then matching seekers. |
| ADM-16 [DOC] | `GET /jobs/{jobId}/details` | ADMIN | Job by id | Missing id gives 404. |

`JobController` at `/api/admin/jobs`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| ADM-20 [DOC] | `GET /` | USER or ADMIN | All jobs | Both roles get 200. |
| ADM-21 [DOC] | `GET /{id}` | USER or ADMIN | One job; 404 if missing | |
| ADM-22 [DOC][DECIDED OQ-3] | `GET /my-jobs/{adminId}` | ADMIN, own id only | Jobs of that admin; `adminId` must equal the caller's userId, else 403 | Given ADMIN 5 requests `/my-jobs/6`, then 403. |
| ADM-23 [DOC][DECIDED OQ-3] | `PUT /{id}` (@Valid) | ADMIN, job owner only | Updates all job fields; 404 if missing; 403 if `job.adminId` differs from the caller's userId | Given a job owned by ADMIN 5, when ADMIN 6 updates it, then 403. |
| ADM-24 [DOC][DECIDED OQ-3] | `DELETE /{id}` | ADMIN, job owner only | Deletes the job; 403 if not the owner | Given a job owned by ADMIN 5, when ADMIN 6 deletes it, then 403 and the job remains. |
| ADM-25 [DOC][DECIDED OQ-3] | `POST /messages` (@Valid) | ADMIN | Employer sends a message to a seeker; `employerId` in the body must equal the caller's userId (else 403); date = now. **201** | Stored with a timestamp; a body with another employer's id gives 403. |
| ADM-26 [DOC][DECIDED OQ-3] | `GET /messages/from-jobseeker/{jobSeekerId}` | USER (own id only) or ADMIN | Messages addressed to that job seeker | Given USER 7 requests seeker 8, then 403. |
| ADM-27 [DOC][DECIDED OQ-3] | `GET /messages/by-employer/{employerId}` | ADMIN, own id only | Messages sent by that employer | Given ADMIN 5 requests employer 6, then 403. |

Internal endpoints (no JWT):

| ID | Endpoint | Result | Acceptance |
|---|---|---|---|
| ADM-30 [DOC] | `POST /api/admin/internal/create` | Validates `X-Internal-Service-Key` manually, then idempotently creates an admin profile (DB-03) | Wrong or missing key rejected (401/403); right key creates a profile. |
| ADM-31 [DOC][DECIDED OQ-4] | `GET /internal/jobs/{jobId}`; `GET /internal/jobs/search?skill=&designation=&company=`; `GET /internal/jobs/search/location?location=` | Job lookup and search for user-service. Each endpoint **validates `X-Internal-Service-Key`** manually, like ADM-30. | Missing or wrong key gives 401. Search matches by LIKE on description/title/company (skill/designation/company); location search is case-insensitive "contains". |

### 9.4 Services, Feign, errors, tests

| ID | Requirement | Acceptance |
|---|---|---|
| ADM-40 [DOC] | `JobService`: `getAllJobs`, `getJobById`, `getJobsByAdminId`, `createJob(req, adminId)`, `updateJob`, `deleteJob`, `searchBySkillDesignationOrCompany`, `searchByLocation`. Repository: `findByAdminId`, a `@Query` JPQL LIKE search, `findByLocationContainingIgnoreCase`. | See ADM-20 to ADM-24. |
| ADM-41 [DOC] | `AdminService.createAdminProfile` is idempotent (`existsByEmail` and `existsById`). `deleteAdmin` calls auth-service first, then deletes the profile. | See DB-03, ADM-13. |
| ADM-42 [DOC] | `EmployerMessageService`: `sendMessage` (date = now), `getMessagesByEmployer`, `getMessagesByJobSeeker`. | See ADM-25 to ADM-27. |
| ADM-43 [DOC] | Feign `AuthServiceClient` (`auth-service`): `DELETE /internal/auth/admin/{adminId}`. Fallback **throws** `RuntimeException("Auth service unavailable")`. | Given auth-service down, when deleting an admin, then the call fails with that message. |
| ADM-44 [DOC][DECIDED OQ-2] | Feign `UserServiceClient` (`USER-SERVICE`): `GET /api/users/search?skillSet=` and `GET /api/users`, returning `UserResponse` (userId, name, email, phone, address, skillSet). Every call sends `X-Internal-Service-Key`. Fallback returns an empty list. | Given user-service down, then an empty list, no error. Given user-service up, then results are returned. |
| ADM-45 [DOC] | `GlobalExceptionHandler`: `EntityNotFoundException` gives 404; `IllegalArgumentException` gives 400; `Exception` gives 500. | |
| ADM-46 [DOC] | `JobServiceTest`: `getAllJobs_returnsList`, `getJobById_found`, `getJobById_notFound`, `createJob_savesAndReturns`, `updateJob_updatesFields`, `updateJob_notFound`, `deleteJob_callsRepository`, `getJobsByAdminId_returnsFilteredList`. | All pass. |
| ADM-47 [ADDED] | Controller tests (MockMvc): USER gets 403 on ADMIN endpoints; 401 without a token; internal create and `/internal/jobs/**` with and without the key; ownership (SEC-08): ADMIN 6 cannot update or delete ADMIN 5's job, cannot read `/my-jobs/5`, and cannot post a message with employerId 5. | All pass. |
| ADM-48 [DECIDED OQ-7] | `JobDto.Request.status` accepts only `ACTIVE`, `CLOSED`, `DRAFT` (DB-06); null becomes `ACTIVE`. | Given `status: "OPEN"`, then 400. |

---

## 10. user-service (port 8081)

Application name `USER-SERVICE`; package base `com.jobportal.userservice`; database `user_service_db`.

### 10.1 DTOs

`UserDto.Request` (name @NotBlank size 2-50, email @NotBlank @Email, phone, address, skillSet); `Response` (userId, name, email, phone, address, skillSet).
`JobApplicationDto.Request` (userId @NotNull @Positive, jobId @NotNull @Positive, applicationDate @PastOrPresent); `Response` (applicationId, userId, jobId, applicationDate).
`JobBasketDto.Request` (userId, jobId, both @NotNull @Positive); `Response` (basketId, userId, jobId, addedDate).
`JobSeekerMessageDto.Request` (description @NotBlank, jobSeekerId, employerId, jobId all @NotNull); `Response` (messageId, description, jobSeekerId, employerId, jobId, date).

### 10.2 Security

| ID | Requirement | Acceptance |
|---|---|---|
| USR-01 [DOC][DECIDED OQ-2] | Stateless, CSRF off. Permit without authentication only: `/v3/api-docs/**`, `/swagger-ui/**`, `/api/users/internal/**` (the internal key is checked in the controller). **Everything else requires authentication**, including `GET /api/users` and `GET /api/users/search`, which are no longer public. | Given no token on `GET /api/users`, then 401. |
| USR-02 [DOC][DECIDED OQ-2, OQ-3] | `JwtAuthFilter` checks `X-Internal-Service-Key` first: if it matches, authenticate as `internal-service` with `ROLE_INTERNAL` (SEC-09). Otherwise validate the Bearer token; principal is the **username (String)** with `userId` in the details (SEC-10); authority `ROLE_<role>`. Missing, expired or invalid token gives **401**. `shouldNotFilter`: `/v3/api-docs`, `/swagger-ui`, `/api/users/internal`. | Given an expired token on `POST /api/applications`, then 401. Given a valid key on `GET /api/users`, then 200. |

### 10.3 Endpoints

`UserController` at `/api/users`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| USR-10 [DOC][DECIDED OQ-3] | `GET /{id}` | USER (own id only) or ADMIN | One profile; 404 if missing | USER 7 reading profile 8 gives 403. |
| USR-11 [DOC][DECIDED OQ-2] | `GET /` | ADMIN or INTERNAL | All profiles | ADMIN token gives 200; USER token 403; no token 401; valid internal key 200. |
| USR-12 [DOC][DECIDED OQ-2] | `GET /search?skillSet=` | ADMIN or INTERNAL | Profiles whose skill set contains the text, ignoring case | "java" matches "Java, Spring". No match gives an empty list. USER token gives 403; no token 401. |
| USR-13 [DOC][DECIDED OQ-3] | `PUT /{id}` (@Valid) | USER (own id only) or ADMIN | Updates all fields; 404 if missing | USER 7 updating profile 8 gives 403. |
| USR-14 [DOC] | `DELETE /{id}` | ADMIN | Deletes; 404 if missing | |
| USR-15 [DOC] | `GET /jobs/search?skill=&designation=&company=` | USER or ADMIN | Proxies admin-service via Feign | admin-service down gives an empty list. |
| USR-16 [DOC] | `GET /jobs/search/location?location=` | USER or ADMIN | Proxies admin-service | Empty list on failure. |
| USR-17 [DOC] | `GET /jobs/{jobId}` | USER or ADMIN | Proxies admin-service | admin-service down: the call fails (fallback throws). |

`JobApplicationController` at `/api/applications`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| USR-20 [DOC][DECIDED OQ-1, OQ-3] | `POST /` (@Valid) | USER | Applies; `userId` in the body must equal the caller's (else 403); `applicationDate` = today if null; a second application for the same user and job returns **409** (SEC-11). **201** on success | First apply gives 201; same user and job again gives 409; another user's id gives 403. |
| USR-21 [DOC][DECIDED OQ-3] | `GET /user/{userId}` | USER (own id only) or ADMIN | Applications of a user | USER 7 requesting user 8 gives 403. |
| USR-22 [DOC] | `GET /job/{jobId}` | ADMIN | Applications for a job | USER gets 403. |
| USR-23 [DOC] | `DELETE /{applicationId}` | ADMIN | Deletes; 404 if missing. **204** | |

`JobBasketController` at `/api/basket`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| USR-30 [DOC][DECIDED OQ-3, OQ-5] | `POST /` (@Valid) | USER | Adds a job to the basket; body `userId` must equal the caller's (else 403); a job already in the basket returns **409**. **201** on success | Adding the same job twice: second call 409. |
| USR-31 [DOC][DECIDED OQ-3] | `GET /{userId}` | USER (own id only) or ADMIN | The user's basket | USER 7 requesting basket 8 gives 403. |
| USR-32 [DOC][DECIDED OQ-3] | `DELETE /{userId}/job/{jobId}` | USER (own id only) | Removes (checks it exists, `@Transactional`). **204** | Missing entry gives 404; another user's id gives 403. |
| USR-33 [DOC][DECIDED OQ-1, OQ-3] | `POST /{userId}/apply/{jobId}` | USER (own id only) | Checks the basket entry exists (404 if not), then applies via `JobApplicationService`; if already applied returns **409**. The basket entry is kept. **201** on success | Not in basket gives 404; in basket creates an application; repeating gives 409. |

`JobSeekerMessageController` at `/api/messages`:

| ID | Endpoint | Role | Result | Acceptance |
|---|---|---|---|---|
| USR-40 [DOC][DECIDED OQ-3] | `POST /` (@Valid) | USER | Seeker messages an employer; `jobSeekerId` in the body must equal the caller's (else 403); date = now. **201** | Another user's id gives 403. |
| USR-41 [DOC][DECIDED OQ-3] | `GET /employer/{employerId}` | ADMIN, own id only | Messages received by that employer (a USER cannot read employers' inboxes) | USER gives 403; ADMIN 5 requesting employer 6 gives 403. |
| USR-42 [DOC][DECIDED OQ-3] | `GET /jobseeker/{jobSeekerId}` | USER (own id only) or ADMIN | Messages sent by that seeker | USER 7 requesting seeker 8 gives 403. |

Internal:

| ID | Endpoint | Result | Acceptance |
|---|---|---|---|
| USR-50 [DOC] | `POST /api/users/internal/create` | Validates `X-Internal-Service-Key` manually, then idempotent profile create (DB-03) | Wrong or missing key rejected; right key creates. |

### 10.4 Feign, errors, tests

| ID | Requirement | Acceptance |
|---|---|---|
| USR-60 [DOC][DECIDED OQ-4] | Feign `AdminClient` (`ADMIN-SERVICE`): `GET /internal/jobs/{jobId}`, `GET /internal/jobs/search?skill=&designation=&company=`, `GET /internal/jobs/search/location?location=`, returning `JobResponse` (jobId, title, location, description, company, experience, salary, noticePeriod, contactEmail, status, adminId). **Every call sends `X-Internal-Service-Key`.** Fallbacks: `getJobById` throws `RuntimeException("Admin service unavailable")`; both searches return an empty list. | See USR-15 to USR-17. |
| USR-61 [DOC] | `GlobalExceptionHandler`: `EntityNotFoundException` gives 404; `IllegalArgumentException` gives 400; `Exception` gives 500. | |
| USR-62 [DOC] | `UserServiceTest`: `getUserById_found`, `getUserById_notFound`, `getAllUsers`, `searchBySkillSet`, `searchBySkillSet_noMatch`, `updateUser`, `updateUser_notFound`, `deleteUser`, `deleteUser_notFound`. `JobApplicationServiceTest`: `apply`, `getApplicationsByUser`, `getApplicationsByJob`, `deleteApplication`. | All pass. |
| USR-63 [ADDED] | Tests for `JobBasketService` (add, duplicate rejected with 409, remove, apply from basket), duplicate application gives 409, controller role checks, ownership 403 cases for every USER endpoint with a path or body id, and 401/403 on `GET /api/users` and `/search`. | All pass. |
| USR-64 [DECIDED OQ-1, OQ-5] | `GlobalExceptionHandler` adds `DuplicateResourceException` mapped to 409 (SEC-11). Duplicate application and duplicate basket item both throw it. | Given a duplicate, then 409 and `{"message": ...}`. |

---

## 11. Inter-service communication

| Caller | Callee | Endpoint | Purpose | On failure |
|---|---|---|---|---|
| auth-service | user-service | `POST /api/users/internal/create` | Create profile on seeker registration | Soft fail |
| auth-service | admin-service | `POST /api/admin/internal/create` | Create profile on employer registration | Soft fail |
| admin-service | auth-service | `DELETE /internal/auth/admin/{id}` | Delete credentials when an admin is deleted | Throws |
| admin-service | user-service | `GET /api/users/search`, `GET /api/users` | Search job seekers by skill | Empty list |
| user-service | admin-service | `GET /internal/jobs/...` | Fetch and search jobs | Empty list, or throws for single-job fetch |

| ID | Requirement | Acceptance |
|---|---|---|
| INT-01 [DOC][DECIDED OQ-2, OQ-4] | **Every** Feign call between services sends `X-Internal-Service-Key: ${INTERNAL_API_KEY}`, including admin-service to user-service searches and user-service to admin-service job lookups. | Given any callee, then it receives the header. |
| INT-02 [DOC] | Circuit breaker behaviour per CFG-07. On open: reads return an empty list; writes from auth-service soft-fail so registration still succeeds; admin-service to auth-service delete throws so the error is visible. | Given a callee that fails 5 of 10 calls, then the fallback is used instead of a timeout. |

---

## 12. Frontend (React 18 SPA)

Create React App project `online-job-portal-frontend`. Scripts: `start` (react-scripts start), `build` (react-scripts build). Dependencies as in 3.1.

```
src/
  api/        axiosConfig.js  authService.js  adminService.js  userService.js
  components/common/   Layout.jsx  Navbar.jsx  Sidebar.jsx  ProtectedRoute.jsx  Spinner.jsx
  context/    AuthContext.jsx
  pages/auth/   Login.jsx  Register.jsx  EmployerRegister.jsx
  pages/admin/  AdminDashboard.jsx  AdminJobs.jsx  AdminUsers.jsx
  pages/user/   UserDashboard.jsx  BrowseJobs.jsx  MyApplications.jsx  JobBasket.jsx  Messages.jsx  Profile.jsx
  styles/     global.css
  utils/      (empty)
  App.jsx  index.js
```

### 12.1 API layer

| ID | Requirement | Acceptance |
|---|---|---|
| FE-01 [DOC] | `axiosConfig.js` creates an Axios instance with base URL from `REACT_APP_API_URL`, defaulting to `http://localhost:8080` (the original hard-codes it; the variable is [ADDED]) and JSON content type. A request interceptor adds `Authorization: Bearer <token>` from `localStorage.token` when present. A response interceptor on **401** clears localStorage and sets `window.location.href = '/login'`. | Given a stored token, then every request carries the header. Given a 401 response, then storage is cleared and the user lands on /login. |
| FE-02 [DOC] | `authService.js` exports `login`, `registerUser`, `registerAdmin`, `createAdmin` mapped to `POST /api/auth/login`, `/api/auth/register`, `/api/auth/admin/register`, `/api/auth/admin/create-admin`. | Given a mock server, then each function calls its path. |
| FE-03 [DOC] | `adminService.js` exports: `getAllAdmins` (`GET /api/admin`), `getAdminById`, `updateAdmin`, `deleteAdmin` (`/api/admin/{id}`); `getAllJobs` (`GET /api/admin/jobs`), `getJobById`, `getMyJobs` (`/api/admin/jobs/my-jobs/{adminId}`), `createJob` (`POST /api/admin/jobs`), `updateJob`, `deleteJob`; `searchJobSeekersBySkill` (`GET /api/admin/seekers/search?skillSet=`); `sendMessageToJobSeeker` (`POST /api/admin/jobs/messages`), `getMessagesByJobSeeker` (`/api/admin/jobs/messages/from-jobseeker/{id}`), `getMessagesByEmployer` (`/api/admin/jobs/messages/by-employer/{id}`), `getMessagesReceivedByEmployer` (`GET /api/messages/employer/{id}`). | Each function calls the listed path and method. |
| FE-04 [DOC] | `userService.js` exports: `getUserById`, `getAllUsers`, `updateUser`, `deleteUser` (`/api/users...`); `applyForJob` (`POST /api/applications`), `getApplicationsByUser`, `getApplicationsByJob`, `deleteApplication`; `searchJobsBySkillDesignationCompany` (`GET /api/users/jobs/search` with `{skill, designation, company}`), `searchJobsByLocation` (`/api/users/jobs/search/location`), `getJobDetails` (`/api/users/jobs/{id}`), `getAllJobs` (`/api/admin/jobs`); `addToBasket` (`POST /api/basket`), `getBasket`, `removeFromBasket` (`DELETE /api/basket/{userId}/job/{jobId}`), `applyFromBasket` (`POST /api/basket/{userId}/apply/{jobId}`); `sendMessageToEmployer` (`POST /api/messages`), `getMessagesForEmployer`, `getMessagesByJobSeeker` (`/api/messages/jobseeker/{id}`), `getEmployerMessagesForSeeker` (`/api/admin/jobs/messages/from-jobseeker/{id}`). | Each function calls the listed path and method. |

### 12.2 Auth state and routing

| ID | Requirement | Acceptance |
|---|---|---|
| FE-10 [DOC] | `AuthContext` provides `user`, `loading`, `loginUser(token)`, `logoutUser()`, `isAdmin()`, `isUser()`. Token is decoded with `jwtDecode`; used fields `userId`, `sub` (username), `role`, `exp`. `parseToken` returns `{token, userId, username: sub, role, exp}`. | Given a token, then `user.username` equals `sub`. |
| FE-11 [DOC][DECIDED OQ-11] | On mount, read the token from localStorage, decode it, check expiry, set state, then set `loading=false`. `loginUser` stores the token, sets the user and returns the role. `logoutUser` removes the token and sets user to null. The whole app is wrapped in `<AuthProvider>`. While a user is logged in, a timer set from the token's `exp` shall call `logoutUser()` and redirect to `/login` when it expires. | Given an expired token in storage, when the app loads, then the user is not logged in. Given a token expiring in 5 seconds, then after 5 seconds the user lands on /login without making an API call. |
| FE-12 [DOC] | `ProtectedRoute({children, allowedRoles})`: while loading shows "Loading..."; no user redirects to `/login`; wrong role redirects to `/unauthorized`. `PublicRoute`: a logged-in user is redirected to `/admin/dashboard` (ADMIN) or `/user/dashboard` (USER). | A USER opening `/admin/jobs` lands on `/unauthorized`. A logged-in ADMIN opening `/login` lands on `/admin/dashboard`. |
| FE-13 [DOC] | Routes: `/` RoleRedirect; `/login`, `/register`, `/register-employer` (PublicRoute); `/unauthorized` (inline 403 message); ADMIN routes `/admin/dashboard`, `/admin/jobs`, `/admin/users`, `/admin/profile` (Profile); USER routes `/user/dashboard`, `/user/jobs` (BrowseJobs), `/user/applications`, `/user/basket`, `/user/messages`, `/user/profile`; `*` navigates to `/login`. Uses `BrowserRouter`. | Every listed path renders the named page for the right role. |
| FE-14 [DOC] | `ToastContainer` at top-right with `autoClose=3000`. | A toast disappears after 3 s. |

### 12.3 Shared components

| ID | Requirement | Acceptance |
|---|---|---|
| FE-20 [DOC] | `Layout` wraps pages: `<Navbar/>`, `<Sidebar/>`, `<main className="content">{children}</main>`. | |
| FE-21 [DOC] | `Navbar`: brand "JobPortal" with `MdWork` icon. ADMIN links: Dashboard, Jobs, Users. USER links: Dashboard, Browse Jobs, My Applications, Basket, Messages. Right side: profile icon link, username, logout button; logout calls `logoutUser()` then navigates to `/login`. | Clicking logout clears the token and shows /login. |
| FE-22 [DOC] | `Sidebar` (NavLink with active class). ADMIN: Dashboard, Manage Jobs, Users, Profile. USER: Dashboard, Browse Jobs, My Applications, Job Basket, Messages, Profile. | The current page's link has the active class. |
| FE-23 [DOC] | `Spinner`: `.spinner-wrapper` containing `.spinner`. | Shown while pages load data. |

### 12.4 Pages

| ID | Page | Requirement | Acceptance |
|---|---|---|---|
| FE-30 [DOC] | Login `/login` | State `{username, password}`, `loading`. Submit calls `login`, then `loginUser(res.data.token)`, then navigates by role. Error shows toast with the server message or "Invalid credentials". Links to `/register` and `/register-employer`. | Valid login goes to the role's dashboard; wrong password shows the toast. |
| FE-31 [DOC] | Register `/register` | Fields username*, password*, name*, email*, phone, address, skillSet. Calls `registerUser`, then goes to `/login`. Error toast. | Missing required field blocks submit. |
| FE-32 [DOC] | EmployerRegister `/register-employer` | Fields username*, password*, name*, email*, organizationName, address, contactNo. Calls `registerAdmin`, then `/login`. | |
| FE-33 [DOC] | AdminDashboard | Parallel `getAllJobs()` and `getAllUsers()`. Four stat cards: Total Jobs (blue), Total Users (green), Active Jobs (purple), Closed Jobs (orange). Active = status `ACTIVE`; Closed = status `CLOSED` (the status is an enum, DB-06; compare exactly). Table of first 5 jobs (title, company, location, status badge). | Counts match the data. |
| FE-34 [DOC] | AdminJobs | Tabs: All Jobs (fetch on mount; edit and delete), My Jobs (`getMyJobs(user.userId)` on tab switch), Search Seekers (skill input, `searchJobSeekersBySkill`, results table), Messages (`getMessagesByEmployer(userId)` and `getMessagesReceivedByEmployer(userId)`, two tables: sent by you, received from seekers). Job form fields title*, company*, location*, description*, experience, salary, noticePeriod, contactEmail, status (select ACTIVE/CLOSED/DRAFT). The modal creates (POST) or edits (PUT) by `editId`. Send Message modal: jobSeekerId (number), description, then `sendMessageToJobSeeker({description, jobSeekerId, employerId: user.userId})`. Delete asks `window.confirm`, then `deleteJob(id)`. Lazy-load My Jobs and Messages only when selected. | Create, edit, delete a job end to end; message a seeker and see it in the sent table. |
| FE-35 [DOC] | AdminUsers | Parallel `getAllUsers()` and `getAllAdmins()`. Tabs Users and Admins, each with name/email search filter and delete button. Create Admin modal (username, password, name, email) calls `createAdmin(form)` and switches to the Admins tab. Delete uses `deleteUser(id)` / `deleteAdmin(id)`. | Search filters rows; creating an admin adds a row. |
| FE-36 [DOC] | UserDashboard | Parallel `getAllJobs()`, `getApplicationsByUser(userId)`, `getBasket(userId)`. Four stat cards that are links: Available Jobs to `/user/jobs`, My Applications to `/user/applications`, Saved in Basket to `/user/basket`, Messages to `/user/messages`. Grid of the first 6 jobs. | Cards navigate correctly. |
| FE-37 [DOC] | BrowseJobs | On mount fetch jobs, applications and basket in parallel to pre-fill applied and saved state. Search tabs: All Jobs, Skills/Designation/Company (three inputs), Location (Enter triggers search). Job card shows title, company (`MdBusiness`), location (`MdLocationOn`), experience, salary, description. Save button (`MdBookmark`/`MdBookmarkBorder`) calls `addToBasket({userId, jobId})` and is disabled if already saved. Apply calls `applyForJob({userId, jobId})` and shows an "Applied" badge if already applied. | Applying shows the badge; saved jobs have a disabled save button. A 409 response (already applied or already saved) shows a toast such as "Already applied" and updates the badge or button instead of an error. [DECIDED OQ-1, OQ-5] |
| FE-38 [DOC] | MyApplications | `getApplicationsByUser(userId)`, then `getJobDetails(app.jobId)` per application. Table: Job Title, Company, Location, Experience, Salary, Applied Date, Status badge "Applied". If a job fetch fails, show "Job #jobId". | Failed job fetch shows the fallback text. |
| FE-39 [DOC] | JobBasket | Parallel `getBasket(userId)` and `getApplicationsByUser(userId)`. Table: Job ID, Saved On, Actions. Apply calls `applyFromBasket(userId, jobId)` and shows an "Applied" badge. Remove calls `removeFromBasket(userId, jobId)` and drops the row. A 409 from apply shows an "Already applied" toast and marks the row Applied. [DECIDED OQ-1] | |
| FE-40 [DOC] | Messages | Tabs Send, Sent, Received. Send form: employerId (number), jobId (number), description, then `sendMessageToEmployer({description, jobSeekerId: user.userId, employerId, jobId})`. Sent tab: `getMessagesByJobSeeker(userId)` (user-service `job_seeker_message`). Received tab: `getEmployerMessagesForSeeker(userId)` (admin-service `employer_message`). Fetch when the tab switches to Sent or Received. | A sent message appears in Sent; an employer's message appears in Received. |
| FE-41 [DOC] | Profile (`/user/profile` and `/admin/profile`) | On mount `isAdmin() ? getAdminById(userId) : getUserById(userId)`. Avatar (`MdPerson`), name, role badge. ADMIN form: name, email, organizationName, contactNo, address, saved with `updateAdmin`. USER form: name, email, phone, address, skillSet, saved with `updateUser`. | Saved changes persist after reload. |

### 12.5 Styling

| ID | Requirement | Acceptance |
|---|---|---|
| FE-50 [DOC] | `global.css` defines CSS variables `--primary #2563eb`, `--primary-dark #1d4ed8`, `--secondary #64748b`, `--success #16a34a`, `--warning #d97706`, `--danger #dc2626`, `--bg #f1f5f9`, `--surface #ffffff`, `--border #e2e8f0`, `--text #1e293b`, `--text-muted #64748b`, `--navbar-h 64px`, `--sidebar-w 240px`, `--radius 8px`, `--shadow 0 1px 3px rgba(0,0,0,0.1)`, `--shadow-md 0 4px 6px rgba(0,0,0,0.07)`. | Variables exist in `:root`. |
| FE-51 [DOC] | Classes: `.app-layout`, `.main-container`, `.content`, `.navbar`, `.navbar-brand`, `.sidebar`, `.sidebar-links a.active`, `.auth-page` (gradient `#1e3a8a` to `#2563eb`), `.auth-card` (max-width 420px, padding 40px, radius 16px), `.auth-card-wide` (640px), `.form-row` (2-column grid), `.btn-primary`, `.btn-secondary`, `.btn-icon` (32x32), `.btn-edit`, `.btn-danger`, `.btn-success`, `.stats-grid`, `.stat-card` with `.stat-blue/green/orange/purple`, `.data-table`, `.badge` with `.badge-approved/pending/applied`, `.filter-tabs`, `.tab` / `.tab.active`, `.modal-overlay` (z-index 200), `.modal` (max-width 520px), `.jobs-list`, `.job-list-card`, `.jobs-grid`, `.job-card`, `.profile-card` (max-width 720px), `.profile-avatar`, `.role-badge`, `.spinner-wrapper`, `.spinner` (40px, spin 0.8s), `.loading-screen`, `.empty-state`, `.empty-state-card`, `.search-box`. Layout: navbar fixed top 64px high with white background and shadow; sidebar 240px wide, sticky, border-right. | Each class exists and looks as described. |
| FE-52 [DOC] | At max-width 768px: sidebar hidden, `.form-row` single column, `.stats-grid` 2 columns, `.navbar-links` hidden, `.job-list-card` stacks vertically. | Resize below 768px shows the changes. |
| FE-53 [ADDED] | Frontend tests (Vitest/Jest and React Testing Library, as CRA supports Jest): `AuthContext` token parsing and expiry, `ProtectedRoute` redirects, `axiosConfig` interceptors. Kept small. | Tests pass. |

### 12.6 Implementation notes carried over [DOC]

1. `user.userId` from the decoded JWT is the id for all profile, application and basket calls.
2. admin-service principal is the `userId` (Long); user-service principal is the `username` (String).
3. Profile is one page used for both roles, switching on `isAdmin()`.
4. Messages are two separate tables: `job_seeker_message` (seeker to employer, user-service) and `employer_message` (employer to seeker, admin-service).

---

## 13. Cross-cutting and non-functional requirements

| ID | Requirement | Acceptance |
|---|---|---|
| NFR-01 [ADDED] | `mvn test` shall pass in each service with no MySQL or Eureka running (H2, `eureka.client.enabled=false`, `spring.cloud.discovery.enabled=false` in the test profile). | Run offline: all green. |
| NFR-02 [ADDED] | Each service README or the root README documents the run order, env vars and ports. | A new developer can run the system from the README alone. |
| NFR-03 [ADDED] | Each repo includes `docker-compose.yml` for MySQL 8 only, creating `jobportaldb`, `admin_db`, `user_service_db`, with the password from `.env`. | `docker compose up -d` provides the databases. |
| NFR-04 [ADDED] | A Postman collection covering every endpoint in sections 8 to 10 is committed. | Importing it and running in order passes. |
| NFR-05 [DOC] | Startup order and fallbacks allow each backend service to start without the others being up. | Starting user-service alone does not crash it. |

---

## 14. End-to-end acceptance scenarios

| ID | Scenario | Pass condition |
|---|---|---|
| E2E-01 | Employer registers and posts a job | Register via `/register-employer`, log in, post a job, it appears in All Jobs and My Jobs. |
| E2E-02 | Seeker registers, searches, applies | Register, log in, search by skill and by location, apply; "Applied" badge shows; application appears in My Applications with job details. |
| E2E-03 | Basket flow | Save a job, see it in Basket, apply from Basket, then remove another saved job. |
| E2E-04 | Messaging both ways | Seeker messages the employer; the employer sees it under Received. Employer messages the seeker; the seeker sees it under Received. |
| E2E-05 | Role protection | A USER cannot open `/admin/*` pages and gets 403 from admin-only APIs. An ADMIN cannot open `/user/*` pages. |
| E2E-06 | Resilience | Stop user-service, then register a seeker: auth returns 201 and the credential exists. Stop admin-service: seeker job search returns an empty list instead of an error. |
| E2E-07 | Token expiry | Remove or expire the token: the next API call returns 401 and the app redirects to `/login`. |
| E2E-08 | Duplicates | Apply to the same job twice (also via the basket): the second attempt returns 409 and the UI shows "Already applied"; the application list has one entry. |
| E2E-09 | Data privacy and ownership | Without a token, `GET /api/users` returns 401. As a USER, `GET /api/users` returns 403 and `GET /api/basket/<another user's id>` returns 403. As ADMIN 6, updating ADMIN 5's job returns 403. Through the gateway, any `/internal/` path returns 404. |

---

## 15. Traceability matrix (fill during Phase 4)

| Requirement ID | Implemented (yes/no/partial) | File and line | Test that verifies it | Result |
|---|---|---|---|---|
| (one row per ID in sections 2 to 13) | | | | |

---

## 16. Definition of done

- [ ] Every requirement has a row in the traceability matrix with no "no" or "partial".
- [ ] `mvn test` passes in all five services; frontend tests pass.
- [ ] All seven E2E scenarios pass manually.
- [ ] `git grep` for the original secrets and DB password returns nothing.
- [ ] README, `.env.example`, `docker-compose.yml` and Postman collection are committed.
- [ ] The author can explain every service, every security class and every Feign call.

---

## 17. Decisions on the open questions, and accepted limitations

Every item the original documents left open has been decided here. The agent shall follow the decision and shall not ask again.

| ID | Question in the original documents | **Decision** | Enforced by |
|---|---|---|---|
| OQ-1 | Applying twice to the same job was not prevented. | **Reject duplicates with 409**, backed by a unique constraint on (user, job). The UI shows "Already applied". | DB-05, USR-20, USR-33, USR-64, FE-37, FE-39, E2E-08 |
| OQ-2 | `GET /api/users` and `/search` were public, exposing every seeker's name, email, phone and address. | **No longer public.** ADMIN token or the internal service key only. admin-service sends the key on its Feign calls. | USR-01, USR-02, USR-11, USR-12, SEC-09, ADM-15, ADM-44, INT-01, E2E-09 |
| OQ-3 | Endpoints with a `userId` path or body only checked the role (any user could read another's basket or applications); any employer could edit any job. | **Ownership enforced.** A USER can only use their own id. An employer can only update or delete their own jobs, view their own jobs and send or read their own messages. | SEC-08, SEC-10, ADM-22 to ADM-27, USR-10, USR-13, USR-20 to USR-42, E2E-09 |
| OQ-4 | Internal endpoints were reachable through the gateway; `/internal/jobs/**` had no key check. | **Gateway returns 404 for any path containing `/internal/`**, and every internal endpoint validates the key. | GW-03, ADM-31, USR-60, INT-01, E2E-09 |
| OQ-5 | Status code for a duplicate basket item was not stated. | **409 Conflict.** | USR-30, USR-64, SEC-11 |
| OQ-6 | Validation failures would have returned 500. | **400** with per-field messages. | SEC-07 |
| OQ-7 | Job `status` was free text. | **Enum ACTIVE, CLOSED, DRAFT**, default ACTIVE, unknown value gives 400. | DB-06, ADM-48, FE-33 |
| OQ-8 | Gateway routes `/user/**` and `/admin/**` were unused. | **Removed.** | GW-01 |
| OQ-9 | Frontend hard-coded `http://localhost:8080`. | **`REACT_APP_API_URL`**, default the same. | FE-01 |
| OQ-10 | No CORS was documented. | **CORS at the gateway** for `http://localhost:3000`. | GW-04 |
| OQ-11 | An expired token was only noticed when an API call returned 401. | **Timer logs the user out at token expiry.** | FE-11 |

### Accepted limitations (not fixed in this version, know them for interviews)

| ID | Limitation | Why it is accepted |
|---|---|---|
| LIM-1 | The ADMIN role is both an employer and a platform admin: any ADMIN can list and delete users and other admins (`/api/admin/{id}`, AdminUsers page). | Matches the documented design. A cleaner model would split the roles into EMPLOYER and PLATFORM_ADMIN. |
| LIM-2 | The JWT is stored in `localStorage`, which an XSS flaw could read. | Documented design, and a normal trade-off for a learning project. HttpOnly cookies avoid it. |
| LIM-3 | All three services share one HMAC secret. Any of them could in theory mint a token. | Simple for a monolith-style learning project. Asymmetric keys (RS256) with a public key in the resource services avoid it. |
| LIM-4 | No rate limiting, no refresh tokens, no password reset. | Out of scope (section 1). |
| LIM-5 | A job seeker's skill search matches by simple text contains, not full-text search. | Matches the documented repositories. |

### Item summary

- **[ADDED]** items (can be rejected without breaking the original behaviour): CFG-01, CFG-02, GW-04, GW-06, SEC-06, SEC-07, AUTH-15, ADM-47, USR-63, EUR-03, NFR-01 to NFR-04, FE-01 (env variable), FE-53.
- **[DECIDED]** items (they change the original behaviour on purpose): DB-05, DB-06, SEC-08 to SEC-11, GW-01, GW-03, ADM-15, ADM-22 to ADM-27, ADM-31, ADM-44, ADM-48, USR-01, USR-02, USR-10 to USR-13, USR-20 to USR-21, USR-30 to USR-33, USR-40 to USR-42, USR-60, USR-64, INT-01, FE-11, FE-33, FE-37, FE-39.
