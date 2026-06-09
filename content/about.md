---
date: '2026-06-08T16:02:17+02:00'
draft: false
title: 'About'
---

### Who am i

I'm Jürgen Neulinger. I build the infrastructure other people's AI systems run on — agent frameworks, RAG, and the plumbing in between. Right now I lead a generative-AI platform team at Erste Digital, where "it works in the demo" and "it survives an audit" are two very different bars. The second one is the job.

My pattern is to build a foundation to the point a team can own it, then move on to the next one. A few of them: the central capability for provisioning and governing the LLMs used at the bank, now run by the Databricks platform team; a config-driven RAG framework where onboarding a new use case is an index and a little auth config, not new pipeline code; an FSM-based agent framework; and an MCP gateway with real enterprise auth — Entra ID, per-tool OPA/Rego authorization, OAuth 2.1 On-Behalf-Of.

Before any of the AI work, I spent a decade in regulatory reporting and credit risk at tier-1 banks — COREP, large exposures, MREL — systems where "the model was probably right" is not a sentence you get to say in a review. That's where most of my opinions come from. I care about determinism, auditability, and being able to put a breakpoint on whatever just made a decision.

### What I write about here

LLM and agent systems built framework-light: state machines, schemas, and narrow model jobs instead of large orchestration frameworks.
Identity, authorization, and governance for AI tooling in regulated environments — Entra ID, OPA, MCP, and the rest of the unglamorous plumbing that actually lets you ship.
The recurring claim that most "AI engineering" is just engineering, and the hard part is the spec, not the model.

My bias, stated up front: I'd rather hand-roll a transparent state machine I can debug at 3am than inherit abstractions I can't see through. I'm occasionally wrong about this — the posts that admit it are usually the better ones.
When I'm not doing that, I'm on a road bike or a pair of skis somewhere in the Alps.

### Elsewhere

LinkedIn: [linkedin.com/in/juergen-neulinger](https://www.linkedin.com/in/juergen-neulinger)

Email: [j.neulinger@gmx.at](mailto:j.neulinger@gmx.at)
