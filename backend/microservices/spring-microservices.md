# Spring Boot Microservices — Architecture Guide

Based on a real stack: Spring Boot services + Eureka discovery + Keycloak auth + Angular front + Docker.

## Reference architecture

```
                        ┌──────────────┐
        browser ───────►│  Frontend    │ (Angular + nginx)
                        └──────┬───────┘
                               │ /api/*
                        ┌──────▼───────┐        ┌─────────────┐
                        │  API Gateway │◄──────►│  Keycloak   │ (OIDC)
                        └──────┬───────┘        └─────────────┘
                 ┌─────────────┼─────────────┐
          ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼───────┐
          │ notes-svc  │ │ users-svc  │ │  ...-svc   │   each: own DB
          └──────┬─────┘ └─────┬──────┘ └────┬───────┘
                 └─────────────┼─────────────┘
                        ┌──────▼───────┐
                        │    Eureka    │ (service discovery)
                        └──────────────┘
```

## Service discovery with Eureka

```yaml
# each service's application.yml
eureka:
  client:
    service-url:
      defaultZone: http://discovery-server:8761/eureka
  instance:
    prefer-ip-address: true
```
- Services register themselves; the gateway/clients resolve by **service name**, not host:port.
- In Docker/k8s, healthchecks matter: a registered-but-dead instance poisons routing.
- Kubernetes note: k8s has native discovery (Services/DNS) — Eureka is mostly for non-k8s or mixed estates.

## Security with Keycloak (OIDC)

The right mental model:
1. **Frontend** redirects to Keycloak → user logs in → front gets tokens (Authorization Code + PKCE flow).
2. Front calls APIs with `Authorization: Bearer <access_token>`.
3. **Every service validates the JWT** (signature via Keycloak's JWKS, `iss`, `aud`, expiry) — not just the gateway.

```yaml
# resource server config (each service)
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://keycloak:8080/realms/notes-realm
```

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
  @Bean
  SecurityFilterChain filter(HttpSecurity http) throws Exception {
    return http
      .csrf(csrf -> csrf.disable())              // stateless API, token-based
      .authorizeHttpRequests(a -> a
        .requestMatchers("/actuator/health/**").permitAll()
        .anyRequest().authenticated())
      .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
      .build();
  }
}
```

Common traps:
- `issuer-uri` must match what's inside the token — watch docker hostname vs localhost mismatches.
- Roles live in `realm_access.roles` → you need a converter to map them to `ROLE_*` authorities.
- Never accept tokens without verifying `aud`/authorized party in multi-client realms.

## Resilience — because the network WILL fail

```java
@Retry(name = "users")
@CircuitBreaker(name = "users", fallbackMethod = "usersDown")
public UserDto getUser(String id) { ... }

List<UserDto> usersDown(String id, Throwable t) {
  return cachedOrDefault(id);       // degraded, not dead
}
```
(resilience4j) Rules of thumb:
- **Timeouts on every remote call** — the default infinite timeout is an outage amplifier.
- Circuit breaker on dependencies that can flap; fallback = degraded answer, not exception.
- Retries only on idempotent operations, with backoff + jitter.

## Data: one database per service

- Sharing a DB between services couples them harder than a monolith — worst of both worlds.
- Cross-service consistency: prefer **events** (outbox pattern) over distributed transactions.
- Need data from another service? Call its API or subscribe to its events. Never its tables.

## Should this even be a microservice?

Honest checklist before splitting:
- [ ] Independent scaling actually needed?
- [ ] Independent deploy cadence actually needed?
- [ ] Team boundaries match service boundaries?
- [ ] You have the ops maturity (CI/CD, observability, on-call)?

A well-modularized monolith ("modulith") is the right answer more often than conference talks admit. Microservices trade code complexity for operational complexity — make sure you're buying something.
