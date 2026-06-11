# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build (skip tests)
./mvnw clean package -DskipTests

# Run
java -jar target/Security6-0.0.1-SNAPSHOT.jar

# Run tests
./mvnw test

# Run a single test class
./mvnw test -Dtest=Security6ApplicationTests
```

Requires MySQL running at `localhost:3306` with a database named `demo` (credentials in `application.properties`). Hibernate is set to `ddl-auto=update` so the schema is managed automatically.

## Architecture

This is a **stateless JWT REST API** built with Spring Boot 3.4 + Spring Security 6 + MySQL.

### Auth flow

```
POST /register  →  UseController  →  UserService.register()
                                      BCrypt-encodes password, saves User entity

POST /login     →  UseController  →  UserService.verify()
                                      AuthenticationManager.authenticate()
                                        → CustomUserDetailsService.loadUserByUsername()
                                        → DaoAuthenticationProvider (BCrypt check)
                                      JwtService.generate(principal.name)
                                      returns LoginResponse { token, username, expiresIn }

GET /welcome    →  JwtAuthenticationFilter (reads Authorization: Bearer <token>)
GET /me              → JwtService.extractUsername() + validateToken()
                       → SecurityContextHolder populated
                     → controller proceeds
```

### Key wiring decisions

- **`JwtAuthenticationFilter`** runs before `UsernamePasswordAuthenticationFilter`. If no `Authorization` header is present it passes through silently; invalid/expired tokens are swallowed and the request proceeds unauthenticated (Spring Security returns 403).
- **`DaoAuthenticationProvider`** is explicitly wired via `.authenticationProvider()` in `WebSecurity.securityFilterChain()` — do not remove this; without it Spring may fall back to its auto-configured provider.
- **`JwtService.signingKey`** is built once at startup via `@PostConstruct` from the Base64-encoded `jwt.secret` property. The secret must be at least 256 bits after decoding for HS256.
- **`CustomDetails`** wraps the `User` entity to satisfy the `UserDetails` contract. It returns an empty authorities list — there is no role-based access control yet.
- Session management is `STATELESS` — no `HttpSession` is created. CSRF is disabled.

### Public vs protected endpoints

| Endpoint | Auth required |
|----------|--------------|
| `POST /register` | No |
| `POST /login` | No |
| `GET /welcome` | Yes (JWT) |
| `GET /me` | Yes (JWT) |
| `GET /csrf` | Yes (JWT) |

### Token expiry

Tokens expire after **10 hours** (`TOKEN_EXPIRY_MS` in `UserService`). There is no refresh-token mechanism.

## Database

The `User` table has `id` (AUTO_INCREMENT), `user_name`, `password` (BCrypt, strength 14). The `id` field uses `@GeneratedValue(strategy = IDENTITY)` — never supply an `id` in registration requests.

## JWT secret rotation

To rotate the secret, replace `jwt.secret` in `application.properties` with a new Base64-encoded HMAC-SHA256 key (≥32 bytes before encoding). All existing tokens are immediately invalidated.