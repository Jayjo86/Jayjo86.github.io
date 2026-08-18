---
title: "MCP Gateway 3/3: What it takes to run one"
description: "The honest accounting for a reference MCP gateway: the deliberate simplifications, what the newer protocol revision changes, and everything still missing before production."
summary: "The honest accounting for a reference MCP gateway: the deliberate simplifications, what the newer protocol revision changes, and everything still missing before production."
date: 2026-08-18T11:13:08+02:00
draft: true
tags: ["mcp", "gateway", "architecture", "security", "operations", "opa"]
series: ["MCP Gateway"]
series_order: 3
series_weight: 3
series_title: "What it takes to run one"
cover:
  image: "images/thumb-03-what-it-takes.png"
  alt: "The honest accounting for a reference MCP gateway: the deliberate simplifications, what the newer protocol revision changes, and everything still missing before production."
  hiddenInSingle: true
---
*Part 3 of three, on a reference MCP gateway built with FastMCP, OPA and Microsoft Entra ID. This part is the honest accounting: where the reference stops, and what a real deployment still needs.*

{{< series-nav >}}

---

## Where we are

[Part 1]({{< relref "posts/mcp-gateway-01-why-you-need-one.md" >}}) built the control point: an OAuth 2.1 authorization server to MCP clients, a confidential client to Entra behind it, no token passthrough, a policy decision before any downstream credential exists, one audit row per call. [Part 2]({{< relref "posts/mcp-gateway-02-three-layers-of-policy.md" >}}) covered the decision itself — a three-layer Rego bundle where the platform team decides which agent may connect, domain teams decide which users reach which backend, and tool owners decide what a specific action requires.

Both of those parts describe things the gateway does. This one describes things it doesn't, and I think it is the most useful of the three.

A reference implementation is only worth learning from if it is honest about its edges. It is built to be read, so the simplifications are part of the lesson: each one marks a decision a real deployment has to make deliberately, and every one of them is a place I have watched a demo quietly pretend the problem doesn't exist. If you are building your own gateway, this is your backlog. If you are buying one, these are the questions to ask the vendor.

---

## The deliberate limitations

**Elevated tools are gated by admin-assigned app roles, not runtime step-up.** Each elevated tool declares a `required_role` — a gateway app role such as `Ledger.Write` — matched against the `roles` claim of the gateway-audience token. A principal without it is denied and the tool is hidden from `tools/list`; it becomes callable once an administrator grants the role in Entra. There is deliberately no interactive consent prompt at call time.

This is the subtle part almost everyone gets wrong, so it is worth spelling out. A naive design checks a *backend* resource scope against the *gateway* token. Entra issues scopes relative to a token's own resource, so a backend scope can never appear in a gateway-audience token and the check either always fails or, worse, gets "fixed" by removing it. App roles live on the gateway app, so they land in the token you already validate.

**M2M and OBO are not interchangeable.** The profile is per backend and `obo` is the default. Switch a backend to `m2m` and its calls run as the *gateway's service principal*, which costs you two things: per-user data authorization at the backend (row-level filtering has nothing to filter on) and backend-side attribution. The human is still recorded at the gateway, but the backend only ever sees a service account. M2M is the right choice for a backend with no per-user data model and the wrong choice everywhere else. "Acts as the user end to end" needs `obo`, and if you flip a backend to `m2m` for convenience you have quietly retracted that claim.

**`tools/list` mints on `.default`, not the per-tool scope.** The credential prefetch [Part 2]({{< relref "posts/mcp-gateway-02-three-layers-of-policy.md" >}}) describes happens before any tool is known, so the least-privilege story has a hole on the path that runs most often. The token is cached and the backend still validates audience, but it is broader than any single call needs.

**State is per-process, and half of it is on disk.** The downstream token caches are in-memory — bounded, TTL-evicted, with single-flight so a burst of concurrent calls fires one Entra exchange rather than N. The OAuth session store is not in memory: pass no `client_storage` and `AzureProvider` falls back to an encrypted on-disk file store (Fernet over a file tree), in a directory named after a fingerprint of the session signing key.

