---
title: "MCP Gateway 2/3: Three layers of policy"
description: "Splitting MCP authorization across three Rego layers with three owners — platform, domain team, tool owner — so that no team can widen another team's grant."
summary: "Splitting MCP authorization across three Rego layers with three owners — platform, domain team, tool owner — so that no team can widen another team's grant."
date: 2026-08-18T11:12:43+02:00
draft: true
tags: ["mcp", "gateway", "architecture", "security", "authorization", "opa", "rego"]
series: ["MCP Gateway"]
series_order: 2
series_weight: 2
series_title: "Three layers of policy"
repo: "https://github.com/Jayjo86/mcp-gateway-reference"
cover:
  image: "images/thumb-02-three-layers.png"
  alt: "Splitting MCP authorization across three Rego layers with three owners — platform, domain team, tool owner — so that no team can widen another team's grant."
  hiddenInSingle: true
---
*Part 2 of three, on a reference MCP gateway built with FastMCP, OPA and Microsoft Entra ID. This part is the policy design — the part I would carry into any implementation, on any substrate.*

{{< series-nav >}}

---

## Where we are

[Part 1]({{< relref "posts/mcp-gateway-01-why-you-need-one.md" >}}) built the control point: an OAuth 2.1 authorization server to MCP clients, a confidential client to Entra ID behind it, no token passthrough, and a decision from an OPA sidecar *before* any downstream credential comes into existence.

This part is the decision itself, and it is the most portable thing in the series. The Rego below is specific to OPA, and the roles are specific to Entra, but the shape — who owns which rule, and what happens when two owners disagree — survives a change of both. If you are evaluating a vendor's gateway rather than building one, this is the section to read with their docs open next to it.

