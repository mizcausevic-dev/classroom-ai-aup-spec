# Classroom AI Acceptable Use Policy (AUP)

A draft specification for **Classroom AI AUPs** — machine-readable district-, school-, course-, or assignment-level declarations of what AI use is permitted in an educational setting.

Today, school AI policies live in 20-page PDFs. An AUP makes the operational subset of that policy *machine-readable*: an LMS reads one JSON document at submission time and resolves "is this allowed?" in O(1).

This spec closes the **EdTech trio**:

| Side | Spec | Declares |
|---|---|---|
| Vendor | [AI Tutor Card](https://github.com/mizcausevic-dev/ai-tutor-card-spec) | what the AI tutor does |
| **District / school / teacher** | **Classroom AI AUP** (this) | **what AI use is permitted** |
| Student | [Student AI Disclosure](https://github.com/mizcausevic-dev/student-ai-disclosure-spec) | what the student actually did |

A consumer joining all three can mechanically answer: *"Is this submission allowed?"*

## What's declared

| Section | What it does |
|---|---|
| **Scope** | district / school / course / assignment with optional `parent_policy_uri` for layered refinement |
| **Permitted use** | role allow-list (from the same 13-role taxonomy as Student AI Disclosure), tool categories, optional explicit tool allow-list, `assistance_extent_max` |
| **Prohibited use** | role deny-list and free-text prohibitions (e.g. "no AI on assessments") |
| **Disclosure requirements** | when Student AI Disclosure is required (`always` / `when_used` / `never`), required prompt-evidence mode, signature, teacher acknowledgment, artifact-hash binding |
| **Supervision** | unsupervised / teacher_visible / teacher_approved / proctored; human-in-loop escalation categories |
| **Vendor requirements** | the procurement weapon: requires Tutor Card / Agent Card, required compliance (FERPA / COPPA / GDPR / state laws), content-filter strength floor, retention day max, third-party-sharing and model-training prohibitions |
| **Parent notification** | notification level, consent required, age threshold (default 13 = COPPA) |
| **Enforcement** | violation responses, appeals URI |

## Conditional rules enforced by the schema

- `scope.type = course` requires non-empty `scope.course_ids`
- `scope.type = assignment` requires non-empty `scope.assignment_ids`
- `assistance_extent_max = none` requires `permitted_roles` to be empty
- `expires_at`, when present, must be strictly after `effective_at`

Beyond schema rules, the spec specifies a **refinement constraint** (`SHOULD`): a child policy MUST NOT permit what a parent prohibits. Walking the chain is the consumer's responsibility.

## Quickstart

1. Generate a stable `policy_id` (kebab-case).
2. Author an AUP conforming to [`aup.schema.json`](aup.schema.json). Start from one of the [examples](examples/).
3. Validate with any JSON Schema 2020-12 validator:
   ```bash
   npx -p ajv-cli -p ajv-formats ajv validate \
     -s aup.schema.json \
     -d examples/district-k12-strict.json \
     -c ajv-formats --spec=draft2020 --strict=false
   ```
4. Serve at `https://<your-institution>/.well-known/ai-aup.json` (district / school) or `/.well-known/aups/<scope-id>.json` (course / assignment).
5. Reference your AUP from each Student AI Disclosure submission via the `aup_uri` field.

## Closing the trio in code

```js
// 1. Fetch the three documents
const disclosure = await fetch(submissionUrl).then(r => r.json());
const aup        = await fetch(disclosure.aup_uri).then(r => r.json());
const tutorCard  = disclosure.tools_used[0]?.tutor_card_uri
  ? await fetch(disclosure.tools_used[0].tutor_card_uri).then(r => r.json())
  : null;

// 2. Check student-side compliance
const disclosed = new Set(disclosure.roles ?? []);
const permitted = new Set(aup.permitted_use.permitted_roles);
const extentOk  = rank(disclosure.assistance_extent) <= rank(aup.permitted_use.assistance_extent_max);
const compliant = [...disclosed].every(r => permitted.has(r)) && extentOk && disclosure.signed_by_student;

// 3. Check vendor-side acceptability (when a tutor is in play)
const needs = aup.vendor_requirements;
const tutorOk = !needs.requires_tutor_card
  ? true
  : tutorCard
    && (!needs.required_compliance?.includes("ferpa")  || tutorCard.data_privacy.ferpa_compliant)
    && (!needs.required_compliance?.includes("coppa")  || tutorCard.data_privacy.coppa_compliant)
    && (tutorCard.data_privacy.retention_days <= (needs.retention_days_max ?? Infinity));

const allowed = compliant && tutorOk;
```

Three JSON documents. Two joins. One answer.

## Files in this repo

- [`SPEC.md`](SPEC.md) — full v0.1 specification (10 sections)
- [`aup.schema.json`](aup.schema.json) — JSON Schema (draft 2020-12) with conditional rules
- [`examples/`](examples/) — reference AUPs:
  - [`district-k12-strict.json`](examples/district-k12-strict.json) — Lincoln District 42, K-12, strict filter, COPPA-gated, hashed-prompt evidence required
  - [`university-cs-permissive.json`](examples/university-cs-permissive.json) — State U CS180 final project, AI primary-author OK, full prompt evidence required
  - [`proctored-exam-no-ai.json`](examples/proctored-exam-no-ai.json) — single-assignment AUP for a proctored Bio 11 midterm: `permitted_roles: []`, `assistance_extent_max: "none"`

## Status

**v0.1 draft.** Issues and pull requests welcome.

## License

MIT-licensed. The specification text, JSON Schema, and example documents in this repository may be freely implemented, extended, redistributed, or incorporated into commercial or non-commercial products with attribution. Reference implementations of this spec (such as [mcp-kinetic-gain](https://github.com/mizcausevic-dev/mcp-kinetic-gain)) are licensed separately under AGPL-3.0.

## Kinetic Gain Protocol Suite

A family of open specifications for the answer-engine and agent era:

| Spec | What it does |
|---|---|
| [AEO Protocol](https://github.com/mizcausevic-dev/aeo-protocol-spec) | Entity declaration at `/.well-known/aeo.json` |
| [Prompt Provenance](https://github.com/mizcausevic-dev/prompt-provenance-spec) | Versioned, lineaged, reviewable LLM prompt records |
| [Agent Cards](https://github.com/mizcausevic-dev/agent-cards-spec) | Declarative agent capability + refusal disclosure |
| [AI Evidence Format](https://github.com/mizcausevic-dev/ai-evidence-format-spec) | Structured citations for LLM-generated claims |
| [MCP Tool Cards](https://github.com/mizcausevic-dev/mcp-tool-card-spec) | Per-tool disclosure for Model Context Protocol servers |
| [AI Tutor Cards](https://github.com/mizcausevic-dev/ai-tutor-card-spec) | EdTech-specialized agent disclosure (vendor-side) |
| [Student AI Disclosure](https://github.com/mizcausevic-dev/student-ai-disclosure-spec) | Student-side disclosure attached to submitted work |
| **Classroom AI AUP** (this) | District / school / course AI policy (the third leg of the EdTech trio) |
| [AI Incident Card](https://github.com/mizcausevic-dev/ai-incident-card-spec) | Post-incident disclosure ("CVE for AI agents") — cross-cuts the entire Suite |

---

**Connect:** [LinkedIn](https://www.linkedin.com/in/mirzacausevic/) · [Kinetic Gain](https://kineticgain.com) · [Medium](https://medium.com/@mizcausevic/) · [Skills](https://mizcausevic.com/skills/)