That default is better than it sounds and worse than you need. Better, because a plain `docker compose restart` keeps every session — nobody has to re-authenticate. Worse, because it lives in the container's writable layer with no volume behind it, so recreating the container throws it away, rotating the signing key orphans it, and two replicas share nothing at all. Redis-backed, encrypted state is the production fix for both halves.

**Resources and prompts are denied, not modeled.** Reading a resource or fetching a prompt is fail-closed-denied with an audit row, because the policy bundle is tool-centric. That is the right default and a real capability gap: this gateway currently cannot front a backend whose value is in its resources.

**The audit write is best-effort.** A failed write is logged loudly and never masks the tool result. That is deliberate — the side effect may already have happened on the backend, and raising would lose the client's result *and* still not produce a record. But it does mean the compliance guarantee today is "we almost always have a row", not "we always do", and those are different sentences in front of a regulator.

**The role check for `tools/list` exists twice.** Rego decides it at call time, and the gateway re-checks `required_role` in Python to filter the list. The second copy is deliberately UX-only, and the drift it can produce is a hidden-but-callable tool, never an unauthorized one. It is still one rule with two implementations. Asking OPA for a batch decision over the candidate set would collapse them, at the cost of a round trip on every list. It stays on the list.

None of these make the pattern wrong. They are the difference between a reference you learn from and a service you run.

And one thing I would push back on if someone proposed it: **don't make the agent allowlist your primary control.** It is tempting, because it is the one that feels like it is stopping the scary new thing. It is a bearer-token allowlist. It governs who was issued a session, not who is holding one. The load-bearing controls are the Entra login with your Conditional Access on it, the user's app roles, and the consent interstitial.

---

## The spec moved while I was building this

Everything in this series implements the **2025-11-25** authorization spec. A newer revision, **2026-07-28**, supersedes it. I published against the older one on purpose, and the reasoning is worth stating because you will face the same choice on whatever schedule MCP decides.

Support for the new revision lands in FastMCP 4, which is still beta. Pinning `4.0.0b2` in a reference implementation *about enterprise security* is the first thing a skeptical reader would screenshot, and beta-to-stable churn means paying the migration twice. So: be correct against the revision that has a stable implementation, and be honest about the delta.

**The architecture turns out to be revision-independent**, and I checked rather than assumed — `fastmcp==4.0.0b2` in a throwaway venv, against the actual API surface. `AzureProvider`, the `jwt_issuer` property the agent-identity fix depends on, `FastMCPProxy`, `mount(namespace=…)` and every middleware hook this gateway uses all survive untouched. Three identities that must not be conflated, no token passthrough, decide-before-mint, one audit row per call: none of it is touched by the new revision. The migration is not a port.

What changes is which surfaces the enforcement point has to cover, and all three are gateway problems the library will not solve for you.

**`server/discover` becomes mandatory**, and clients may call it first. Right now it is the one method that would reach a backend with no policy decision and no audit row. The single-chokepoint design from [Part 1]({{< relref "posts/mcp-gateway-01-why-you-need-one.md" >}}) is what makes fixing that a new `Operation` rather than a new code path.

**`subscriptions/listen` replaces the GET stream and `resources/subscribe`.** FastMCP 4 implements it in the SDK layer with no middleware hook, so it routes around the PEP entirely — straight past the fail-closed position on resources. The exposure is bounded, since notifications carry a URI rather than content, but "nothing reaches a backend unaudited" has to be a decision rather than an omission. The honest interim answer is blocking it at the ASGI layer until a hook exists.

**Multi Round-Trip Requests replace sampling, elicitation and roots.** A call can return `input_required` and be retried carrying the human's answer. That breaks two audit assumptions at once. One logical operation becomes N calls with N unrelated trace IDs, so the record fragments exactly where a human supplied input — the most interesting part of the record for NIS2 or DORA. And the argument hash covers `arguments` only, so a retry with materially different human input hashes identically to the original.

That last one generalises well past MCP, and it is the piece I would tell someone about at a conference: **an audit design assumes a request is one round trip until the day it isn't.** If your compliance story rests on one row per call, a protocol that turns one operation into a conversation quietly turns your evidence into confetti. The fix is an operation-level correlation ID distinct from the per-request trace ID. The reason to know about it now is arithmetic — retrofitting an ID into an existing audit table is a migration, adding it up front is a column.

