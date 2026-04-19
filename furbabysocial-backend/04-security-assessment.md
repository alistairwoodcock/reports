# FurBaby Social Backend - Security Assessment

**Date:** 2026-04-19

---

## Severity Summary

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High | 5 |
| Medium | 4 |
| Low | 3 |
| Informational | 3 |

---

## Findings

### CRITICAL

#### SEC-01: Production Database Publicly Reachable from the Internet

**Location:** AWS RDS instance `production-db.cmicgh6krkni.ap-southeast-2.rds.amazonaws.com`, port 5432

**Description:** The production Postgres database is directly reachable from the public internet. Verified by connecting from a residential connection using only the hostname, port, username, and password — no VPN, no bastion, no IP allowlisting step. This means at least one of the following is true:

1. `PubliclyAccessible = true` on the RDS instance (should be `false` for prod).
2. The RDS security group ingress allows `0.0.0.0/0` on port 5432 (or a very broad CIDR).
3. The database lives in a public subnet with a route to the Internet Gateway (should be a private subnet with no IGW route).

The database is protected only by username/password authentication. There is no network-layer boundary between an attacker on the internet and your production data.

**Impact:**
- **Any leaked `DB_PASSWORD` becomes an immediate full-prod compromise.** Password leaks happen — through logs, CI artifacts, shell histories, backup tarballs, old commits, accidentally-shared transcripts. Today every such leak is a game-over event. In the industry-standard shape (private subnet + SG-to-SG allow), a leaked password also needs network access to be exploitable.
- **Shared credential + brute-force surface.** Until IAM database auth is enabled, authentication is a single shared secret. A public endpoint lets attackers brute-force or credential-stuff it with no rate limit and no observable signal beyond RDS logs.
- **Accidental exposure** through mis-scoped ops queries from personal machines, laptops that later get stolen, or debugging tooling that caches credentials.

**Recommendation (in order of urgency):**
1. **Set `PubliclyAccessible=false`** on the RDS instance (`aws rds modify-db-instance --no-publicly-accessible`).
2. **Tighten the SG**: allow port 5432 only from the ECS task security group for app access, and from a bastion/SSM/VPN SG for human access. Remove any `0.0.0.0/0` rule.
3. **Move the instance to a private subnet** if it isn't already — one with a route table that has no IGW target.
4. **Stand up AWS Systems Manager Session Manager** (or a bastion, or AWS Client VPN) for the handful of humans who need DB access. This removes the need to ever expose the DB to the internet for ops work.
5. **Enable IAM database authentication** so human access uses short-lived IAM-vended credentials rather than a shared password.
6. **Enable enhanced logging and CloudWatch alarms** on failed connection attempts — detection for the brute-force surface that currently isn't being watched.

---

#### SEC-02: No Rate Limiting on OTP or Login

**Location:** `api/users.py` (OTP + login endpoints), `services/users.py:281` (Twilio send), `settings.py:16–17`

**Description:** `limits>=3.6.0` is in `requirements.txt` but is not wired into the app (no `slowapi` or equivalent middleware). `AppConfig.otp_attempts = 3` and `otp_minutes = 15` exist but are enforced only in application logic via DB counters after the request has already been serviced — there is no IP-based or phone-number-based throttle in front of Twilio.

**Impact:**
- SMS-bombing / Twilio quota exhaustion by hammering `/auth/otp/send`.
- Username enumeration (response differences disclosing valid phone numbers).
- OTP brute force via regenerate-bypass — 4-digit OTPs have only 10 000 possibilities; with 3 attempts per generated OTP and no send-side throttle, an attacker rotates new OTPs indefinitely.

**Recommendation:** Install `slowapi` and apply aggressive limits:
```python
@limiter.limit("3/15minutes")  # OTP send
@limiter.limit("5/5minutes")   # OTP verify
@limiter.limit("10/minute")    # login_admin
```
Key limits on phone number (for OTP send) and on IP (for all). Ensure the response for "unknown number" is identical to "OTP sent" to avoid enumeration.

