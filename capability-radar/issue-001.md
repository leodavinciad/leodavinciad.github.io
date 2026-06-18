---
layout: page
title: "Capability Radar #001 — MCP discovery is the bottleneck"
permalink: /capability-radar/issue-001.html
---

# Capability Radar #001 — MCP discovery is the bottleneck

**Status:** sample public issue / MVP validation artifact  
**Audience:** AI agent builders, automation operators, MCP users, and technical founders  
**Core question:** which agent capabilities are useful enough to test now?

## Top recommendation

### Test: Glama MCP Registry / Inspector / Gateway

**Category:** MCP registry, inspector, hosted gateway  
**Verdict:** **Test**  
**Why it matters:** Glama is attacking the capability discovery problem directly: browse MCP servers, inspect tools in-browser, and put a gateway in front of multiple servers with logging and access control. For builders, the useful bit is not only directory search; it is the try-before-install workflow and gateway concept.  
**Builder use case:** evaluate candidate MCP servers before adding them to an agent runtime; compare tool descriptions and schemas; test whether a server exposes the capability you actually need.  
**Friction:** external hosted platform; production use requires trust in gateway/security model.  
**Recommendation:** test with non-sensitive MCP servers first. Do not route private credentials or internal systems until the security model is reviewed.

Source: [Glama](https://glama.ai/)

---

## Test this week

### 1. Official MCP Registry

**Category:** MCP server registry  
**Verdict:** **Test**  
**Why it matters:** The official registry is the closest thing to a canonical MCP discovery surface. The modelcontextprotocol/servers GitHub repo now explicitly points users to the registry for published servers and clarifies that the repo is mainly for reference implementations.  
**Builder use case:** use it as the default first stop for MCP server discovery before searching random GitHub repos.  
**Friction:** registry coverage may lag the wider ecosystem; still verify individual server maintenance and security.  
**Recommendation:** make it the baseline source in the weekly monitoring pipeline.

Sources: [MCP servers repo](https://github.com/modelcontextprotocol/servers), [MCP Registry](https://registry.modelcontextprotocol.io/)

### 2. Agent Skills / `SKILL.md` standard

**Category:** Agent skill standard  
**Verdict:** **Test**  
**Why it matters:** Agent Skills package repeatable workflows as folders with `SKILL.md`, scripts, templates, and references. The pattern is spreading across multiple agent clients. This makes capabilities more portable than one-off prompt snippets.  
**Builder use case:** convert repeated workflows — code review, report generation, MCP setup, QA procedures — into reusable skills that load only when relevant.  
**Friction:** ecosystem discovery is immature; quality varies; skills can encode unsafe workflows if not reviewed.  
**Recommendation:** start by writing one internal skill for a painful repeated workflow, then evaluate public skill repos later.

Sources: [Agent Skills overview](https://skill.md/), [Agent Skills specification](https://skill.md/specification)

### 3. Smithery MCP directory

**Category:** MCP server directory / publishing surface  
**Verdict:** **Test**  
**Why it matters:** Smithery is one of the visible discovery and publishing surfaces for MCP servers. It is useful for finding servers by category and seeing how MCP publishers present themselves.  
**Builder use case:** discover candidate MCP servers and study listing metadata before deciding what to install or replicate.  
**Friction:** directory presence is not proof of maintenance, safety, or production quality.  
**Recommendation:** use for discovery, not trust. Pair every candidate with GitHub activity, docs quality, and local sandbox checks.

Source: [Smithery MCPs](https://smithery.ai/servers)

---

## Skip for now

### 1. Random MCP servers with weak docs

**Category:** MCP server long tail  
**Verdict:** **Skip**  
**Why it matters:** The ecosystem is large and duplicative. An MCP server with no clear install path, unclear credential handling, or stale commits should not be added to a real workflow just because the capability sounds useful.  
**Recommendation:** require minimum trust gates: recent maintenance, clear README, explicit auth model, local test path, and small blast radius.

### 2. Production routing through unreviewed hosted gateways

**Category:** MCP hosting / gateway  
**Verdict:** **Skip for production; test in sandbox**  
**Why it matters:** Gateways are powerful because they centralize access, logging, and credentials. That also makes them sensitive infrastructure.  
**Recommendation:** sandbox first; never route private company systems or paid API keys without a review.

### 3. “Agent marketplace” claims without real buyer workflows

**Category:** Agent marketplaces  
**Verdict:** **Skip**  
**Why it matters:** Many marketplace concepts are narrative-first. The useful near-term problem is not agents shopping autonomously; it is human builders discovering maintained capabilities with clean install and trust signals.  
**Recommendation:** watch marketplaces that solve discovery, verification, install, and billing friction — not just listing pages.

---

## Watch closely

### 1. x402 and agent-native paid APIs

**Category:** Payment/access protocol  
**Verdict:** **Watch**  
**Why it matters:** x402 uses HTTP `402 Payment Required` style flows to let agents pay for resources/API calls. It may become a useful primitive for agent-accessible paid capabilities.  
**Builder use case:** pay-per-call APIs, datasets, and microservices without traditional account signup.  
**Risk:** still early; wallet custody, spend limits, accounting, fraud, and compliance need careful handling.  
**Recommendation:** track ecosystem explorers and examples; do not put autonomous spend in production yet.

Sources: [x402scan](https://www.x402scan.com/), [Chainalysis x402 analysis](https://www.chainalysis.com/blog/x402-agentic-payments-adoption)

### 2. Google AP2 / agent payment authorization

**Category:** Agent payments / authorization protocol  
**Verdict:** **Watch**  
**Why it matters:** AP2 focuses on secure authorization and mandates for agent-led payments across payment methods. That is directly relevant to agent capability marketplaces if agents eventually purchase tools, data, or services.  
**Recommendation:** watch for developer-friendly implementations and MCP/A2A integration examples.

Source: [Google Cloud AP2 announcement](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)

### 3. Capability metadata quality

**Category:** Discovery infrastructure  
**Verdict:** **Watch**  
**Why it matters:** Tool descriptions, schemas, installation metadata, trust signals, and usage examples determine whether agents and humans can select the right capability. Discovery quality may become the real competitive surface.  
**Recommendation:** track registries that expose machine-readable metadata, quality scores, examples, and security notes.

---

## Builder experiment

Run this 30-minute test:

1. Pick one repeated workflow your agent performs.
2. Search the official MCP registry, Smithery, and Glama for one relevant capability.
3. Reject anything without clear docs/auth/security notes.
4. Install only in a sandbox profile/project.
5. Record:
   - install time
   - credentials required
   - tool names/descriptions
   - whether the agent selected the right tool
   - whether the output saved time

If the capability does not save time in one hour, remove it.

## Radar queue for issue #002

- MCP server security scoring
- Best sources for Agent Skill discovery
- x402 paid API examples
- Agent marketplace reality check
- Capability metadata schema proposal

## CTA

Suggest one capability to review next: MCP server, Agent Skill, registry, payment/access protocol, or marketplace.

Interim contact: `leodavinciad@users.noreply.github.com` with subject `Capability Radar suggestion`.
