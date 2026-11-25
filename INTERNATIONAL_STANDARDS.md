# 108 Ting Ecosystem — International Standards Map

**Status:** APPROVED, 2026-11-25 (owner: *"รับ S1–S7 ทั้งหมด"*, in session claude/12f0f112). Proposed the same day. Written by claude/12f0f112 (JH-API) on the owner's goal:
*"สร้างทุกอย่างให้ เป็น มาตรฐานเดียว แล้ว เป็นมาตรฐาน สากล ระดับโลก"* (make everything one standard, and make it an international, world-class one).
Decisions S1–S7 (§3) were approved by the owner as recommended. Everything else maps what already exists.

## 1. The rule

**Every house standard in this repo names the international standard it implements, and differs from it only where it says why.**
A house standard exists only to choose among the options an international standard leaves open, or to cover what no international standard covers.
It never re-invents something a published standard already settles.

Why:
- Agents already know RFCs, W3C and OpenTelemetry, and so do the tools (proxies, log stores, client libraries, scanners).
- A shape that follows a published standard is read correctly by things we did not write.
- A house-only shape has to be taught to every new agent and every new tool, and it drifts. Five different envelopes in the bank chain alone are the example (`API_CONNECTION_STANDARD.md` §1.2).

## 2. The map

Status:
- **Adopted** means the house standard follows the international one.
- **Partial** means it follows it with a gap (named).
- **Gap** means it doesn't follow it yet.

| Area | International standard | House standard (this repo) | Status | Gap or action |
|---|---|---|---|---|
| HTTP methods and status codes | **RFC 9110** HTTP Semantics | `API_CONNECTION_STANDARD.md` §2: `code` equals the HTTP status; success is any 2xx | Adopted | — |
| Rate limiting | **RFC 6585** §4 `429 Too Many Requests` + RFC 9110 `Retry-After` (§10.2.3); IETF draft *RateLimit header fields* | none | Gap | Every 429 sends `Retry-After`. Adopt the RateLimit headers when the draft becomes an RFC |
| Error body | **RFC 9457** Problem Details for HTTP APIs | `API_CONNECTION_STANDARD.md` §2: `{code, message, data:{reason, detail?}}` | Partial | One envelope covers success and error, which RFC 9457 does not. Decision **S1** |
| JSON | **RFC 8259** | `API_CONNECTION_STANDARD.md` §2 rule 6: snake_case between services; client APIs keep the casing their apps read | Adopted | — |
| Time | **RFC 3339** (a profile of ISO 8601), always with an offset | log108 `ts`; wire timestamps | Adopted | Wire timestamps carry the offset, never a naive local time |
| Identifiers | **RFC 9562** UUIDs | UUIDs for request ids and installations | Adopted | Prefer UUIDv7 for new keys, which sort by time |
| Money | **ISO 4217** currency codes, amounts in minor units | integer satang, or a decimal **string** (fleet rule) | Partial | Where an amount crosses a wire, its currency travels with it (`THB`) |
| Language | **BCP 47** (RFC 5646) tags, `Accept-Language` | apps send `Accept-Language`; `th`, `en`, `vi` | Adopted | — |
| Request correlation | **W3C Trace Context** (`traceparent`, `tracestate`) | `x-request-id` (log108 LOG-1) | Partial | Decision **S2** |
| Logs | **OpenTelemetry** Logs Data Model and Semantic Conventions | `108-logging` README: `ts`, `level`, `service`, `request_id`, `target`, `message` | Partial | No error fields yet (status, reason, route). Decision **S3** |
| API description | **OpenAPI 3.1** | writer fixtures in `contracts/` (`API_CONNECTION_STANDARD.md` §5); jh-api ships ts-rs types, and `jh-client` builds a spec from a hand-written controller | Gap | Decision **S4** |
| Idempotent retries | IETF draft *The Idempotency-Key HTTP Header Field* | jh-api `IdempotencyMiddleware` reads `Idempotency-Key` | Adopted (draft) | Its duplicate answer must become an enveloped `409`/`422` with `data.reason`; today it is an empty 422 |
| Authentication | **OAuth 2.0** (RFC 6749) / OAuth 2.1 draft, **OpenID Connect Core 1.0**, **JWT** (RFC 7519), EdDSA (RFC 8037) | identity issues EdDSA JWTs with `iss` and per-service `aud` | Adopted | — |
| Versioning | **SemVer 2.0.0** | `VERSIONING_STANDARD.md` (`1.MINOR.PATCH`) | Adopted | — |
| Commits | **Conventional Commits 1.0.0** | used across repos | Adopted | — |
| Application security | **OWASP ASVS 5.0** (level 2); **OWASP API Security Top 10 (2023)** | fleet review checklists; per-repo audits | Gap | Decision **S5** |
| Accessibility (apps and web) | **WCAG 2.2** level AA | none | Gap | Decision **S6** |
| Supply chain | **SLSA v1.0** build levels; SBOM in **CycloneDX** or **SPDX** | `cargo deny` where `deny.toml` exists; self-hosted CI | Partial | Generate an SBOM per release image. Record SLSA build level 2 provenance (signed, from CI) |
| Containers and deploy | **OCI** image spec; Helm; Kubernetes probes | `DEPLOYMENT_STANDARD_K3S.md` | Adopted | — |