---

### HIGH

#### SEC-03: Wildcard CORS Policy (Admin Panel Exposure)

**Location:** `main.py:31–36`, `admin.py`

**Description:**
```python
app.add_middleware(
    CORSMiddleware,
    allow_headers=["*"],
    allow_origins=["*"],
    allow_methods=["*"],
)
```
CORS is a **browser-enforced** mechanism. Native iOS/Android clients consuming this API from the mobile app ignore CORS headers entirely, so the wildcard has no effect on the mobile surface. Two browser-reachable surfaces make it still worth fixing:

1. **SQLAdmin panel** — the same FastAPI app mounts an HTML admin panel (`admin.py`, `main.py:55–61`) that is browser-served. A wildcard origin expands the attack surface for CSRF-style probing and cross-origin reads against admin endpoints. This is the concrete risk today.
2. **Latent web surfaces** — any future web dashboard, OAuth callback, deep-link handler, or marketing page that fetches from this API inherits the wildcard.

`allow_credentials` is left at the default `False`, which blunts cookie-based CSRF. If anyone ever flips it to `True` (e.g. to make the admin panel work from a separate origin), the wildcard+credentials combination is invalid per the CORS spec and browsers will start rejecting requests — the wrong-but-easy fix at that point is usually to add origins piecemeal, which is the right answer anyway.

**Impact:** Cross-origin probing / reads against the admin panel from any malicious web page. Minimal impact on the pure mobile API surface.

**Recommendation:** Restrict `allow_origins` to the real browser origins — `get_config().app.front_end_url` and the admin panel's origin. Restrict `allow_methods` to the verbs actually used. Drop `allow_headers=["*"]` in favour of the real header set. If the admin is only accessed from one known origin (or behind a VPN), set it explicitly.

---

#### SEC-04: Unauthenticated Photo Upload (Pass-Through Server Upload)

**Location:** `api/posts.py:46–61` (posts), `api/pets.py:118–132` (pets — same bug in sibling endpoint), `services/photos.py:34–45`, `utils/s3.py:17–27`, `utils/s3.py:38–60`

**Description:** `POST /post/photo` does not declare `Depends(get_current_user)` — unlike every other mutating handler in the same file (`post_create`, `post_photos_delete`, `comment_create`, etc.). Any unauthenticated caller on the internet can upload arbitrary image files.

The upload is also a **pass-through server flow**, not a direct-to-S3 presigned-PUT:

```
client → FastAPI worker (UploadFile)
       → services.photos.upload_photo()
       → utils.s3.S3Service.upload()  ← sync boto3
       → S3
```

A presigned-URL helper exists (`utils/s3.py:38–60`) but is hard-coded to `ClientMethod="get_object"` — it's used only for **downloads**. No presigned-PUT is wired anywhere. So every upload:

1. Consumes ingress bandwidth on the FastAPI worker.
2. Blocks the async event loop for the duration of the boto3 put (boto3 is sync — see scalability §2).
3. Creates a `PhotoModel` row in Postgres with just the S3 key and **no user / post / pet binding** (`services/photos.py:41–43`). Binding to a parent resource happens in a later call.
4. Puts an object at `media/post/{uuid}.{ext}` in S3.
5. Returns the `PhotoType` (including the S3 key) to the caller.

There is no orphan reaper. The README's open TODO ("Delete unused photos from S3 — background task") confirms this. Every abusive upload is permanent S3 storage + a permanent orphan DB row.

The equivalent pet photo upload in `api/pets.py:118–132` is **affected by the same bug** — it also has no `Depends(get_current_user)`. The source comment at `api/posts.py:53` (`TODO should be replaces: duplicated code from pets`) confirms the two endpoints are cut-and-paste siblings. Fix both.

