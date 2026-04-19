# FurBaby Social Backend - Post-Cleanup Roadmap

**Date:** 2026-04-19

---

## Scope

This document is **not** a list of current bugs. The audit findings in reports 02–06 are the "things that are broken today." This roadmap is what to think about *after* those have been remediated — the next tier of operational maturity you'll want in place before the product grows beyond its current shape.

Everything here is framed as "future concerns." None of it is an emergency. But the earlier you plan for them, the cheaper each one is when you actually need it.

---

## 1. Environment Parity

### Today
- CI has pipelines for dev, stage, and prod (`.drone.yml`), but only prod is actually receiving real users.
- Dev and stage share a single ECS cluster per the Drone config; in practice stage appears to be an unused placeholder.
- Every schema migration, configuration change, and dependency bump is being tested directly against production.

### Why it matters
- **Migrations are unrehearsed.** Alembic's `test_migration.py` stairway validates revisions against a clean DB, but it doesn't catch issues that only surface against real-shape data.
- **Security changes can't be rehearsed.** Flipping the RDS to `PubliclyAccessible=false` (SEC-01), tightening security groups, or changing the Centrifugo host all need a dry run somewhere — and today, "somewhere" is prod.
- **Mobile QA has nowhere safe to live.** Internal TestFlight / internal Play tracks should point at a staging backend, not prod.

### What to build
- A real stage environment — same AWS account as prod is fine, just tagged and isolated. Size everything smaller: `db.t4g.small` RDS, 1-task ECS, no CloudFront (serve directly from the ALB) if cost is a concern.
- Mobile stage builds consuming the stage backend, distributed via internal TestFlight + Play internal testing.
- A **promotion flow** in CI: the prod deploy consumes the image digest that passed stage, not a fresh build from main. "If it worked on stage, it'll work on prod" becomes a real property of the pipeline.
- Dev can remain a looser shared environment — its current shape is appropriate for its purpose.

### Rough cost
One small RDS instance + one ECS task + an ALB. Depending on the class, roughly **USD 50–100/month**. Cheap compared to the cost of one bad prod deploy.

---

## 2. Scaling Ceilings

The scalability report (05) covers the things to fix *within* the current architecture — indexes, over-fetching, background tasks, sync-SDK unblocking. Those get you from "current small user base" to "a few thousand DAU." Past that, the architecture itself starts to limit you.

### What breaks first, and when

| Component | Current shape | Likely breakpoint | Fix at that point |
|---|---|---|---|
| **RDS** | Single small instance (class not yet verified) | 5–10k DAU once indexes + over-fetching are fixed | Scale class up; add a read replica for feed queries |
| **ECS tasks** | 1 task | *Already a problem:* any deploy or crash is full outage | Move to min 2 tasks with auto-scaling; ALB health-drains during deploy |
| **DB connections** | `pool_size=10, max_overflow=5` per task | Second task = 30 connections against whatever RDS connection limit you have | RDS Proxy (or pgbouncer) in front of Postgres |
| **Centrifugo** | Single instance | 1–5k concurrent WebSocket users | Horizontal Centrifugo with Redis broker, or migrate to a hosted alternative |
| **Redis** | Single instance, per-worker in-memory fallback | 10k+ active sessions, or Redis outage breaks cache coherence | ElastiCache cluster mode, disable the in-memory fallback |
| **S3 + CloudFront** | Already horizontally scalable | Not a bottleneck | — |

### What to do now (or next)

