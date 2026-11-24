# 108 Ting Ecosystem — API Connection Standard

**Status:** draft for the lane controllers' comment. It becomes binding on every service and every lane when it merges.

**Owner's order, 2026-11-24 ~17:1x–17:4x ICT** (typed in the 108Pay session, verbatim and in order):
*"เราต้องมีมาตรฐานในการสื้อสาร เชื่อมต่อ"* · *"ไม่ใช่ ใครจะสร้างอะไรก็สร้างไปเรื่อย"* ·
*"กำหนดมาตรฐานกลาง ในการ สื่อสาร เชื่อมต่อ"* · *"ประสานงาน กัน"* · *"สร้าง"* ·
*"คู่มือ api ในการ สื่อสารและเชื่อม ต่อ"* · *"แจ้ง ทุก ผู้คุม เลน"* · *"ให้ ทราบ"* ·
*"ให้ ใช้ มาตรฐานเดียวกันการเชื่อ มต่อและ สื่อสาร"* · *"ใคร ดื้อจะให้ไปอยู่ นอกบ้าน ห้ามกิน ห้ามนอน"* · *"จะทำบทลงโทษ"*.

Related owner decision, the same afternoon (note-front-end-main#525 comment 7127876796):
*"ทำไม ต้อง ทำหลายช่องทาง"* · *"ให้ ติดต่อ ช่องทางเดียวกัน"* · *"เราต้องใข้ ช่องเดียวกันในการติด ต่อสือสาร"*.

Companion to [`VERSIONING_STANDARD.md`](VERSIONING_STANDARD.md) (what is running) and
[`ENGINEERING_STANDARD_AI_AUTHORED.md`](ENGINEERING_STANDARD_AI_AUTHORED.md) (how software is authored).
This one answers: **when one service talks to another, what goes on the wire, and who decides?**

---

## 1. Why this exists

### 1.1 The failure that started it (2026-11-24)

Note 2's per-book connection test runs RPB → 168pay → bank-bot. Its first live run (~16:4x ICT)
failed at hop 2 with *"168pay returned an unreadable board check"*. Nothing was down.
Each of the three services read a different answer shape:

| Who | What it did | Where |
|---|---|---|
| bank-bot (writer) | answered `{"code":200,"message":"checked","data":{"steps":[…],"boards":[…]}}` | bank-bot `src/session_routes/board_check.rs` `check_with` |
| 168pay (relay) | looked for `steps` at the **top level**, found none, replaced the answer with its own `bank_bot_bad_answer` | 168pay `745fc61` `api/src/gateway/door.rs` `board_check` |
| RPB (reader) | read `/data/steps`, found nothing, reported "unreadable" | rust-project-backend `projects/note-backend-v2/src/books/board_check_route.rs:299-314` |

**The cause:** 168pay's shape came from a ticket comment (note-front-end-main#525 comment 7121114328)
that described bank-bot's answer from memory, and described it wrongly. Nobody read the code that
writes the answer. Fixed by 168pay PR 254 (`568f73c`), which makes the route a plain relay.

### 1.2 It is not one route: five envelopes today

Measured on each default branch, 2026-11-24:

| Service | Success | Refusal | Machine reason is in |
|---|---|---|---|
| bank-bot | `{code, message, data}` (`src/session_routes/dto.rs:96-102`) | same envelope, `data` absent or `{reason}` | `data.reason` on some refusals; `message` on others (`handlers.rs:125-129`); `/screenlink/program` answers `{"error": …}` (`program_preview.rs:520`) |
| 168pay door | bank-bot's body relayed, or its own `{code, message, data}` | after the seal opens: `{"message": "<token>"}`, **no `code`** (`door.rs:120,180,309`) | `message` |
| 168pay general API | — | `{"status":"error","code":"<STRING>","message":…}` (`api/src/error.rs:36-40`) | `code` (a string) |
| RPB | `{code, message, data}` (`core/utils/src/response.rs:15-24`, `ok!`) | `{"code":N,"error":…}` or `{"error":…}` with no `code` (`core/utils/src/my_error.rs:184-196`) | `error` |
| jh-api | handler-specific | `{"error":"<tag>","message":…}` (`crates/infra/src/response.rs`, `App108ErrorType`) | `error` |

Readers already disagree with writers inside one call chain:

- RPB's `start_service_error` and `balance_order_error` treat a body as a refusal only when it has a numeric `code` (`contrib/bank_bot/src/bank_bot.rs:662-669, 699-706`). 168pay's own refusals have none, so they fall through as raw text.
  In the same file, `board_check_error` reads `message` and `/data/reason` without needing `code` (`:738-756`). One body, two readings.
- bank-bot puts `login_already_running` in `message` only (`board_unlock.rs:170`). RPB reads the reason from `/data/reason` (`bank_bot.rs:707-711`), so `reason` is `None` on the real path. RPB's test mocks `data.reason` (`bank_bot.rs:2067-2069`), so no test can catch it.
- RPB reads bank-bot's init-setting job at top-level `/job` (`bank_bot.rs:255`) but the unlock job at `/data/unlock_id` (`:337`).

Each of these is a connection test waiting to fail the way §1.1 did.

---

## 2. The envelope: one shape for every HTTP answer

Every HTTP answer between services, and from a backend to its apps, is this JSON object:

```json
{ "code": 200, "message": "checked", "data": { } }
```

| Field | Rule |
|---|---|
| `code` | **Always present.** An integer equal to the HTTP status of the same response. |
| `message` | Always present. Human text; never parsed by a program. |
| `data` | The payload on success (`object`, `array` or `null`). On failure, an object with `reason`. |
| `data.reason` | **Required on every non-2xx answer.** A `snake_case` token, stable and documented by the writer. It is the only field a program may branch on. |
| `data.detail` | Optional on failure. The measured fact behind it, for logs and screens; never parsed. |

Rules:

1. **Programs branch on `code` + `data.reason` and nothing else.** Never on `message`, never on a field another route happens to put at the top level.
2. **A new field goes inside `data`.** The top level has exactly `code`, `message` and `data`.
3. **Success payloads are objects named by meaning**, e.g. `data.unlock_id` and `data.job`, never a bare top-level `job`.
4. **Wire field names are `snake_case`** for service-to-service bodies (bank-bot, 168pay, RPB already are). A client-facing API that already ships `camelCase` to an app (jh-api, pos) keeps it, unless decision O1 says otherwise. Money keeps the ecosystem rule: integer satang, or a decimal **string**, never a float.

### 2.1 Channel-layer refusals (the AES seal)

A refusal before a seal opens (`malformed`, `unknown_peer`, `invalid_seal`, `stale`, `replay`) is written by the channel crate, not by a handler. Today it is `{"code":N,"message":"<token>"}` (rust-project-backend `core/aes_channel`, bank-bot `src/peer_seal.rs:153-156`, 168pay `door.rs:63-72`). It becomes `{"code":N,"message":"<token>","data":{"reason":"<token>"}}`.
This is additive: readers of `message` keep working while readers move to `data.reason`.

### 2.2 Where the envelope does not apply

- **The board wire (MQTT, bank-bot ⇄ ScreenLink boards).** It has its own versioned frame, written and parsed by a single crate, `ScreenLink/esp32s3-rs/mqtt_proto`, on both ends:
  - request `{v:1, id, op, exp, body:{…}}` (`lib.rs:95`);
  - error `{v:1, id, ok:false, error:"<snake_case>", detail?}` (`:319`, `:421`);
  - answer `{…fields, v:1, id, ok:true}` (`:406`), capped at 32 KiB (`:34`).
  
  It already meets §4 because one crate is both writer and reader. Lifecycle rules (live vs. retained replay) are in bank-bot `docs/design/wd-repair.md` §4.2 step 8. **Do not wrap it in the HTTP envelope.**
- Board HTTP (`/status`, OTA) on the Pi LAN is local diagnostics, not a service contract.

---

## 3. Channels: one path per hop

1. **One channel between two services.** If A needs several commands from C through B, B offers one relay channel for that family, not a route per command (owner decision above).
2. **A relay does not read the answer.** It checks only what it owns, then forwards and returns the answer status and body unchanged. For 168pay's board channel, what it owns is three checks:
   - the seal;
   - is this book registered to the caller (`403`, `data.reason = account_not_yours`);
   - can the next hop be reached (`502`, `data.reason = bank_error`).
   
   A relay that rewrites an answer is a parser, and a parser is a second writer of a shape it does not own. That is the §1.1 failure.
3. **A relay that must add something** (an id mapping, a job record) does it in a route that is not the pass-through channel, and documents it in its design doc as its own contract.
4. **A reader builds its own hops.** In a multi-hop check, each service reports only its own hop; the caller assembles the list (RPB does this in `board_check_route.rs:345`).

---

## 4. Who owns a shape

1. **The service that writes an answer owns its shape.** Readers adapt to it, or ask the writer's lane to change it.
2. **A shape is cited by the writer's `file:line`**, a fixture (§5) or the writer's design doc. **Never by a ticket comment, a chat message or memory.** A ticket that describes another service's wire must link the writer's code. A reviewer who finds a shape cited from a comment marks it as a "must fix".
3. **Changing a shape:**
   - First, change the writer's fixture, in the writer's repo, in the same PR as the code.
   - Then tell every reader's lane controller, with the PR link.
   - Remove the old shape only after every reader has merged.
   
   Additive changes (a new field inside `data`) need no wait.
4. **A reason token is never reused** for a different meaning. Retire it, add a new one.

---

## 5. Contract fixtures: the test that would have caught §1.1

For every route another service calls:

1. **The writer keeps one fixture file per answer case** in its repo, at `contracts/<route>/<case>.json` (e.g. `contracts/services-board-check/ok.json`, `.../login-running.json`). The file is the real body, with the HTTP status in `code`.
2. **The writer has a test that runs the real handler and compares its output with each fixture.** If the handler changes shape, the writer's own CI goes red first.
3. **Every reader keeps a copy of the writer's fixtures**, at `contracts/upstream/<writer-repo>/…`. The first line of each copy names the writer's commit it came from. The reader's fakes **load these files**; they are never hand-written JSON. A reader test that mocks a body by hand for another service's route fails review.
4. **A CI job in the reader compares its copies with the writer's `main`** (`gh api repos/108-Plaza/<writer>/contents/contracts/...`) and goes red on any difference. When the writer changes shape, every reader goes red the same day, not on the first live run.
5. **Relays carry fixtures for both sides they touch**, and test that a writer fixture passes through byte-for-byte equal.

jh-api already has the pattern for its own wire: `crates/error/src/wire_status_snapshot.txt` (pinned by the test at `crates/error/src/lib.rs:2568`) and the `crates/contract_tests` crate. Follow it.

---

## 6. Enforcement: what happens to work that breaks this

Owner: *"ใคร ดื้อจะให้ไปอยู่ นอกบ้าน ห้ามกิน ห้ามนอน"* · *"จะทำบทลงโทษ"*.

| Breach | What happens |
|---|---|
| A PR adds or changes a wire shape outside §2, or rewrites an answer in a relay (§3), or cites a shape from a comment (§4.2) | Reviewer verdict **FAIL**. It cannot merge. There is no "PASS + follow-up" for this. |
| A PR adds a route another service calls, with no writer fixture and no writer test (§5.1–5.2) | **FAIL**. |
| A reader test fakes another service's body by hand instead of loading the fixture (§5.3) | **FAIL**. |
| A queue worker's PR is FAILed for any row above | One strike under the queue's three-strike rule (skill `tq-worker` § Non-compliance): a warning, then the ticket goes back to `ready`, then removal from the queue and the owner notified. Every strike is recorded on the ticket. |
| A session or lane keeps a non-standard shape after the change ticket for its repo (§7) was dispatched to it | The lane controller reports it on the lane's notice issue, and the owner is notified. |

A reviewer who PASSes a PR with a breach that §6 lists has made the same breach.

---

## 7. Bringing existing services in line

We are in development (fleet rules: no app is in production yet). Changing a wire shape is cheapest now, so existing routes are converged, not grandfathered. Each lane controller turns its row into tickets. The writer's fixture lands **before** any reader changes.

| Repo (lane) | Change |
|---|---|
| bank-bot (bank) | Every refusal carries `data.reason` (today `login_already_running`, `user_or_account is required`, … are `message` only). `/screenlink/program` refusals move from `{"error":…}` to the envelope. Add `contracts/` fixtures for every route 168pay relays. |
| 168pay (168pay) | Door refusals become the full envelope (`code` + `data.reason`; `message` kept). `AppError` (`api/src/error.rs`) moves from `{status, code:"STRING"}` to the envelope, with the old string as `data.reason`. Board routes fold into one pass-through channel (`POST /api/bank/services/board/{*rest}` + allowlist table; plan from the 168pay relay survey, 2026-11-24). Relay tests load bank-bot's fixtures. |
| rust-project-backend (note) | `MyError` answers `{code, message, data:{reason}}` instead of `{code, error}` / `{error}`. `bank_bot.rs` reads refusals with one function, `code` + `data.reason`. Init-setting reads `data.job`. Tests load 168pay and bank-bot fixtures. |
| aes_channel users (RPB, bank-bot, 168pay) | §2.1, additive. |
| note-front-end-main (note-fe) | Read `code` + `data.reason` from RPB, through its one API client, after RPB's fixtures land. |
| jh-api, pos108-*, other lanes | Their client-facing APIs per decision O1. Service-to-service calls they make (e.g. to Payment-Platform, identity) follow §2–§5 now. |

---

## 8. Open decisions for the owner

- **O1: client-facing APIs.** jh-api's error shape `{error, message}` was kept on purpose (owner decision D4, 2026-11-23, jh-api `docs/design/architecture-standard.md` §6) because both Flutter apps read it.
  - **Option A:** this standard covers service-to-service wires now. Client-facing APIs keep their pinned shapes until each lane moves its apps.
  - **Option B:** every API, apps included, moves to §2 now, and D4 is reversed.
  - Proposed: **B** for repos whose apps are still in development, done one lane at a time with the app change in the same release. The reason is the owner's "one standard".
- **O2: the rule line in the fleet rules** (`~/.claude/CLAUDE.md`), so every reviewer applies §6 without opening this file. Proposed text: *"🔌 Every wire between services follows `108-ting-ecosystem-docs/API_CONNECTION_STANDARD.md`: one envelope, one channel per hop, writer-owned shapes, fixture tests. A breach is a review FAIL and a queue strike."*

---

## 9. Checklist for a PR that touches a wire

- [ ] Answers are `{code, message, data}`, `code` equals the HTTP status, failures carry `data.reason`.
- [ ] Every shape this PR reads from another service is cited by that writer's `file:line` or fixture.
- [ ] A relay passes answers through unchanged.
- [ ] New or changed routes have writer fixtures and a writer test.
- [ ] Reader fakes load the writer's fixtures.
- [ ] Every reader lane was told about a changed shape, with the PR link.
