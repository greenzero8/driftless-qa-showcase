# Driftless QA

**Live app: [driftlessqa.com](https://driftlessqa.com)**

A web app that compares approved email copy against the email that actually got built, and reports
what changed during production.

It **compares; it does not proofread.** If the approved copy has a typo and the final email
reproduces it exactly, that passes. That boundary is the product.

> This repository is a showcase. It documents the architecture and engineering decisions behind
> Driftless QA. The application source is in a private repository.

---

## The problem

Email QA platforms — Litmus, Email on Acid — are good at rendering, links, accessibility and
deliverability. None of them answer a more basic question: **does the final email actually say what
the approved copy said?**

Copy drifts during production. It moves from a copy doc into a template builder, gets edited late,
and small changes survive to send past several rounds of human review: a changed price, last
month's offer, a CTA that lost its capitalization, a paragraph quietly dropped. Once it sends,
there is no unsending it.

Driftless QA does that one check.

---

## Architecture

The comparison is a nine-step pipeline. The deterministic core runs first and produces every
verdict; two AI steps are layered on top and are strictly additive.

```
  PASTED INPUT
  approved copy doc                    final email (ESP HTML or rendered text)
        │                                          │
        ▼                                          ▼
  ┌───────────┐                            ┌───────────┐
  │ 1 EXTRACT │  strip markup, keep        │ 1 EXTRACT │
  │ 2 PARSE   │  visible text + light      │ 2 PARSE   │
  │           │  structure                 │           │
  └─────┬─────┘                            └─────┬─────┘
        │  labeled blocks                        │  ordered text runs
        │  (Headline:, Body:, CTA:…)             │
        ▼                                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ 3 REMOVE    subject / preheader, boilerplate         │
  │ 4 NORMALIZE smart quotes, entities, whitespace       │
  │ 5 DEDUPE    responsive duplicates (mobile variants)  │
  └───────────────────────┬──────────────────────────────┘
                          ▼
  ┌──────────────────────────────────────────────────────┐
  │ 6 ALIGN   which run is this block?                   │
  │                                                      │
  │   tier 1  string similarity, deterministic           │
  │           ├─ resolved ──────────────┐                │
  │           └─ leftovers              │                │
  │                  ▼                  │                │
  │   tier 2  model proposes pairings   │   ← AI         │
  │           ├─ paired ────────────────┤                │
  │           └─ still unresolved ──────┼─→ WARNING      │
  └───────────────────────┬─────────────┘   (never a     │
                          ▼                  guess)      │
  ┌──────────────────────────────────────────────────────┐
  │ 7 DIFF      word-level, on aligned pairs only        │
  │ 8 CLASSIFY  issue type + severity  ← deterministic   │
  │ 9 EXPLAIN   one sentence per finding   ← AI          │
  └───────────────────────┬──────────────────────────────┘
                          ▼
                    PASS / FAIL / WARNING
```

Images are handled on a parallel path: reachable `<img>` sources are fetched server-side and read
in one batched vision call, so copy baked into a hero JPEG is not a blind spot.

---

## The two rules

Every design decision in the project resolves to one of these.

### 1. The model proposes. Code decides.

A model is allowed to suggest *which approved block corresponds to which part of the email*. It is
never allowed to return a pass or a fail. Every verdict is computed by deterministic code from the
pairings it was given.

The tier-2 prompt asks only "which of these goes with which," and explicitly refuses to ask what
changed or whether it matters. That separation is the guardrail against the main failure mode of
simply pasting both documents into a chat window and asking — a model will confidently narrate
differences that are not there.

The practical consequence: **the same two documents always produce the same answer.**

### 2. Never diff a pair you aren't confident about.

If alignment cannot confidently determine which part of the email a block became, the app says so
instead of comparing it against the wrong thing. That surfaces as a dismissable warning rather than
a failure.

This is a deliberate asymmetry. Only failures drive the top-line verdict; warnings never do. A false
warning costs the user a few seconds. A false *failure* — a red banner on a correct email — destroys
trust in every green result afterwards, which is the only thing that makes a green result worth
anything.

---

## Where AI is used, and where it deliberately isn't

| Step | Approach | Why |
|---|---|---|
| Extraction, normalization, dedupe | Deterministic | Mechanical and exactly specifiable. A model here adds cost, latency and nondeterminism for nothing. |
| Alignment tier 1 | Deterministic (Levenshtein, token overlap, containment) | Resolves clean labeled input completely. On those inputs **no model call happens at all** — the comparison is instant and free. |
| Alignment tier 2 | Model, structured output | Only receives what tier 1 could not place. Heavy rewrites defeat string similarity but are obvious to a reader. |
| Reading text inside images | Vision model, confidence-gated | Text baked into artwork is invisible to every other check. Capped at Warning — extraction confidence doesn't support a hard verdict. |
| Verdicts | **Deterministic, always** | See rule 1. |
| Finding explanations | Model, one batched call | The only genuinely generative step. Replaced hand-written templates. |

**Both AI steps are allowed to fail.** If the API key is absent, the prepaid balance is exhausted,
or a call errors, the pipeline returns exactly what the deterministic core alone would have
returned. The app is complete and useful without either.

---

## Engineering decisions worth calling out

**Thresholds were tuned against an eval set, not by feel.** The tier-1 alignment threshold began at
a guessed 0.90. Sweeping it against the fixture set showed every pair that must match scoring ≥ 0.91
and every pair that must not scoring ≤ 0.75 — a usable band of roughly (0.75, 0.91), with a cliff at
0.92 where a paragraph reworded just short of the line stops matching and one honest wording failure
becomes two wrong findings. The value sits near the centre of the band rather than on the edge of
the cliff.

**Models were chosen per job, by measurement.** The three AI jobs are configured independently
because they were benchmarked independently, three runs each, after the shared eval set proved
unable to separate them. The vision benchmark was decisive: on artwork with typos deliberately baked
in, one candidate model read `Limted` as `Limited` and `Shiping` as `Shipping` at 0.99 confidence on
every run — silently repairing the exact defect the step exists to catch. A cheaper model that
*looks* flawless on an aggregate score can be actively wrong at the thing you bought it for.

**Fetching user-supplied image URLs is treated as a security boundary.** The app accepts arbitrary
HTML from anonymous users, extracts URLs from it, and has the server fetch them — textbook SSRF
exposure. Requests are validated on scheme, hostname, resolved IP (private and link-local ranges
rejected after DNS resolution, not before), redirect behaviour, content type, response size and a
hard timeout. Rejections are silent: a descriptive error message is a free probing tool for an
attacker.

**Limits are documented honestly rather than oversold.** Rate limiting is in-memory and therefore
per-serverless-instance — it stops casual looping and runaway client code, and nothing more. The
real spend ceiling is a prepaid balance with auto-reload off. Adding a database to make the limit
rigorous was explicitly judged not worth it at this scale, and the reasoning is recorded next to the
code rather than left for someone to rediscover.

**Performance was profiled, not guessed.** Alignment is O(blocks × runs) with a Levenshtein per
pair and accounts for ~96% of wall time on a large comparison. Cost tracks what survives extraction,
so the HTML path is cheap at any size — 400 KB of markup compares in ~24 ms, because a real ESP
export is overwhelmingly markup (one measured template carried 2.5 KB of copy under 108 KB of
Outlook hacks and inline CSS).

---

## Testing and evaluation

| | |
|---|---|
| Unit tests | **291**, across 14 suites |
| Eval fixtures | **28** end-to-end email pairs — 14 clean, 14 with a planted defect |
| Current eval score | **14/14 defects caught, at the correct severity, 0 false positives** |

The eval set is the project's regression net for behaviour that unit tests can't express. Each
fixture is a realistic approved-copy / final-email pair with a known expected outcome, covering
clean cases that must stay clean (smart quotes, responsive duplicates, MJML wrappers, repeated CTAs,
boilerplate footers) and defects that must be caught at the right level (changed wording, dropped
sentences, injected paragraphs, reordered sections, capitalization drift).

It grades **severity as well as detection**: flagging a real problem at the wrong level counts
against the score. No threshold in the codebase changes without running it.

---

## Tech stack

**Application**
- Next.js 16 (App Router, Server Components), React 19, TypeScript (strict)
- Tailwind CSS v4
- Deployed on Vercel, custom domain, automatic deploys from `main`

**AI**
- Anthropic API via the official TypeScript SDK
- Zod schemas for structured model output — the model returns typed pairings and readings, not prose
  to be parsed
- Per-job model configuration, overridable by environment variable with no code change

**Engine**
- Pure TypeScript, no runtime dependencies beyond an HTML parser
- Every module in the comparison core is composed of pure functions, which is what makes 291 unit
  tests cheap to write and fast to run
- Node's built-in test runner — no test framework dependency

**Security & operations**
- All model calls server-side; the API key never reaches the browser, the repo, or a log line
- SSRF-hardened image fetching (see above)
- Optional password gate: HMAC-signed session cookie, key derived from the password, expiry carried
  in the cookie and signed over. Off by default; enabling it is one environment variable and no code
  change
- In-memory per-IP rate limiting with a documented, honest threat model
- Nothing pasted by a user is stored anywhere

---

## What I'd do next

- **Bound the plain-text slow path.** Alignment cost is fine for real copy docs but grows badly on
  pathological plain-text input. There's a result-preserving prefilter available — the similarity
  ratio can't exceed `min(len)/max(len)`, so a pair whose upper bound already loses can skip the
  Levenshtein entirely.
- **Persist eval runs** so threshold changes can be compared across commits rather than judged one
  run at a time.

---

Built by **Dave Hyde** · [driftlessqa.com](https://driftlessqa.com) · [About the project](https://driftlessqa.com/about)
