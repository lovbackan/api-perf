# api-perf — Task Map

Status markers: `[ ]` todo · `[~]` in progress · `[x]` done

---

## Phase 0 — Spec & design (complete before writing code)

`api-perf` is `web-speed-test` for raw HTTP — same statistical philosophy, same transparency requirements, no browser overhead. Several design decisions need to be locked in first.

- [ ] **HTTP client:** use Node's built-in `node:http`/`node:https` directly — no external HTTP client. This gives us granular timing hooks (`socket`, `connect`, `secureConnect`, `response`) without abstraction layers that hide timing. Document this in methodology.
- [ ] **What timings are actually measurable without a browser?**
  - DNS lookup: `socket` event timing
  - TCP connect: `connect` event timing
  - TLS handshake: `secureConnect` event timing
  - TTFB: time from request sent to first `data` event
  - Total response time: time to `end` event
  - Response size: compressed (from `Content-Length` or actual bytes) and decompressed
  - Note: no DOM, no JS execution, no render metrics — those are browser-only. Be explicit about this in README.
- [ ] **Redirect handling:** follow redirects by default, count redirect hops, report redirect chain. `--no-follow-redirects` flag to disable.
- [ ] **Compression:** decompress gzip/br/deflate responses automatically, report both compressed and decompressed sizes.
- [ ] **Auth:** same as web-speed-test — custom headers, no credentials logged.
- [ ] **Comparison mode:** reuse same statistical comparison approach as web-speed-test (Mann-Whitney U, rank-biserial correlation) — implement locally, not as a shared library (tools should remain independent).
- [ ] **Finalize full metric list and write detailed README**

---

## Phase 1 — Project setup

- [ ] Install dependencies: `commander`, `tsx`, `typescript`
- [ ] `tsconfig.json`
- [ ] `src/index.ts` — CLI entry point
- [ ] Verify `--help` works

---

## Phase 2 — Network pre-test calibration

Goal: measure network stability before the main test — same philosophy as web-speed-test. You need to know if your measurement environment is stable before trusting results.

- [ ] For each unique hostname being tested, send 6 HEAD requests
- [ ] Measure round-trip time per request
- [ ] Report: median RTT, P10/P90, CV%, stability label (HIGH/MEDIUM/LOW)
- [ ] Include calibration results in every report
- [ ] If network stability is LOW (CV > 25%), print a warning: results may be unreliable — but do NOT block the test

---

## Phase 3 — HTTP timing measurement

Goal: for a single URL and run count, collect granular timing data per run.

**Timer attach points (using `http.ClientRequest` events):**

```
t0: request initiated
t1: socket assigned (DNS lookup starts)
t2: socket 'connect' event (TCP established)
t3: socket 'secureConnect' event (TLS done, HTTPS only)
t4: first 'data' event on response (TTFB)
t5: response 'end' event (full download complete)
```

- [ ] Implement `measureRequest(url, options): RunResult` — single timed request
- [ ] Compute derived timings:
  - DNS lookup = t2 − t1 (from socket assignment to TCP connect; DNS resolves during this window)
  - TCP connect = t2 − t1 (for HTTP) or t3 − t1 (for HTTPS, includes TLS)
  - TLS negotiation = t3 − t2 (HTTPS only; 0 for HTTP)
  - TTFB = t4 − t0
  - Content download = t5 − t4
  - Total response time = t5 − t0
- [ ] Capture: HTTP status code, response headers, redirect chain (URLs + status codes)
- [ ] Capture response size: compressed bytes (wire size) and decompressed bytes
- [ ] Handle HTTP redirects: follow up to 10 hops, report chain. Reset timers on each hop.
- [ ] Handle errors gracefully: connection refused, timeout, DNS failure — record as failed run with error type, don't crash
- [ ] `--timeout <ms>` flag (default 30000)

**Known limitation (document in methodology):** DNS timing using socket events is an approximation. The socket is assigned after the DNS lookup begins, so the window from socket assignment to TCP connect includes both DNS and potentially some queue time. This is the same limitation as browser-based tools — it cannot be avoided without OS-level hooks.

---

## Phase 4 — Multi-run statistics

Goal: same statistical engine as web-speed-test.