## 2a. Quality targets: stable, fast, reliable, easy to maintain

Owner, 2026-11-25: *"ความ เสถียร์ ต้องได้ ความเร็ว ความน่าเชื่อถือ ความง่ายในการดูแล แก้ไข"* (it must be stable, fast and reliable, and easy to look after and fix) · *"ง่ายในที่นี้คือ ai ทำงานร่วมกับคน"* ("easy" here means AI working together with people).
The system is run by AI agents supervised by at most two people (`ENGINEERING_STANDARD_AI_AUTHORED.md`). So "easy to maintain" is measured by how far an agent gets on its own, and how small the human step is.

The international model for these qualities is **ISO/IEC 25010** (software product quality). They are measured the way SRE practice does (**SLIs and SLOs** with error budgets), and delivery health is measured with the **DORA** four keys.

| Owner's word | ISO/IEC 25010 characteristic | How it is measured (SLI) | Proposed target (SLO) |
|---|---|---|---|
| เสถียร (stable) | Reliability: faultlessness ("maturity" before the 2023 edition), availability | share of requests answered without a 5xx, per service, over 30 days | ≥ 99.9% per client-facing API (decision **S7**) |
| เร็ว (fast) | Performance efficiency: time behaviour | p95 server latency per route template (`http.route`), from the log or metric line | p95 ≤ 300 ms for reads and ≤ 800 ms for writes, outside calls to third parties (**S7**) |
| น่าเชื่อถือ (reliable) | Reliability: fault tolerance, recoverability | correct results under retry (`Idempotency-Key`), and recovery time after a failure (DORA *time to restore*) | restore in < 1 hour; zero duplicate money movements |
| ดูแลง่าย แก้ไขง่าย (easy to maintain and fix), meaning **AI and people working together** | Maintainability: analysability, modifiability, testability, reusability (ISO/IEC 25010) | (1) **an agent can go from a symptom to the cause on its own**: the screen's reason equals the `data.reason` the app reads, which equals the log's `error.type`, which leads to one place in the code; (2) every decision has its reason written where the next session reads it; (3) DORA *change lead time* and *change failure rate*; (4) the `software-design` copy count and the architecture ratchet | a fault is diagnosed by an agent from one token, with no human digging; a person's part is one decision or one approval, never investigation; change failure rate < 15%; copy counts only go down |

Each target becomes a CI gate or a dashboard, never a sentence only:
- **Latency and error SLIs** come from the S3 log lines (one per answer, with `http.route` and `http.response.status_code`) in VictoriaLogs.
- **DORA** comes from GitHub (merge time and deploy runs) and from incident tickets.
- **Maintainability** comes from the gates that already exist: the architecture ratchet (jh-api G1–G3) and the `software-design` review FAILs.

## 3. Decisions (approved 2026-11-25)

The owner accepted every recommendation below: *"รับ S1–S7 ทั้งหมด"*. Each lane turns its Partial/Gap rows into tickets (§4).

| Decision | Owned by (who changes what) |
|---|---|
| S1 envelope ↔ RFC 9457 mapping, reason URIs | `API_CONNECTION_STANDARD.md` (108Pay lane) adds the mapping table; each writer publishes its reason registry |
| S2 W3C Trace Context | `log108` (108-logging lane) and every HTTP service's middleware |
| S3 OpenTelemetry log field names, `error.type` = `data.reason` | `log108` (108-logging lane), once, for every service |
| S4 OpenAPI 3.1 from writer code | each HTTP service lane; jh-api first, generating jh-client from it |
| S5 OWASP ASVS 5.0 L2 + API Top 10 | the fleet review checklist (`review-and-merge`) |
| S6 WCAG 2.2 AA | the app and web lanes |
| S7 SLOs per §2a | each lane publishes SLIs from the S3 log lines, jh-api and pos108-core first |