**Do now regardless of scale:**
- **Run with ≥2 ECS tasks.** This isn't about throughput; it's about availability. One task means every deploy is a zero-downtime lie and every crash is a full outage.
- **Put an ALB with target-group health checks in front.** Health-drain during deploy so users don't hit draining tasks.
- **Deepen the health check** (already SEC recommendation #14) so a broken task actually fails health.

**Do when you're planning to scale past current load:**
- **RDS Proxy** before going multi-task with any real concurrency. Otherwise connection count escalates linearly with task count.
- **Target-tracking auto-scaling** on p95 latency and CPU — ECS will grow the service under load.
- **Read replica** for the heavy feed queries once indexes are in and the hot queries are identified in APM.

**Know-thy-sizing:** Worth confirming the current RDS instance class via a one-minute AWS Console check. If it's `t3.micro` the runway is shorter than you think; if it's `t3.small` or larger there's months of headroom once the index work is done.

---

## 3. Operational Resilience

### Today
- **Elastic APM** is installed and collects traces + errors (when `ELASTIC_APM_SERVER_URL` is set).
- **`appello-logger` + Logstash** for structured logging.
- **Slack notifications** on Drone build success/failure.
- **Health check** is a static 200 (scalability §7) — doesn't check DB, Redis, Firebase, or Centrifugo.
- **No documented alert rules.** APM captures errors but nothing pages anyone.
- **No SLOs** defined.
- **No on-call rotation** is implied by the tooling.

### What to add

**Alerting and on-call**
- Define a small set of SLOs to start — e.g. "99% of `/auth/otp/send` under 2s," "0.5% non-deprecated 5xx rate," "health check green for 99.9% of minutes."
- Wire APM → PagerDuty (or Opsgenie) for rule-based alerts on the SLOs.
- Document an on-call rotation even if it's one person — it sets expectations for response time.

**Health and readiness**
- Deepen `/healthcheck` to verify DB, Redis, and Firebase connectivity. Wire it into the ECS target-group health check so broken tasks actually fail health and get replaced.
- Separate liveness (is the process alive?) from readiness (can it serve traffic?) — ECS supports both.

**Backups and DR**
- Verify RDS **point-in-time recovery** is enabled and the retention window is adequate (7–35 days typical).
- **Actually test a restore.** "We have backups" is only true once you've successfully restored one into a scratch environment. Do it once per quarter.
- For a pet-social-network, formal DR targets (RTO/RPO) are probably overkill today, but once you're taking payments or storing sensitive data, a documented DR plan is worth having.

**Runbooks**
- Three short markdown docs in the repo, each under a page:
  - *"The API is down"* — how to diagnose, who to contact, how to revert the last deploy.
  - *"The DB is slow"* — where APM's slow-query view lives, how to kill a runaway query, how to fail over to a replica if one exists.
  - *"Deploy rollback"* — the exact ECS commands to redeploy a previous image tag.
- These don't have to be perfect to be useful. A three-sentence "here's what happens when X breaks" beats nothing.

---

## 4. Security Maturity (Beyond the Audit Basics)

Once the audit findings are closed (reports 04 remediation list), the next tier:

- **WAF in front of the API.** AWS WAF with the managed rule sets — bot protection, SQL-injection signatures, common CVE rules. Cheap and catches the baseline internet noise you don't want to see in APM.
- **Secrets Manager rotation.** Today secrets are stored in Secrets Manager but not rotated. RDS password rotation is a built-in Secrets Manager feature; enable it and set a rotation cadence.
- **External penetration test** once the audit fixes have shipped. It's the correct time — you've closed the holes you know about, now find out what you don't. Scope it to the API surface + mobile app.
- **Compliance posture check** if you have AU users. Australia's Privacy Act / APPs have specific requirements around breach notification, user access, and correction. Worth confirming your posture before you're asked.

---

## 5. Product / Platform Gaps

Not security or infra — things that come up as the product matures and are much cheaper to add early than retrofit:

- **Feature flags.** LaunchDarkly, Unleash, or even a simple DB-backed flag table. Lets you ship code dark, enable per-cohort, and disable on a bad-deploy signal without re-deploying.
- **Analytics pipeline.** Segment, PostHog, Amplitude. Today you have request counts in APM and not much else about what users actually do. The product conversations get much easier when "DAU," "retention," and "time on feed" have real numbers behind them.
- **Purpose-built admin tooling.** SQLAdmin is fine for dev poking around; once you have a support team, they'll need curated views (user lookup with recent activity, force-logout, refund-if-applicable, content moderation queue). Expect this to be a small internal app rather than extending SQLAdmin indefinitely.
- **Data exports.** A simple "download all my data" endpoint comes up once you have engaged users. Same plumbing you'll want anyway for GDPR/APP-style access requests.

---

## 6. Cost & Governance

Boring but material once AWS bills start mattering:

- **Tag every AWS resource** with `env`, `service`, `owner`. Lets Cost Explorer attribute spend — essential once dev/stage costs diverge from prod.
- **Cost alerting** on monthly spend thresholds. AWS Budgets is free and emails you before a runaway task bankrupts the month.
- **Dependency update cadence.** Even with Dependabot / pip-audit configured (audit rec #17), you need a rhythm for reviewing and merging — a half-day every two weeks keeps the queue manageable.
- **Architecture Decision Records (ADRs).** Short markdown notes in the repo for non-obvious decisions — "why we kept SQLAlchemy 1.x in 2026," "why we chose Centrifugo over socket.io," "why posts use offset pagination today." The next engineer doesn't re-open decisions that were deliberate.

---

## Prioritisation Summary

If the post-cleanup roadmap were a prioritised list:

| Priority | Item | Why this order |
|---|---|---|
| 1 | Stand up stage environment + promotion flow | Unblocks safe testing for every subsequent change |
| 2 | Min 2 ECS tasks + deeper health check + ALB drain | Availability; no longer "every deploy is a Russian roulette spin" |
| 3 | Alerting + SLOs + PagerDuty wiring | You can't fix what you don't know about |
| 4 | Verified RDS backup + tested restore drill | "Have backups" → "can actually recover" |
| 5 | WAF + Secrets Manager rotation | Cheap security hardening |
| 6 | Feature flags + analytics | Product tooling that pays off as the team grows |
| 7 | RDS Proxy + read replica | When scale warrants it (probably not until mid-cleanup) |
| 8 | Pen test | After the audit fixes ship |
| 9 | Admin tooling v2 + data exports | Once there's a support team / compliance need |
| 10 | Resource tagging + cost alerting + ADR discipline | Ongoing governance |

None of these are audit findings. They're the things that turn a "works in production today" system into "can grow to 10× users without being rewritten."