One thing gets easier. 2026-07-28 formally deprecates Dynamic Client Registration in favour of CIMD, with a twelve-month window. The policy layer already treats a verified CIMD URL and a self-registered DCR UUID as different grades of evidence, so CIMD stops being an argument I have to make and becomes the one the spec makes for me.

---

## What's missing for production

A consolidated checklist, grouped so you can hand each group to the right team.

### State and scale

- Shared session and token store (Redis plus Fernet encryption), and stateless HTTP mode for horizontal scale and mid-session replica failover.
- SSE-safe ingress (buffering off, long timeouts), exact-origin CORS, rate limiting, request-size limits, HA replicas, a load test.
- **Budget for the OPA coupling.** Fail-closed means the PDP sits in the critical path: OPA down is gateway down. That is the correct security posture and a genuine availability dependency, so size, monitor and alert on it like one instead of discovering it during an incident.

### Identity and authorization correctness

- Keep authorization on gateway **app roles** rather than downstream scopes or raw directory groups. Both alternatives have sharp edges — an audience mismatch that can never match, and a claim that silently disappears past 200 groups.
- Real OAuth step-up (`401`/`403` with `WWW-Authenticate: insufficient_scope`) for interactive elevation.
- Conditional Access on the gateway app. Consider DPoP or token binding, so a stolen session JWT isn't enough on its own.

### Rego policy lifecycle

Policy is code that changes who can do their job. It needs the same lifecycle as the gateway itself, and it is the part most teams retrofit after their first self-inflicted outage.

- **Ownership and review.** CODEOWNERS per layer: the platform team owns `platform.rego`, each domain team owns its slice of `domain.rego`, tool teams own their `tool.rego` sections. A domain team must not be able to merge a change to the platform layer. Because the three layers are a conjunction no team can widen another's grant, and that property is exactly what makes distributed ownership safe — so protect it in review.
- **CI on every policy PR.** `opa test policy/bundle policy/tests` as a required check, plus `opa fmt --diff` and `opa check --strict` to catch the undefined-versus-false traps from [Part 2]({{< relref "posts/mcp-gateway-02-three-layers-of-policy.md" >}}) before they ship. Coverage on the Rego, so a new rule with no test is visible.
- **Contract tests against the application.** Policy and code share a vocabulary of tool names, role names and server names. Test that they agree — the repo does this for the registry and `data.json` — and extend it to the Entra app roles actually defined in the tenant.
- **Distribution from git, not from a volume mount.** OPAL, an OPA bundle server, or a signed bundle in object storage. Turn on bundle signature verification on the OPA side so a compromised distribution path can't rewrite policy. Version every bundle and record the version in the audit row, so "what rule denied this?" is still answerable six months later.
- **Staged rollout with a shadow mode.** The single biggest gap in what I built: there is no way to ship a tightened policy in dry-run and see what it *would* have denied. Add a decision mode that evaluates the new bundle alongside the live one and logs disagreements without enforcing. Without it every tightening is a leap, and the safe-feeling move — never tighten — is how policy rots.
- **Decision logs to the SIEM.** OPA's decision log is a different record from the gateway's audit row and complements it: it captures the full input and the rules that fired. Ship it, retain it, and alert on deny-rate spikes. A sudden wave of denials is either an attack or a botched rollout, and you want to know which one within minutes.
- **A break-glass path.** A documented, audited, time-boxed way to grant an exception when policy is wrong at 3am, and an alert when it is used. Teams without one edit `data.json` on the running container and never tell anybody.
- **Data separate from rules.** `data.json` — which servers, which roles, which caps — changes weekly. The `.rego` files, which decide how decisions compose, change rarely. Different review bars, ideally different pipelines.

### Onboarding a new MCP server

Adding a backend currently touches six places across three repos and two systems. That is fine for two servers and does not survive twenty. Build the paved road before you need it.

What a team goes through today:

1. **Entra app registration** for the new server — Application ID URI, `requestedAccessTokenVersion: 2`, delegated scopes for OBO (`Ledger.Read`, `Ledger.Write`), or an Application-type app role for M2M.
2. **Grant and admin-consent** those scopes to the gateway's app registration.
3. **Gateway app roles** for the new domain (`ledger.read`) and any elevated tools (`Ledger.Write`), assigned to users or to a group mapped onto the role.
4. **Gateway config** — URL, audience, app-ID GUID, scope, downstream profile.
5. **Policy data** — `enabled_servers`, `server_roles`, and a `tools` entry per tool with its required role and argument constraints.
6. **Tool registry** — regulatory tags (NIS2 / DORA / AI Act) and the least-privilege downstream scope suffix, per tool.

What it should become:

- **A registration request as a pull request.** One manifest per MCP server — audience, profile, tools, governance tags, requested roles — that generates the gateway config and the policy data entries. Review then becomes a single diff a security reviewer can actually read, instead of six diffs across three repos.
- **Automate the Entra side.** The app registration, the scope definitions and the gateway grant are all Terraform, Bicep or Graph API calls. Hand-clicking them is where `api://`-versus-GUID audience mismatches and missing `requestedAccessTokenVersion: 2` come from. Those two cost me more time than anything else in this project, and both surface as opaque runtime errors rather than setup errors.
- **A conformance check before a server goes live.** Automated: does it reject a token minted for another backend? Does it return 401 with RFC 9728 metadata that resolves? Do its declared tools match what it serves? Does every tool carry governance metadata? A server that fails any of these should not be mountable.
- **Named owners and a review cadence.** Every server and every tool needs an owning team in CODEOWNERS, a regulatory classification made deliberately rather than defaulted, and a periodic re-attestation of who holds which app role. Regulators ask who approved this, and "it was in the config" is not an answer.
- **A deprecation path.** `enabled_servers` gives you a kill switch; the process around it is what's missing — announcement, grace period, an audit-log check for remaining callers, removal. Decommissioning is the step everyone leaves out of the paved road and then does by hand under time pressure.

### Audit and compliance

- Tamper-evident storage: hash chain, signed receipts, append-only, or WORM.
- Guaranteed delivery through an outbox or queue, so a database outage never drops a record. Today a failed write is logged, not retried.
- Retention enforcement as a scheduled job (the AI Act's floor is six months).
- Remember what this is. An *authorization* gateway is not a compliance solution. DLP and content scanning, model governance, HA/BC/DR, DPIA, conformity assessment, third-party risk and human-review workflows are all out of scope by design, and pretending otherwise is how a control point becomes a compliance theatre prop.

### Operational hardening

Pin every dependency. Reproducible, non-root container builds. Real secret management rather than a `.env` file. Real TLS at the ingress. A signing-key rotation runbook. And an owner for the perpetual MCP-spec-tracking tax, because it is a standing cost, not a project.

---

## Closing

Across three parts the thesis has been the same: **the plumbing is a library, the judgment is yours.** FastMCP handles the MCP protocol, transport, aggregation and the OAuth surface. Entra ID handles identity. OPA handles the decision. What you build and own is the enforcement point, the per-backend token exchange and the audit record — the parts that encode *your* enterprise's rules and *your* regulator's expectations. That is also the part a product cannot buy you, which is why this series is written as a blueprint rather than a deployment guide.

Of the three parts, this is the one I would keep if I could only keep one. A gateway that decides correctly is a weekend of reading. A gateway you can operate is policy lifecycle, an onboarding path teams will actually follow, an audit trail that survives a protocol change, and somebody whose job it is to track a spec that moves every few months. The code is the easy half, and the list above is why.

An MCP gateway isn't optional infrastructure for a serious enterprise rollout. It is the difference between "agents can call our systems" and "agents can call our systems, as the right user, only when allowed, with a record we can show a regulator." The point of building this one in the open was to show that the whole control point — the OAuth broker, the PEP, both token exchanges, the audit writer — fits in under two thousand lines of readable Python. Small enough to hold in your head, which is the only way anyone ends up trusting it. Provided, always, that you also understand where it stops.