- [ ] Run N times (default 5, configurable with `-r`)
- [ ] Randomize run order if testing multiple URLs (`--no-randomize` to disable)
- [ ] Per-metric statistics: median, mean, P10, P90, stddev, CV%
- [ ] Reliability ratings: HIGH (CV < 10%), MEDIUM (10–25%), LOW (>25%)
- [ ] Outlier detection: IQR method — flag outliers, never remove them
- [ ] Cold-start analysis: separate run 1 from runs 2–N — report both
  - Cold run = first request (fresh TCP + TLS connection)
  - Warm runs = subsequent requests (may reuse connection — depends on keep-alive)
  - Note in methodology: unlike web-speed-test with Puppeteer's fresh browser context, connection reuse between runs depends on Node.js's HTTP agent behavior. Document this.
- [ ] Status code consistency check: if different runs get different status codes, warn

---

## Phase 5 — Before/after comparison (`--compare`)

Replicates web-speed-test's comparison feature for API endpoints.

- [ ] Load a previous run's JSON report
- [ ] Config mismatch detection: warn if run count, URL, or timeout differ
- [ ] Per-metric: Mann-Whitney U test to detect significant difference
- [ ] Effect size: rank-biserial correlation (negligible/small/medium/large)
- [ ] Noise-aware labels: changes within measurement noise band labeled "~same"
- [ ] Include same caveats as web-speed-test: correlation ≠ causation, small sample size

---

## Phase 6 — Output

### Markdown report structure

```
# api-perf report — <url>
<timestamp> · <N> runs · <status code> · <median total time>ms

## Overview
<3-line TL;DR>

## Timing breakdown (median across N runs)
DNS Lookup        Xms  [reliability]
TCP Connect       Xms  [reliability]
TLS Negotiation   Xms  [reliability]
TTFB              Xms  [reliability]
Content Download  Xms  [reliability]
Total Response    Xms  [reliability]

## Statistical detail
...

## Cold-start vs warm-run analysis
...

## Response details
Status code, size compressed/decompressed, redirect chain, headers of interest

## Network calibration
...

## Before/after comparison   ← only with --compare
...

## Methodology
<auto-generated>

## Threats to validity
```

- [ ] Implement markdown report generator
- [ ] Auto-save to `reports/<hostname>/<timestamp>.md` + `.json`
- [ ] `--json` stdout mode

### Auto-generated methodology section

- [ ] HTTP client: `node:http`/`node:https` built-in
- [ ] Timer attach points and derived timing formulas
- [ ] Redirect handling setting
- [ ] Run count and randomization setting
- [ ] Connection keep-alive behavior (document Node.js default)
- [ ] DNS timing approximation limitation
- [ ] Threats to validity: single geographic location, no CDN edge realism, lab data not field data

---

## Phase 7 — Performance budgets (`--budget`)

- [ ] Available budget keys:
  - `ttfbMs`
  - `totalResponseMs`
  - `tlsNegotiationMs`
  - `contentDownloadMs`
  - `compressedSizeBytes`
  - `decompressedSizeBytes`
- [ ] All compared against median across runs
- [ ] Exit code 1 on exceeded
- [ ] Budget results in report

---

## Phase 8 — Multi-URL support

- [ ] `-u` flag repeatable (same as web-speed-test)
- [ ] Test each URL independently, report each separately
- [ ] Enforce single-domain rule: all URLs must share the same hostname (same reasoning as web-speed-test — multi-domain results are not comparable without domain context)

---

## Phase 9 — End-to-end validation

- [ ] Test against a known endpoint (e.g. `https://httpbin.org/get`)
- [ ] Verify timing breakdown sums approximately to total response time
- [ ] Verify cold-start run 1 shows higher DNS/TCP times than warm runs
- [ ] Verify redirect chain is captured and reported correctly
- [ ] Review README against actual implementation

---

## Open questions

- [ ] Should we support HTTP/2? Node's built-in doesn't support H2 without `http2` module. H2 changes connection semantics significantly (multiplexing, no separate TCP per request). Proposal: HTTP/1.1 only for v1, note limitation in README.
- [ ] Keep-alive between runs: by default Node.js uses keep-alive. Should we disable it per run to get a "fresh connection" measurement? Proposal: offer `--fresh-connection` flag that creates a new Agent per run (equivalent to web-speed-test's `--full-isolation`). On by default for fairness.
- [ ] Should we support POST/PUT/PATCH with a body? Proposal: yes, via `--method` and `--body` flags. Defer to v1.1 unless easy to add.
- [ ] gRPC support? Definitely out of scope for v1.