One thing from Part 1 is load-bearing here, so it is worth one more line. **Three identities ride on every request**: the *actor* (the human, from Entra's `sub` / `upn` / `roles`), the *agent* (the MCP client program — Claude Code, Cursor, a script — identified by the client ID this gateway's own OAuth server issued at registration), and the *broker* (this gateway, Entra's `azp`, a constant on every request ever made). Collapsing the last two is the easiest way to build a gateway that looks like it authorizes and doesn't. The policy below decides on the first two, separately and for different reasons.

---

## The policy: three Rego layers, three owners

The bundle is split by **who owns the rule**, not by what the rule checks. Each layer is a separate file with its own CODEOWNERS entry, and a tool call resolves to a single decision:

```rego
default allow := false

allow if {
	platform_allow   # platform team
	domain_allow     # domain team
	tool_allow       # tool team
}
```

Three teams ship independently, and no team can widen another team's grant, because a conjunction makes every layer a veto. That is the whole trick, and it is worth being deliberate about: the moment someone adds a fourth rule joined by `or` to unblock a launch, the property is gone and nobody will notice for months.

All the facts the layers decide against live in one `data.json` that ships with the bundle.

### Layer 1 — platform: which agent may talk to this gateway at all

This is the supply-chain layer. It answers "is Claude Code allowed here, or only our internally vetted client?" and says nothing about the human.

```rego
# Absent key → "audit". An unrecognised value denies everything with an explicit
# reason, so a typo fails loudly instead of silently disabling the control.
_enforcement := object.get(data.platform, "agent_enforcement", "audit")

# "audit" — the agent id reaches the PDP and the audit row and gates nothing.
agent_ok if _enforcement == "audit"

# "allowlist" — opt in to the supply-chain control.
agent_ok if {
	_enforcement == "allowlist"
	agent_trusted
}

agent_trusted if {
	input.agent.kind == "cimd"
	input.agent.id in data.platform.allowed_agents
}
```

```json
"platform": {
  "agent_enforcement": "allowlist",
  "allowed_agents": ["https://claude.ai/oauth/claude-code-client-metadata"],
  "allowed_dynamic_client_ids": []
}
```

Two decisions here that I would make the same way again.

**The default is "record it, don't gate on it."** Authorization in this gateway is the *user's*. The domain and tool layers both decide on Entra-signed `roles`, and that is the load-bearing control; it works completely without this layer. Agent identity is primarily attribution — which client software did this, on every audit row — which is a NIS2 and DORA reporting obligation rather than an authorization one.

Gating on it is a real supply-chain control, but a coarse one. The session token is a bearer token, so an allowlist governs who was *issued* one, never who is *using* one. The control that fires at the right moment against a hostile client is the OAuth consent interstitial, not this rule. Turning the allowlist on is a reasonable thing to want; believing it is your primary defense is not.

**CIMD entries scale, DCR entries don't.** Allowlisting a CIMD URL covers every installation of that client software everywhere. Allowlisting a DCR UUID covers one machine until it re-registers. Both are supported, and there is a test in [the repo](https://github.com/Jayjo86/mcp-gateway-reference) that fails if someone drops a non-CIMD string into `allowed_agents`, where it could never match anything.

The deny reason names the exact string to paste into `data.json`:

```
agent "9f2c1a4e-..." (dcr) is not on the platform allowlist
```

That turns "turn enforcement on" from a log-diving exercise into self-service, which matters more than it sounds: the moment this layer stops being a tautology, it starts denying, and the person it denies first is usually you, on a Friday.

### Layer 2 — domain: which users may reach which backend

Owned by the team that runs each MCP server. This is where the human's entitlements enter.

```rego
domain_allow if {
	server := data.tools[input.tool].server
	server in data.domain.enabled_servers
	some r in input.actor.roles
	r in data.domain.server_roles[server]
}
```

```json
"domain": {
  "enabled_servers": ["mcp-server-a", "mcp-server-b"],
  "server_roles": {
    "mcp-server-a": ["crm.read"],
    "mcp-server-b": ["ledger.read"]
  }
}
```

Coarse on purpose: it is a door, not a lock. Holding `crm.read` gets you *to* the CRM backend, and what you may do once you are there is the next layer's problem. A domain team can also disable its whole server through `enabled_servers` without touching anyone else's rules, which is where change-freeze windows and environment gates end up living naturally.

**Authorize on Entra app roles, not raw directory groups.** This is the trap I sidestepped deliberately, and it is worth the paragraph because it fails in the worst possible way. Entra drops the `groups` claim entirely once a user is a member of more than 200 groups in a JWT access token (150 in SAML), and replaces it with a Graph pointer to fetch the list. Policy that authorizes on groups therefore silently denies your most heavily grouped users — who tend to be your most senior ones — and only in production, and only for some of them. App roles never overflow. If you want group-based administration, assign a directory *group to an app role* in Entra and let the token carry the small role value instead.

**This layer also gates `tools/list`.** A list operation has to decide whether to mint a downstream credential for a whole backend *before* any specific tool is known, so it can't route through `domain_allow`, which looks the server up via the tool. A parallel `list_allow` rule takes `input.server` directly and folds in the platform check:

```rego
list_allow if {
	_server_enabled
	_has_role_for_server
	agent_ok
}
```

Without it, `tools/list` is a free pass. A principal holding zero app roles still triggers a full round of Entra token exchanges, unaudited, on the path that runs most often. Enumeration is an authorization decision, and treating it as plumbing is how gateways end up with a hole in the exact place their diagram claims to be a chokepoint.

**It is also where the catalogue gets reconciled.** "Discover the tools from the backends" is usually proposed as something the gateway does at *its own* boot, and that doesn't work. At boot there is no user, so discovery has to use client credentials — and a backend running the `obo` profile may have no app-role assignment for the gateway at all. You would be forcing an M2M grant onto every backend purely so the gateway can read a list.

The discovery call you actually want is the one the **agent** already makes. A client connects, sends `tools/list`, the gateway fans out to each mounted proxy, and each backend answers with its real tool set — with a user token behind it, so OBO works. By the time the middleware has that list in hand it *is* the discovered catalogue, and reconciling it against the governance overlay costs one comparison:

- a tool the registry and `data.json` both know about is listed, subject to the role check below;
- a tool they don't know about is **dropped**, and named once at WARNING.

Dropping it is the point. Before that change an ungoverned tool was listed and then denied as unknown when someone called it, which meant the gateway advertised a capability it would always refuse. Fail-closed and silent is still a bug. Fail-closed and loud is a control.

### Layer 3 — tool: what this specific tool requires

Owned by whoever owns the tool. Two kinds of constraint.

**An elevated app role**, for tools that need more than "you may reach this backend":

```rego
role_ok if _configured_role == ""

role_ok if {
	_configured_role != ""
	_configured_role in input.actor.roles
}
```

**Content-based constraints on the actual arguments**, so policy can cap an amount or restrict an enum:

```rego
args_ok := false if {
	input.tool == "ledger_post_entry"
	_ledger_amount_present
	is_number(input.args.amount)
	abs(input.args.amount) > data.tools.ledger_post_entry.max_amount
}
```

```json
"ledger_post_entry": {
  "server": "mcp-server-b",
  "required_role": "Ledger.Write",
  "max_amount": 1000000
}
```

Note the `abs()`. A large *negative* posting is exactly as significant to a ledger as a positive one, and a cap written as `amount > max` waves it straight through. Half the value of putting a constraint like this in Rego rather than inside the tool is that it becomes reviewable by someone who is not the tool's author, and this is the kind of thing that reviewer catches.

**But content-based policy needs real values, and real values are the problem.** Sending every argument to the PDP means customer names, memos and amounts cross a process boundary on every single call. That is defensible for a co-located sidecar and indefensible the moment a platform team centralises OPA, which is what happens in year two.

So each tool declares the fields its policy actually reads, and nothing else is sent:

```python
"ledger_post_entry": ToolMeta(
    ...
    required_role="Ledger.Write",
    downstream_scope="Ledger.Write",
    policy_args=("amount",),   # tool.rego caps amount at max_amount
)
```

An allowlist, not a denylist, so adding an argument to a tool cannot silently start leaking it. The projection happens inside `build_input`, which is the single place an argument can reach the PDP, rather than trusting each caller to pre-filter. The audit row still hashes the *full* argument set, because the hash has to identify the call that actually ran.

It fails in the safe direction too. Forget to declare an argument the Rego constrains and it arrives absent, so the presence check denies — a confusing denial rather than a silently unenforced cap. And because both artifacts live in the same repo, a test parses `input.args.X` out of `tool.rego` and asserts the registry declares it, in both directions: nothing constrained that isn't sent, nothing sent that isn't constrained.

### The Rego trap that cost me an afternoon

There is a subtlety here I would bet most content-based policies get wrong, because the intuitive fix and the correct fix look identical. **You cannot fail closed on a missing argument by negating a builtin.**

Given an absent `input.args.amount`:

- `input.args.amount > max` is undefined, so the rule never fires and `default args_ok := true` quietly wins. Most people spot this one.
- `not is_number(input.args.amount)` looks like the fix. It isn't. It also never fires.

The second one is the surprise, because everyone knows Rego's `not` succeeds on an undefined expression — `not input.args.amount` on an absent key really is true. The difference is the builtin call. OPA rewrites a ref inside a negated expression into an assignment hoisted *outside* the `not`, roughly `__local0__ = input.args.amount; not is_number(__local0__)`. The assignment is undefined, so the body is undefined before the negation is ever evaluated, and the rule contributes nothing.

You can watch it happen in about thirty seconds:

```rego
package trap

default b_ok := true

b_ok := false if not is_number(input.args.amount)
```

```console
$ echo '{"args": {}}' | opa eval -d trap.rego -I -f raw 'data.trap.b_ok'
true      # the constraint did not fire
```

The fix is to check presence over a concrete key set first, because membership is always defined whether the key is there or not:

```rego
_ledger_amount_present if "amount" in object.keys(input.args)
```

Miss it and you have a policy that reads like a cap and enforces nothing whenever the argument is omitted — which is precisely the input an attacker sends. Deny-by-default at the *rule* level does not give you deny-by-default at the *field* level, and that surprises people who have been writing Rego for years.

### Reporting the right denial

One more detail, with a wildly disproportionate impact on the people operating this thing. When the tool layer denies, the gateway needs to know *why*, so it can raise a `MissingEntitlement` naming the role to request. But `required_role` must only be exported when the role check is what actually failed:

```rego
default required_role := ""

required_role := _configured_role if not role_ok
```

Export it whenever a tool merely *has* a configured role, and an args-only denial — an amount over the cap — tells an administrator to grant a role the caller already holds. Someone then spends an afternoon in the Entra portal fixing a problem that was never about entitlements, and in the worst version of this they grant a write role to fix what was actually a read problem. Bad error messages are a privilege-escalation vector; this is the cheapest place to prove it to yourself.

---

## Next

Three layers, three owners, one conjunction, and no team able to widen another team's grant. That is the design I would keep, and the one I would hold a product to.

What I have not done is tell you where it stops. [Part 3: What it takes to run one]({{< relref "posts/mcp-gateway-03-what-it-takes-to-run-one.md" >}}) is the honest accounting — the deliberate simplifications, the protocol revision this targets and what the newer one changes, and the consolidated list of everything still missing for production.