**S1 — error body: keep `{code, message, data}` (owner-approved O1), mapped exactly onto RFC 9457.**
- **Option A, recommended:** keep the approved envelope. It is the only shape that covers success and error alike, and RFC 9457 defines errors only. Publish the exact mapping and register every reason token at a stable URL, which plays the role of the RFC's `type`:

  | RFC 9457 member | Envelope |
  |---|---|
  | `status` | `code` |
  | `title` | `message` |
  | `type` | `data.reason` (token); `https://errors.108plaza.net/<reason>` as its URI |
  | `detail` | `data.detail` |
  | `instance` | the `x-request-id` response header, carrying the S2 trace id (`traceparent` is a request header; the response-side `traceresponse` is only a W3C Trace Context Level 2 draft) |

- **Option B:** switch error bodies to `application/problem+json`. Success then keeps the envelope, so there would be two shapes. The owner's rule is one standard, so this is not recommended.

**S2 — correlation: adopt W3C Trace Context.**
- Every service accepts and propagates `traceparent`, and starts one when none arrives.
- `x-request-id` stays as an alias carrying the same trace id, for tools that read it.
- Log lines carry `trace_id` and `span_id`.

**S3 — logs: log108 adopts OpenTelemetry field names for HTTP and error facts.**
Existing fields keep their meaning:

| Existing field | OpenTelemetry equivalent |
|---|---|
| `level` | `severity_text` |
| `service` | `service.name` |
| `request_id` | `trace_id` |

The fields an error line must carry:
- `http.request.method`
- `http.route` (the route template, never the raw path with ids)
- `http.response.status_code`
- `error.type`: **the same token as `data.reason`**

The point is that the reason a person sees on screen, the `data.reason` an app reads, and the field a log query filters on are **one token**.
The server writes exactly one log line per non-2xx answer, at the single place that builds the envelope:
- `error` for 5xx;
- `warn` for 4xx that point at a fault (409 conflicts, 422);
- `info` for expected refusals (401, 403, 404).

Nothing sensitive goes into that line (log108 redaction).

**S4 — contracts: OpenAPI 3.1, generated from the writer's code.**
- Each HTTP service publishes an OpenAPI 3.1 document generated from its handlers (Rust: `utoipa`).
- The `contracts/` fixtures validate against it.
- Clients generate their types from it instead of copying them by hand.
- Measured 2026-11-25: jh-client exists in 3 diverging copies (`heros/jh-client`, and vendored copies in `108heros-web` and `108jobs-web`).

**S5 — security baseline: OWASP ASVS 5.0 level 2, plus the API Security Top 10 (2023), in every review checklist.**
A PR that touches auth, money, files or input parsing names the ASVS requirements it meets.

**S7 — quality targets (§2a): adopt the SLOs in §2a, starting with jh-api and pos108-core.**
Each lane publishes its SLIs from the S3 log lines before the targets are enforced. A breach of an error budget pauses feature merges in that lane until a fix lands (SRE error-budget policy).

**S6 — accessibility: WCAG 2.2 AA for every app and web screen.**
- Contrast, text scaling, screen-reader labels, and touch-target size.
- Checked by the app lanes' widget tests and web lint (axe).

## 4. How this is enforced

- A house standard that maps to an international one cites it in its first section.
- A review that finds a new house-only shape where a published standard applies is a **FAIL** (the same rule as `API_CONNECTION_STANDARD.md` §6).
- Lanes adopt in their own order. Each lane controller turns the rows marked Partial or Gap into tickets; the owner approved S1–S7 on 2026-11-25.

## 5. Sources

Each source is named by number and title, so it can be looked up:
- RFC 9110 (HTTP Semantics, 2022)
- RFC 6585 (Additional HTTP Status Codes, 2012: `429`)
- RFC 9457 (Problem Details for HTTP APIs, 2023)
- RFC 8259 (JSON)
- RFC 3339 (Date and Time on the Internet)
- RFC 9562 (UUIDs, 2024)
- RFC 5646 (BCP 47)
- RFC 6749 (OAuth 2.0)
- RFC 7519 (JWT)
- RFC 8037 (EdDSA for JOSE)
- W3C Trace Context (Recommendation)
- OpenTelemetry Logs Data Model and Semantic Conventions
- OpenAPI Specification 3.1
- SemVer 2.0.0
- Conventional Commits 1.0.0
- OWASP ASVS 5.0
- OWASP API Security Top 10 2023
- WCAG 2.2
- SLSA v1.0
- CycloneDX and SPDX
- IETF HTTPAPI drafts on `Idempotency-Key` and RateLimit header fields
- ISO/IEC 25010:2023 (Systems and software Quality Requirements and Evaluation, SQuaRE: product quality model)
- Google SRE practice (SLIs, SLOs and error budgets)
- DORA four key metrics