**Impact:**
- **Storage cost / quota DoS** — an attacker can fill the bucket with zero account effort.
- **Bandwidth and worker-slot DoS** — each upload occupies a worker and blocks the event loop on sync boto3.
- **Amplifies SEC-09 (decompression-bomb class) and SEC-10 (format allow-list / stored extension)** — those attacks are now reachable without an account.
- **Hostile content staging** — uploaded keys can later be referenced in reports, comments, or abuse payloads.
- **Permanent DB + S3 bloat** due to the missing orphan cleanup.

**Recommendation (short-term, minutes):**
1. Add `current_user = Depends(get_current_user)` to both `post_photo_upload` (`api/posts.py:46`) and `pet_photo_upload` (`api/pets.py:118`).
2. Stamp `author.id` / `owner.id` onto the `PhotoModel` so later delete / bind paths can authorize properly and a future reaper can key on it.
3. Replace the copy-paste with a shared `upload_photos_for(reference, current_user, ...)` helper so future additions of a third upload endpoint can't forget auth again.

**Recommendation (medium-term):**
4. Switch to **presigned-PUT uploads**: backend issues a short-lived `put_object` presigned URL bound to `media/{user_id}/{uuid}.{ext}`; client uploads directly to S3; a webhook or lazy-bind on the next API call finalises the `PhotoModel` row. Benefits: bytes never touch your workers, the boto3-blocks-the-loop problem disappears for uploads, and S3 bucket policy can enforce max size and content-type at the edge.
5. Implement the README's photo reaper — a scheduled task that deletes `PhotoModel` rows (and their S3 objects) with no parent binding older than N hours.

---

#### SEC-05: Long-lived Tokens with No Revocation

**Location:** `settings.py:86–87`, `utils/auth.py`, `api/users.py` (no logout)

**Description:** Access tokens last **6 days** (`expire_minutes = 8640`) and refresh tokens last **30 days**. There is no logout endpoint that revokes tokens, no `jti` claim with a Redis blacklist, and refresh tokens are not rotated on use. A leaked token is valid until its natural expiry.

