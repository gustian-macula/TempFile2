# NEMESIS Live Scan — Claude Code implementation prompt

This supersedes the earlier "App Discovery catalog + per-finding Validate" flow. NEMESIS Discovery is now a **crawler** that feeds **automatic** validation; there is no per-finding Validate button. Everything else we agreed stands: deterministic-first engine, a **Dynamic Validation Catalog** with a promotion pipeline, the **Validation Agent configured under Product Settings → AI → NEMESIS Agents**, Interact + Record/Replay with session-auth, and AEGIS hand-off into a **NEMESIS section inside AEGIS**.

Paste everything in the block below into Claude Code. Fill the STACK line first.

```
Build the NEMESIS Live Scan feature for our Macula security platform (modules: ARES, AEGIS,
HORUS, NEMESIS). Match the existing platform look and feel: dark navy theme, module
top-switcher, NEMESIS accent amber (#e6a52c), severity colors crit #e8563f / high #ef8f3c /
med #e0b83a / low #3fb6d9, proven #3fb98a, LLM/synth #8b7cf0, crawl/horus #3fb6d9. Reuse our
shared UI components and conventions.

STACK: <fill in — e.g. React + TypeScript + our design system; Node/Express + PostgreSQL +
Prisma; a job/queue runner for long scans (BullMQ/Temporal/etc.)>.

=== CORE MODEL CHANGE (read first) ===
A NEMESIS engagement runs ONE continuous SCAN with two phases that overlap in time:
  1. DISCOVERY = an authenticated or unauthenticated CRAWLER that maps the app's resources
     (URLs, endpoints, forms, params, API operations) into a resource tree. It persists the
     request+response and an extracted signature for every resource.
  2. VALIDATION = runs AUTOMATICALLY and continuously: as the crawler emits a resource that
     has testable inputs, validation is triggered for it immediately. THERE IS NO per-finding
     "Validate" button and NO "Validate All". The only run controls are PAUSE/RESUME and
     RESTART (single continuous scan; no per-branch rescan for now).
Discovery results appear FIRST (the tree populates before validation confirms anything), so the
default view is the Discovery Tree.

DETERMINISTIC-FIRST (unchanged, do not violate): validation maps each resource's signature to a
scenario in the Dynamic Validation Catalog and runs it deterministically. An LLM agent is
invoked ONLY when (a) no catalog scenario matches (synthesize a new one on demand) or (b) false-
positive triage is needed. Every LLM-synthesized scenario that proves a vuln is promoted to the
catalog as a permanent deterministic check, so the LLM is never used twice for the same
signature. Log every LLM call (model, tokens, cost, tenant, escalation_reason, promoted?) for
per-tenant attribution.

=== CRAWLER (Discovery) ===
- Modes: authenticated (seeded from an auth_profile, see AUTH) or unauthenticated.
- Settings per scan:
    * depth (int, DEFAULT 5) = max crawl-link distance (hops) from the seed URL(s).
    * extensions/routes to hunt = configurable set, default on: html, php, aspx, jsp, js, json,
      do, action, cgi, /api/, .env, /graphql — cover common Node/Java/.NET/PHP frameworks.
      Treat these as "fetch & analyze if the discovered URL matches" filters, extensible.
    * mode (authenticated | unauthenticated).
- Crawl respects the engagement's attack_policy scope (allowed hosts/paths) and an authorization
  gate before any traffic.
- For each discovered resource, persist: url, method, parent (by longest path-prefix match),
  depth, discovered inputs/params, request, response, latency, extracted signature.
- Emit resources incrementally over a stream/websocket so the UI tree grows live.

=== AUTO-VALIDATION ===
- A resource with testable inputs enters the validation queue the moment it's discovered.
- Validator (deterministic): match signature → Dynamic Validation Catalog scenario → execute its
  probes → evaluate deterministic success_condition → PROVEN (capture evidence) | REJECTED.
- No match → LLM escalation: FP triage; if plausible, synthesize a scenario, execute it live,
  and on success PROMOTE it to the catalog (promotion pipeline below). FP → prune (pruned
  findings are kept as informational, excluded from proven counts).
- Finding states: potential → validating → (proven | dismissed_fp). (synthesizing is a
  sub-state of validating when the agent is engaged.)
- Emit validation state changes on the same stream so the UI updates nodes + proof feed live.

=== DYNAMIC VALIDATION CATALOG + PROMOTION (unchanged from prior spec) ===
- validation_scenario: key (unique, e.g. idor.crosstenant.seqid.v1), vuln_class, match
  (signature selector), probes[] ({method,path,headers,body,inject,success_condition}),
  success_condition (deterministic: time_delta | reflection-of-benign-canary | body_diff
  foreign-marker | version_match | boolean_diff), severity_hint, safe (must be true; reject
  destructive/DoS/exfil), source (seed|llm_synth|forge_recorded), tenant_scope (global|tenant),
  status (active|pending_review|disabled), version, origin_finding_id, created_by.
- Promotion pipeline: agent returns scenario JSON → engine executes it live → only if
  success_condition is met, assign stable key (bump version on conflict), schema+safe validate,
  dedupe by match-signature hash, write to catalog (active if auto-promote for llm_synth tenant
  scope else pending_review), emit catalog.scenario.promoted event (UI toast + counter). Same
  pipeline accepts forge_recorded scenarios from the recorder. llm_synth starts tenant-scoped;
  reviewer can promote tenant→global.
- AUTO-PROMOTE POLICY: <choose one — "auto-promote llm_synth scenarios to active for the
  discovering tenant" OR "land them in pending_review for human approval first">. Make it a
  setting on the NEMESIS Agent config.

=== NEMESIS AGENTS CONFIG (Product Settings → AI → NEMESIS Agents) ===
The Validation Agent is registered/configured here (NOT an "Agents Factory"). Model a
nemesis_agent entity: id, name, role (validator|fp_triage|…), provider, model, system_prompt,
system_prompt_version, guardrails (safe-only, max_probes), auto_promote (bool), tenant_id.
Provider-agnostic behind one interface (DeepSeek, Anthropic, etc.). System prompt is stored and
versioned; it requires strict-JSON output matching the scenario schema, deterministic machine-
checkable success conditions, safe proof markers only, no fabricated evidence.

=== AUTH (Settings → Authentication Profiles) ===
Credentials are never free-text on the scan form. auth_profile entity: id, name, type
(bearer | cookie_login | api_key | basic | oauth2_cc), config (token endpoint / login URL+form
fields / header name), secret_ref (secrets manager; never inline), tenant_id. The scan and the
recorder reference an auth_profile_id; secrets resolve at run time.

=== UI: LIVE SCAN SCREEN ===
Header: engagement name + target URL; run controls = Scan settings (popover: depth slider
default 5, extension/route chips, auth mode), PAUSE/RESUME, RESTART, and "Send validated →
AEGIS". A top PIPELINE strip with three live stages and progress rings:
  CRAWL (resources mapped, % coverage) → VALIDATE (inputs tested) → PROVEN (confirmed, + FP
  pruned count). Work visibly flows through the three stages as the scan runs.

Main area has TWO VIEWS (tabs, in this order):
  1. DISCOVERY TREE (default, shown first) — an indented hierarchical sitemap of every mapped
     resource, children nested under parent by path prefix, each node labeled with its path;
     proven findings annotated inline with title + CVSS. This is the "what's in the site" view.
  2. ATTACK SURFACE — a radial force/graph view of the same data: nodes = resources/inputs
     radiating from the seed root; nodes pulse while validating and light up by CVSS severity
     when proven; clean/FP nodes dim. This is the exposure-at-a-glance view. Because large apps
     produce hundreds of nodes, include from the start: filter by severity, filter/collapse by
     branch (subtree), and zoom/pan. Hover shows a tooltip (path, class, CVSS, state); clicking
     a proven node opens Interact.

Right rail: PROOF FEED — confirmed findings stream in as evidence cards the moment they're
proven (newest on top), each with severity, CVSS, catalog-vs-LLM-synthesized tag, path/class,
and actions: Interact, Evidence, → AEGIS. A small HUD shows status, depth, coverage %, LLM
escalations.

Both views and the proof feed are driven by the same live stream of scan events.

=== INTERACT + RECORD & REPLAY (unchanged) ===
Opened from any proven node or proof card (modal or drawer). Editable request pane + live
response pane + "Resend request" (modify → resend → observe). Recording is request/response-
driven: each Resend while recording appends a {request,response} step. Controls: Start recording
→ (active, red) → STOP recording → Save recording → Replay. Save persists steps as a
forge_recorded validation_scenario via the promotion pipeline; Replay re-executes them in order.
Auth persistence: carry a session context seeded from the engagement's auth_profile; inject
current cookies/bearer/api-key/basic into each step and capture Set-Cookie/refreshed tokens to
update it so the chain stays authenticated; store which values are session-derived vs fixed
(parameterize {{auth}}/{{session}}) so Replay re-establishes auth fresh, never a stale cookie.
Never store raw credentials; redact live tokens in saved evidence.

=== AEGIS HAND-OFF (into AEGIS's NEMESIS section) ===
Per-finding "→ AEGIS" and header "Send validated → AEGIS" (bulk). POST /aegis/detection-requests
{ source:"nemesis", finding_id, vuln_class, mitre_technique, evidence:{request,response}, target,
cvss, severity }. AEGIS currently has an ARES-fed section; ADD a NEMESIS section (sibling to
ARES) and route by `source` ("ares" vs "nemesis"); both consume the same endpoint, partitioned
by source. Detailed AEGIS NEMESIS-section UI is deferred. Persist a nemesis-side aegis_handoff
record.

=== REPORTING (unchanged) ===
Over CONFIRMED findings only (exclude potential + dismissed FP): summary cards (total validated +
per CVSS band), a Critical/High/Medium/Low bar chart by CVSS, and a donut of the distribution;
export PDF/CSV. Bands: Critical 9.0–10, High 7.0–8.9, Medium 4.0–6.9, Low 0.1–3.9; compute the
CVSS vector at validation time.

=== DATA MODEL ===
engagement(id, target, mode, auth_profile_id, attack_policy_id, tenant_id, status, created_at)
scan(id, engagement_id, status[idle|running|paused|complete], depth, extensions[], mode,
     coverage_pct, started_at, finished_at)
resource(id, scan_id, url, method, parent_id, depth, inputs jsonb, request, response, latency,
         signature, state[discovered])
finding(id, scan_id, resource_id, title, vuln_class, owasp, mitre, cvss, severity,
        state[potential|validating|synthesizing|proven|dismissed_fp], evidence_request,
        evidence_response, via[catalog|llm|forge], created_at)
validation_scenario(... as above ...)
auth_profile(... as above ...)
attack_policy(id, name, allowed_hosts[], allowed_paths[], allowed_vuln_classes[], intensity,
              tenant_scope)
nemesis_agent(... as above ...)
recording(id, finding_id, steps[jsonb {request,response,session_derived_fields}], auth_profile_id,
          status[recording|stopped|saved], scenario_id)
llm_usage_log(id, finding_id, tenant_id, escalation_reason, model, input_tokens, output_tokens,
              token_cost, signature_hash, promoted_to_catalog, created_at)
aegis_handoff(id, finding_id, source, mitre_technique, status, aegis_rule_id, created_at)

=== ENDPOINTS / STREAM ===
POST /nemesis/engagements
POST /nemesis/engagements/{id}/scan            (start a scan)
POST /nemesis/scans/{id}/pause
POST /nemesis/scans/{id}/resume
POST /nemesis/scans/{id}/restart
GET  /nemesis/scans/{id}/stream                (SSE/WebSocket: resource.discovered,
                                                validation.started, finding.proven,
                                                finding.dismissed_fp, scenario.promoted,
                                                scan.progress, scan.complete)
GET  /nemesis/scans/{id}/resources             (tree snapshot)
GET  /nemesis/scans/{id}/findings
POST /nemesis/findings/{id}/synthesize         (explicit on-demand scenario generation)
POST /nemesis/validation-scenarios             (+ list, /{id}/promote-scope, /{id}/disable)
POST /nemesis/recordings                        (+ /{id}/step, /{id}/stop, /{id}/save, /{id}/replay)
POST /aegis/detection-requests

=== BUILD ORDER ===
1. Schema + migrations; engagement/scan CRUD; Auth Profiles + Attack Policies.
2. Crawler (Discovery) with depth + extension filters + auth modes; persist resources; emit
   resource.discovered on the stream. Build the Discovery Tree UI against the stream first.
3. Auto-validation queue + deterministic Validator against the Validation Catalog; emit
   finding.* events; Proof Feed + node state in the UI.
4. LLM escalation (FP triage + on-demand synthesis) + promotion pipeline; NEMESIS Agents config
   screen under Product Settings → AI.
5. Attack Surface graph view (with severity/branch filters, collapse, zoom) over the same stream.
6. Interact + Record/Replay with session-auth.
7. AEGIS hand-off (NEMESIS section) + Reporting.
8. Wire llm_usage_log throughout for cost attribution.

Start with the schema + migrations and the scan event stream contract, then the crawler and the
Discovery Tree bound to the stream. Ask me before making stack-specific assumptions you're unsure
about (queue/runner choice, SSE vs WebSocket, graph library).
```

---

*Companion: the clickable `nemesis-scan-mockup.html` shows the exact target UX — pipeline strip, Discovery Tree (first) and Attack Surface views, Pause/Restart, Proof Feed, and Interact with Start/Stop/Save/Replay.*