**Impact:** Device theft, token exfiltration via compromised mobile storage (the mobile report's SEC-01 flags AsyncStorage plaintext), or a leaked log entry all yield a minimum 6-day window of account takeover.

**Recommendation:**
1. Shorten access tokens to 15–60 minutes; keep refresh at 7–14 days.
2. Add `jti` claims and a Redis token blacklist with TTL = remaining token lifetime.
3. Rotate refresh tokens on every use (old refresh → blacklist on exchange).
4. Add `POST /auth/logout` (current device) and `POST /auth/logout_all` (every device).

---

#### SEC-06: Plaintext OTP Storage + Hard-Coded `"1111"` Backdoor

**Location:** `services/users.py:345–370` (`generate_otp`), `services/users.py:368` (weak RNG), `services/users.py:223, 254, 383` (OTP comparison), `utils/sms.py:12–41` (test number list and SMS suppression)

**Description:** Three distinct problems compound here.

**(a) OTPs are stored and compared in plaintext.** The OTP is a 4-digit string written to `UserModel.otp` (`services/users.py:278`) and compared with `!=` (`login_user:223`, `change_phone:254`, `verify_otp:383`). A read-only DB breach reveals every in-flight OTP and the OTP of any account whose user is mid-login.

**(b) A fixed OTP of `"1111"` is a permanent backdoor for 22 hard-coded phone numbers — in all environments.** `generate_otp()` at `services/users.py:361–366`:

```python
if (
    get_config().app.environment.lower() == "dev"
    or phone_number in twilio.get_test_numbers()
):
    logger.debug(f"Tested phone number detected: {phone_number}")
    user_data["otp"] = "1111"  # used for testing
```

The second clause is **not gated to dev**. The list lives in `utils/sms.py:12–35` and includes 22 Australian / NZ / Armenian phone numbers, several annotated with real names in comments — *"client's phone number"*, *"Shane Kim"*, *"Stasi Anastasi"*, *"Ben Carter"*, *"Julie Anastasi"*, *"Dani Ross"*, *"Lucy"*, *"Esther"*, *"Edward"*, *"Daniel"*, *"Simon"*, *"Marco"*, *"Cameron"*. On production, any of those accounts has a permanent OTP of `1111` and receives **no SMS** (suppression path in `utils/sms.py:40–41`). An attacker who guesses or enumerates any of those numbers logs in as that user.

The first clause is also a concern operationally: in any environment set to `ENVIRONMENT=dev` (including preview/ephemeral stacks), **every** user's OTP is `1111`. If such an environment is reachable from the public internet, it's a blanket auth bypass.

**(c) Weak RNG for the non-test path.** `services/users.py:368` uses `random.randint` (Mersenne Twister) rather than `secrets.randbelow`. Hard to exploit in practice for a 4-digit output, but the wrong module for a security token.

**Impact:**
- **Targeted account takeover** of any of the 22 hard-coded numbers — many of which appear to be staff, the client, or their family — on production, with no SMS or telemetry to the real user.
- **PII disclosure in source**: real names and phone numbers committed to git, which per SEC-14 should be treated as a separate data-handling incident.
- **Blanket bypass on any `ENVIRONMENT=dev` deployment** reachable from the internet.
- **Any DB breach leaks active OTPs in the clear.**

**Recommendation:**
1. Remove the entire `test_numbers` list and the `or phone_number in twilio.get_test_numbers()` branch from `generate_otp`. If bypass is needed for Apple review / E2E tests, gate it behind a single env flag (`OTP_TEST_BYPASS_ENABLED`) that is forcibly `False` in stage/prod Pydantic validation. Prefer Twilio's own magic test numbers.
2. Hash OTPs with bcrypt before storage; compare with `bcrypt.checkpw`.
3. Switch OTP generation to `secrets.randbelow(10_000)` (or go to 6-digit OTPs; 4 digits is small even with rate limiting in place).
4. Purge the 22 real names + phone numbers from git history and treat it as a PII incident — notify the individuals.
5. Log OTP send/verify events **without the OTP value**.

---

#### SEC-07: Broken Object-Level Authorization (IDOR) on Three Endpoints

**Location:** `api/users.py:319–329` + `services/users.py:288–301` (device hijack); `api/pets.py:118–132` (covered under SEC-04); `api/users.py:299–316` (photo-delete oracle)

**Description:** Object-level authorization — the check that *this* caller is allowed to touch *this specific object ID* — is **mostly done well**. Posts (`api/posts.py:121, 136`), pets (`api/pets.py:206, 242`), comments (`api/posts.py:221`), user photos (`api/users.py:291`), and photo-deletion paths (`api/posts.py:76–77`, `api/pets.py:147`) all assert `resource.author_id / owner_id == current_user.id` before proceeding. Three endpoints break the pattern:

**(a) Device ownership can be hijacked.** `POST /me/device` takes a client-supplied `device_id` (a Firebase FCM token). `services/users.py:288–301`:

```python
async def add_device(user_id: int, device_id: str, session: AsyncSession):
    if await UserDeviceDAO(session).check_existence(device_id=device_id):
        await UserDeviceDAO(session).update_owner(
            device_id=device_id, user_id=user_id
        )
        ...
```

If the `device_id` is already registered to another user, ownership is silently **reassigned** to the caller. An attacker who knows victim B's FCM token — tokens routinely leak via crash reports, analytics, and logs — calls `POST /me/device` with B's token and starts receiving B's push notifications (and B stops receiving their own). This is a direct IDOR with user-visible impact: targeted users lose OTP / friend-request / message push delivery, and an attacker gains a covert channel into the victim's notification stream.

**(b) Pet photo upload accepts anonymous callers.** `api/pets.py:118–132` has no authentication dependency — covered in full under SEC-04's expanded scope.

**(c) Photo delete leaks existence via response differential.** `api/users.py:299–316`:

```python
if id in [photo.id for photo in current_user.photos]:
    key = await delete_photo(photo_id=id, session=session)
    ...
    return MessageType(detail=f"Photo {key} was deleted")

return MessageType(detail="Photo was deleted before")
```

Three distinct outcomes (owned-and-deleted / not-owned / doesn't-exist) collapse into two response bodies that differ from each other, which lets an attacker enumerate photo IDs and probe which ones exist in the system. It also leaks the S3 key in the success path, which shouldn't be necessary.

**Impact:**
- **Device hijack** — targeted DoS of a victim's push channel and covert hijack of their OTP / notification stream. Quiet (no signal to the victim) and trivially automatable.
- **Unauthenticated pet photo upload** — everything in SEC-04.
- **Photo-ID enumeration** — low-severity information disclosure; useful reconnaissance feed for other attacks.

**Recommendation:**
1. **Device endpoint:** make `POST /me/device` idempotent only for the caller. If the device_id exists under a *different* user, either reject with 409 or require a fresh OTP-style verification before reassigning. At minimum, log + alert on every re-owner event — a legitimate user changing phones is rare; silent stealing should be detectable.
2. **Pet photo upload:** add `Depends(get_current_user)` — same two-line fix as SEC-04.
3. **Photo delete:** return 403 for not-yours, 404 for doesn't-exist, 200 for deleted, with identical body shapes on the two non-success cases. Stop returning the S3 key in the success body.
4. **Systemic fix:** add a small `assert_owned(resource, user)` helper and adopt it everywhere an ID comes from the URL. Cover the authorization matrix in tests (a dedicated `tests/authz/` directory proving 403 on every mutating endpoint when called with a non-owner's ID).

---

### MEDIUM

#### SEC-08: JWT Default Value + No Startup Validation

**Location:** `settings.py:82–91`, `main.py:37`, `auth_backend.py` (AdminAuth), `retrieve-secrets.py`

**Description:** `JWTConfig.secret` defaults to the string literal `"some_secret"` (`settings.py:83`). In practice this default is **not reached in deployed environments** — `run.sh` runs `retrieve-secrets.py` at container start, which exports the JWT secret from AWS Secrets Manager, and Pydantic's `env_prefix = "jwt_"` overrides the default from the `JWT_SECRET` env var. The risk is therefore latent rather than active:

1. **No startup validation.** `retrieve-secrets.py` blindly exports whatever JSON keys it receives. If the Secrets Manager payload for a new environment is missing the `JWT_SECRET` key (or contains a typo — `Jwt_Secret` / `JWT_TOKEN`), the app silently falls back to `"some_secret"` with no boot-time assertion to catch it.
2. **Shared secret across auth surfaces.** The same `get_config().jwt.secret` keys Starlette's `SessionMiddleware` (`main.py:37`) and the SQLAdmin `AdminAuth` backend. A single secret compromise (or rotation slip-up) affects API tokens, signed session cookies, and the admin panel simultaneously.

**Impact:** A misconfigured environment — most likely a preview, ephemeral, or newly-provisioned stack — would run on a publicly known constant without any signal that something is wrong. Attackers can forge valid JWTs and admin session cookies against any such instance.

**Recommendation:**
1. Remove the default: declare `secret: str` with no fallback so Pydantic raises `ValidationError` on boot when the env var is missing.
2. Add a validator rejecting `"some_secret"` and values under 32 bytes.
3. Split the three concerns into distinct secrets — `JWT_SECRET`, `SESSION_SECRET`, `ADMIN_SESSION_SECRET` — each generated with `secrets.token_urlsafe(32)`. Rotate independently.
4. Schema-validate the `retrieve-secrets.py` output against the expected key set before export.

---

#### SEC-09: Weak Image Validation (Upload DoS + Downstream Decoder Exposure)

**Location:** `services/photos.py:81–89` (`validate_photo`), `services/photos.py:34–45` (upload path), `utils/s3.py:17–27`

**Description:** `validate_photo()` runs `Image.open(photo.file); img.verify()`. Because the backend subsequently uploads the raw bytes to S3 via `boto3.upload_fileobj` **without re-decoding**, the classic "tiny PNG declaring 30 000 × 30 000 pixels OOMs the worker on pixel load" attack does not fire on the worker itself — `verify()` parses the container structure but does not decode pixel data. The practical risks are a narrower but still real set:

1. **No byte-size cap before `Image.open()`.** `UploadFile` has no `max_length`. A 500 MB file spools through FastAPI and into the Pillow header parser before anything rejects it — a plain upload-size DoS that is particularly exploitable given SEC-04 (unauthenticated upload).
2. **Format-specific header-time allocations.** Pillow's `Image.open()` and `verify()` are not free on hostile inputs. TIFF IFD parsers, WebP VP8L transforms, and animated-GIF colour-table parsers all allocate based on values read from the header before any pixel decode happens. `Image.MAX_IMAGE_PIXELS` does not help here — its guard fires during pixel load, not structural parsing. Pillow has a long history of CVEs in this area; staying current requires a dependency-scanning step which this project doesn't have (see CI/CD report).
3. **`verify()` is single-shot and doesn't cover the stored bytes.** Pillow's docs are explicit: *"If you need to load the image after using this method, you must reopen the image file."* The code seeks to 0 and hands the stream to boto3; `verify()` only attested to what the header looked like at first read, not to the bytes that end up in S3.
4. **Downstream decoder exposure is the real blast radius.** The backend stores whatever validated, but every consumer — the React Native mobile app rendering the feed, any admin-panel thumbnail, any future web client — decodes pixels based on declared dimensions. Mobile decoders are much weaker than Pillow; a hostile image accepted by the backend can crash every mobile client that scrolls past it in a public feed.

**Impact:**
- Worker memory pressure / spool-disk exhaustion from oversized uploads, pre-auth (combined with SEC-04).
- Possible worker-side allocation spikes on crafted TIFF/WebP/GIF headers.
- **System-wide client crashes** from hostile images surviving validation and then being decoded by mobile clients that land on a feed containing them. This is the highest-impact piece and the reason this stays at Medium rather than Low.

**Recommendation:**
1. Enforce a hard byte cap (e.g. 10 MB) *before* reading the stream — reject at the FastAPI dependency layer using a streaming size check, not after `UploadFile` has fully spooled.
2. Set `Image.MAX_IMAGE_PIXELS` explicitly (e.g. 40 megapixels) and convert the `DecompressionBombWarning` default into a hard `raise UnsupportedMediaType`.
3. **Re-encode every accepted image with Pillow before handing bytes to S3**: open → resize/clamp dimensions → strip metadata → save to a fresh buffer → upload that buffer. This turns any surviving structural weirdness into a clean re-encoded image and removes EXIF PII as a bonus.
4. Combine with SEC-10 (format allow-list) so unexpected formats never reach the parser in the first place.
5. Add dependency-vulnerability scanning (pip-audit / Dependabot) to catch new Pillow CVEs as they're published (see CI/CD report).

---

#### SEC-10: No Format Allow-List + Client-Controlled File Extension on S3

**Location:** `services/photos.py:81–89` (Pillow check), `utils/s3.py:17–27` (S3 upload)

**Description:** This one is subtler than a plain "no MIME checking" headline — there are three layered gaps.

**(a) Pillow content check is present, and it's the stronger check.** `Image.open() + img.verify()` is a *content* validation — it parses the bytes as an actual image. That's correctly stronger than `photo.content_type`, which is client-supplied and trivially spoofable. A `.exe` renamed `.jpg` does not pass. So the "is it actually an image?" question is answered.

**(b) But there is no format allow-list.** Pillow supports ~40 formats (JPEG, PNG, WebP, GIF, TIFF, BMP, ICO, PCX, PSD, DDS, TGA, WMF, …). `Image.open() + verify()` accepts every one of them. The backend probably only wants to store JPEG / PNG / WebP for the CDN — accepting the long tail widens the CVE surface to every Pillow parser. A format allow-list (checking `img.format in {"JPEG", "PNG", "WEBP"}` *after* open is the cleanest implementation) closes this without depending on the client-supplied MIME at all.

**(c) The stored file extension is client-controlled, and no `Content-Type` is set on the S3 object.** `utils/s3.py:22`:

```python
filename = f"{uuid.uuid4().hex}{os.path.splitext(file.filename)[-1]}"
key = f"media/{reference}/{filename}"
bucket.upload_fileobj(file.file, key)
```

Two things happen here that shouldn't. The extension comes from `file.filename` (a client-controlled string), not from `img.format` (what Pillow determined the image actually is). And `upload_fileobj` is called with no `ExtraArgs={"ContentType": ...}`, so S3 defaults to `binary/octet-stream` unless something downstream overrides.

**Concrete attack:** upload a legitimate PNG named `attack.svg` or `attack.html`. Pillow opens the PNG content fine → validation passes. The S3 key is stored as `media/post/{uuid}.svg`. Served from CloudFront:
- If CloudFront (or a Lambda@Edge) derives `Content-Type` from the extension, browsers receive `image/svg+xml` or `text/html` and **execute** the response. SVG supports `<script>` tags; HTML obviously does. That's stored XSS on the CDN.
- If CloudFront passes through S3's `binary/octet-stream`, browsers typically trigger a download — not a security issue but not what you want in a feed.

The exploitability depends on CloudFront's config, which this repo doesn't contain — but trusting a client-controlled string for the stored extension while *also* not setting an explicit Content-Type is the kind of latent issue that breaks silently the moment the CDN config changes.

**Impact:**
- Expanded CVE surface via unnecessary Pillow formats.
- Latent stored-XSS on CloudFront via attacker-chosen file extension if the CDN derives Content-Type from path. Severity is conditional on CDN config, which is why this stays Medium.

**Recommendation:**
1. After `Image.open()`, allow-list `img.format in {"JPEG", "PNG", "WEBP"}` and reject otherwise.
2. Derive the stored extension from `img.format`, not from `file.filename`. The client never chooses a byte of the stored S3 key.
3. Pass `ExtraArgs={"ContentType": mime_for_format}` to `upload_fileobj` so the S3 object is served with the correct content-type regardless of CDN inference.
4. Audit the CloudFront distribution for how it resolves `Content-Type` — ideally force it to read from S3 metadata rather than inferring from path.
5. Combine with SEC-09's recommendation to **re-encode** accepted images — at that point the stored file is a freshly-produced Pillow output and any client-controlled format ambiguity is gone.

---

#### SEC-11: Sync SMTP / SMS in Request Path Leaks Blocking Time

**Location:** `utils/email.py`, `utils/sms.py`

**Description:** Twilio and SMTP operations are synchronous in an async context. Beyond the scalability impact (see scalability report), they create a timing oracle: a response that takes 300 ms on a known phone number vs. 5 ms on an unknown one leaks which numbers are registered. Rate limiting (SEC-02) slows enumeration but doesn't remove the signal.

**Recommendation:** Move Twilio off the response path entirely — generate + store the OTP, enqueue the SMS as a background task, and return immediately with an enumeration-safe body (`"If that number is registered, an OTP has been sent."`). As a band-aid until that ships, add a minimum-latency floor so every response takes ≥500 ms regardless of path.

---

### LOW

#### SEC-12: Centrifugo Tokens Not Bound to Channels

**Location:** `utils/centrifugo.py:33–42`

**Description:** WS connect tokens carry only `{sub: user_id, exp: +1h}`. There is no `channels` claim constraining the subscriptions a client may request.

**Recommendation:** Add `channels` to the payload (e.g. `["furbaby:{env}:user#{user_id}"]`) and reduce lifetime to ~15 min; issue short-lived tokens on reconnect.

---

#### SEC-13: `createuser.py` Has No Audit Trail

**Location:** `createuser.py`

**Description:** The superuser bootstrap script creates privileged accounts with no logging, approval, or 2FA gate.

**Recommendation:** Log all superuser creations to an admin-audit table; require a break-glass env flag; prefer a one-off management command with logged invocation.

---

#### SEC-14: PII in Source (Test Numbers)

**Location:** `utils/sms.py`

**Description:** A block of apparently real names and phone numbers is hard-coded as the SMS test whitelist.

**Recommendation:** Move to an env-provided list. Treat the existing commit history as compromised PII and decide whether to rotate / apologise.

---

### INFORMATIONAL

#### SEC-15: `is_blocked` Logic Bug in `GET /user/list`

**Location:** `api/users.py:164–187`

**Description:** The `users_list` endpoint derives each returned user's `is_blocked` flag as `user.id not in friends_list_ids`. That conflates "we're not friends" with "you're on my block list" — every non-friend is reported to the client as blocked.

**Impact:** Not a security-boundary violation, but a likely UI lie (and a potential privacy concern — a mobile user may believe they've blocked more people than they actually have, or vice-versa). Flagged informational because it's an authorization-adjacent logic bug uncovered while auditing SEC-07.

**Recommendation:** Check against `FriendshipDAO.get_blocked_ids(current_user.id)` — the call exists elsewhere in this same module (`api/users.py:211–212`) and returns the real block list.

---

#### SEC-16: No Input Length Caps on Text Fields

**Location:** `schemas/*.py`

**Description:** Free-text fields (bio, post body, pet name, report text) do not consistently declare `max_length`. This invites abusive payloads.

**Recommendation:** Enforce `Field(..., max_length=...)` on every string field; add `min_length=1` where empty is invalid.

---

#### SEC-17: Firebase Credential Handling

**Location:** `firebase_keys.py`, `.deploy/`, `.gitignore`

**Description:** Firebase service-account JSON is written to `.deploy/furbaby_firebase_config` at boot, downloaded per environment by `firebase_keys.py`. The filename pattern is git-ignored. The approach is reasonable, but ensure:
- `.deploy/` is broadly ignored, not just the specific file name.
- The service account is scoped to the minimum Firebase roles needed (auth custom-token, messaging send, Firestore if used).

---

## Positive Security Practices Observed

- SQL is built exclusively through the SQLAlchemy query API; no raw `text()` string interpolation was found, so injection risk is low.
- Secret retrieval at runtime via `retrieve-secrets.py` → AWS Secrets Manager is the correct pattern.
- `ggshield` (GitGuardian) secret scanning runs as the first CI step on every branch (`.drone.yml:69–78`).
- A typed Pydantic settings layer prevents accidental misconfiguration (with the notable exception of the JWT secret default — see SEC-08).
- Elastic APM + appello-logger give good post-incident visibility.

---

## Immediate Remediation List

1. **SEC-01** — make the production RDS instance private; tighten security groups to the app-tier SG only; set up SSM / bastion for human access.
2. **SEC-02** — wire `slowapi` onto OTP and login.
3. **SEC-04** — add `Depends(get_current_user)` to both photo upload endpoints.
4. **SEC-05** — shorten access token lifetime; implement revocation.
5. **SEC-06** — remove the `"1111"` backdoor; hash OTPs; gate test bypass behind an env flag.
6. **SEC-03** — scope CORS to the admin-panel origin.
7. **SEC-08** — remove the JWT default + add a Pydantic validator; split JWT / session / admin secrets.
