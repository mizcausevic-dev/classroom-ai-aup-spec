# Classroom AI Acceptable Use Policy (AUP) v0.1 — Specification

**Status:** Draft
**Version:** 0.1.0
**Editor:** Miz Causevic
**License:** AGPL-3.0 (this document, schema, and examples). Implementations are unrestricted.

RFC 2119 keywords (**MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**) apply throughout.

---

## 1. Scope

This specification defines a JSON document format for **Classroom AI Acceptable Use Policies (AUPs)** — machine-readable district-, school-, course-, or assignment-level declarations of what AI use is permitted in an educational setting and what is required of vendors who serve that setting.

A Classroom AI AUP is consumed by:

- **Learning Management Systems** (Canvas, Schoology, Google Classroom, Moodle) reading the operative policy at submission time and surfacing "AI is allowed for this assignment in roles X, Y, Z; you must disclose"
- **AI tutor vendors** checking whether their tutor's posture (Tutor Card) matches the institution's requirements before signing a contract
- **District procurement teams** publishing one machine-readable policy a thousand vendors can read mechanically
- **Students** seeing the operative rules without reading a 20-page handbook
- **The Student AI Disclosure document itself**, which references its operative AUP via `aup_uri`

A Classroom AI AUP is **the district / school / teacher side** of the EdTech trio:

| Side | Spec | Declares |
|---|---|---|
| Vendor | [AI Tutor Card](https://github.com/mizcausevic-dev/ai-tutor-card-spec) | what the AI tutor does |
| **District / school / teacher** | **Classroom AI AUP** (this) | **what AI use is permitted** |
| Student | [Student AI Disclosure](https://github.com/mizcausevic-dev/student-ai-disclosure-spec) | what the student actually did |

A grader, an LMS, or an automated compliance checker can mechanically join all three and produce a single answer: "this student's use complies with the operative policy and the vendor's posture is acceptable to the institution."

## 2. Terminology

- **Policy** — a single Classroom AI AUP document.
- **Operative policy** — the AUP in force at the moment a student submits an artifact. Determined by walking from assignment → course → school → district scope until a policy is found.
- **Scope** — the population the policy applies to (district, school, course, or single assignment).
- **Parent policy** — a broader policy a narrower one refines. Course AUPs typically refine school AUPs; school AUPs refine district AUPs.
- **Permitted role** — one of the 13 values in the [Student AI Disclosure role taxonomy](https://github.com/mizcausevic-dev/student-ai-disclosure-spec/blob/main/SPEC.md#48-roles-conditional) the policy allows.
- **Disclosure requirement** — whether the student-side Student AI Disclosure is required: always, only when AI is used, or never.

## 3. Design philosophy

Three principles drive the design:

### 3.1 Machine-readable beats handbook

District AI policies today live in 20-page PDFs. The single most important property of this spec is that an LMS, a vendor's compliance system, or a grader's automated checker can read a single JSON document and resolve "what's allowed here" in O(1) time. The PDF still exists; the AUP is the operational alias.

### 3.2 Layered refinement

A course AUP refines a school AUP refines a district AUP. The spec encodes this via `scope.type` and `scope.parent_policy_uri`. A consumer walks up the hierarchy if needed. Each layer can only **tighten**, never **loosen** the parent — though this is a SHOULD, not a MUST, because real institutions sometimes carve narrow exceptions and need an audit trail more than they need an enforcement rail.

### 3.3 Procurement-grade vendor requirements

The `vendor_requirements` section is the procurement weapon. A district publishing its AUP can declare in JSON: "vendors MUST publish a Tutor Card; vendors MUST attest FERPA and COPPA; content filter strength MUST be 'strict'; retention MUST be ≤ 90 days." A vendor's Tutor Card can be mechanically checked against this in milliseconds. Procurement reviews shrink from weeks to minutes.

## 4. Document structure

### 4.1 `aup_version` (required)

A semver string. **MUST** be `"0.1"` for documents conforming to this draft.

### 4.2 `policy_id` (required)

A stable identifier, unique within the publisher. Kebab-case recommended. Example: `"lincoln-district-42-2026-ai-aup"`.

### 4.3 `policy_name` (required)

Human-readable title. Example: `"Lincoln High District 42 AI Acceptable Use Policy"`.

### 4.4 `version` (required)

Semver for the policy itself. Distinct from `aup_version` (the spec version).

### 4.5 `effective_at` (required)

ISO 8601 timestamp. The moment this policy takes effect. Consumers MAY treat a policy with `effective_at` in the future as not-yet-operative.

### 4.6 `expires_at` (optional)

ISO 8601 timestamp. The moment this policy is superseded. Consumers MUST NOT enforce an expired policy.

### 4.7 `replaces` (optional)

A pointer to the prior policy URI this one replaces. Used for audit trail.

### 4.8 `scope` (required)

The population this policy applies to.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | enum | yes | `district` / `school` / `course` / `assignment` |
| `institution_id` | string | yes | Stable identifier for the publishing institution. |
| `grade_bands` | array of enum | no | One or more of `K-5`, `6-8`, `9-12`, `undergrad`, `grad`, `adult-ed`. |
| `course_ids` | array of string | conditional | Required when `type` is `course`. |
| `assignment_ids` | array of string | conditional | Required when `type` is `assignment`. |
| `parent_policy_uri` | URI | no | Pointer to the broader policy this one refines. |

### 4.9 `permitted_use` (required)

What AI use is permitted.

| Field | Type | Required | Description |
|---|---|---|---|
| `permitted_roles` | array of enum | yes | Subset of the 13-value role taxonomy. Empty array = no AI use permitted. |
| `permitted_tool_categories` | array of enum | no | `text_chat` / `tutoring` / `code_assistant` / `image_generation` / `translation` / `research_synthesis` / `data_analysis` / `general`. |
| `permitted_tools` | array of object | no | Explicit allow-list. Each: `{ name, agent_card_uri?, tutor_card_uri?, notes? }`. |
| `assistance_extent_max` | enum | yes | Maximum extent allowed: `none` / `minor` / `substantial` / `primary_author`. |

The role taxonomy values are the same enum used by the [Student AI Disclosure spec, §4.8](https://github.com/mizcausevic-dev/student-ai-disclosure-spec/blob/main/SPEC.md#48-roles-conditional). The two specs MUST stay in sync; an AUP cannot permit a role the Student AI Disclosure cannot disclose.

### 4.10 `prohibited_use` (optional)

What AI use is explicitly prohibited. Useful for narrow carve-outs (e.g. "AI is permitted for editing but never for drafting in a creative-writing course").

| Field | Type | Required | Description |
|---|---|---|---|
| `prohibited_roles` | array of enum | no | Roles explicitly forbidden. |
| `prohibited_uses` | array of string | no | Free-text prohibitions (cheating on assessments, generating answers to test questions, surveillance, etc.). |

### 4.11 `disclosure_requirements` (required)

When and how the student-side Student AI Disclosure is required.

| Field | Type | Required | Description |
|---|---|---|---|
| `required_when` | enum | yes | `always` (every submission carries a disclosure) / `when_used` (only when AI was used) / `never`. |
| `required_prompt_evidence_mode` | enum | no | `full` / `hashed` / `omitted` / `any`. Default is `any`. |
| `signature_required` | boolean | yes | Whether `signed_by_student: true` is required. |
| `teacher_acknowledgment_required` | boolean | no | Whether grading workflow must capture a teacher acknowledgment. |
| `artifact_hash_required` | boolean | no | Whether the disclosure must bind to artifact bytes via `artifact_hash`. Default `true`. |

### 4.12 `supervision` (optional)

How AI use is supervised.

| Field | Type | Required | Description |
|---|---|---|---|
| `level` | enum | yes | `unsupervised` / `teacher_visible` / `teacher_approved` / `proctored`. |
| `human_in_loop_categories` | array of string | no | Topics that always escalate to a human counselor (mental_health, abuse, self_harm, etc.). |

### 4.13 `vendor_requirements` (optional but procurement-critical)

Properties a vendor's [Tutor Card](https://github.com/mizcausevic-dev/ai-tutor-card-spec) or [Agent Card](https://github.com/mizcausevic-dev/agent-cards-spec) MUST satisfy to be approved for use under this AUP.

| Field | Type | Required | Description |
|---|---|---|---|
| `requires_tutor_card` | boolean | no | Vendor MUST publish a Tutor Card. |
| `requires_agent_card` | boolean | no | Vendor MUST publish an Agent Card. |
| `required_compliance` | array of enum | no | `ferpa` / `coppa` / `gdpr` / `state-specific`. |
| `state_specific_laws` | array of string | no | E.g. `["NY-Ed-Law-2-D", "CA-SOPIPA"]`. |
| `required_content_filter_strength_min` | enum | no | `strict` / `moderate` / `light`. Matches Tutor Card. |
| `requires_mandated_reporter_protocol` | boolean | no | |
| `requires_human_in_loop_for` | array of string | no | E.g. `["mental_health_disclosure"]`. |
| `retention_days_max` | integer | no | Vendor data retention upper bound. |
| `prohibits_third_party_data_sharing` | boolean | no | |
| `prohibits_model_training_on_student_data` | boolean | no | |

### 4.14 `parent_notification` (optional but COPPA-relevant)

| Field | Type | Required | Description |
|---|---|---|---|
| `notification_level` | enum | yes | `none` / `at_enrollment` / `per_assignment` / `per_use`. |
| `parent_consent_required` | boolean | yes | Whether parent consent gates AI use. |
| `consent_age_threshold` | integer | no | Default 13 (COPPA). |

### 4.15 `enforcement` (optional)

| Field | Type | Required | Description |
|---|---|---|---|
| `violation_response` | array of string | no | Free-text consequences (academic-integrity referral, grade reduction, etc.). |
| `appeals_process_uri` | URI | no | Where students appeal. |

### 4.16 `published_by` (required)

The publishing institution / individual.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Display name. |
| `role` | string | no | E.g. `"district"`, `"school"`, `"course-instructor"`. |
| `contact_uri` | URI | no | mailto: or webpage. |

### 4.17 `published_at` (required)

ISO 8601 timestamp. When this version of the policy was published.

### 4.18 `audit_log_uri` (optional)

URI of the policy's change-log document.

## 5. Conditional rules

### 5.1 Scope completeness

If `scope.type` is `course`, then `scope.course_ids` **MUST** be present and non-empty.
If `scope.type` is `assignment`, then `scope.assignment_ids` **MUST** be present and non-empty.

### 5.2 Extent vs. roles consistency

If `permitted_use.assistance_extent_max` is `none`, then `permitted_use.permitted_roles` **MUST** be empty.
Conversely, if `permitted_use.permitted_roles` is non-empty, then `permitted_use.assistance_extent_max` **MUST NOT** be `none`.

### 5.3 Role disjointness

`permitted_use.permitted_roles` and `prohibited_use.prohibited_roles` (when both present) **MUST** be disjoint sets.

### 5.4 Refinement constraint (SHOULD)

A policy that declares a `scope.parent_policy_uri` **SHOULD NOT** permit roles, tools, or extents that the parent policy prohibits. Consumers MAY enforce this by walking the chain.

### 5.5 Effective window

`expires_at`, when present, **MUST** be strictly later than `effective_at`.

### 5.6 Consent threshold

If `parent_notification.parent_consent_required` is `true` and the policy applies to any student under `parent_notification.consent_age_threshold` (default 13), consumers **MUST** treat the AUP as gated on parent consent rather than enrollment alone.

## 6. Well-known location

A district- or school-level AUP **SHOULD** be served at:

```
https://<institution-domain>/.well-known/ai-aup.json
```

A course- or assignment-scoped AUP **MAY** be served at:

```
https://<institution-domain>/.well-known/aups/<scope-id>.json
```

The Student AI Disclosure's `aup_uri` field is the canonical pointer from a submission to the operative AUP.

## 7. Closing the EdTech trio

A consumer joining all three specs can mechanically answer one question: *"Is this submission allowed?"*

```
let aup       = fetch(disclosure.aup_uri);
let disclosed = disclosure.roles;
let permitted = aup.permitted_use.permitted_roles;
let extent_ok = (
  ordinal(disclosure.assistance_extent) <=
  ordinal(aup.permitted_use.assistance_extent_max)
);

let compliant = isSubset(disclosed, permitted) &&
                extent_ok &&
                disclosure.signed_by_student;
```

For tutoring tools, a separate join:

```
let tutor  = fetch(tutor_card_uri);
let needs  = aup.vendor_requirements;

let tutor_ok = (
  (!needs.requires_tutor_card || tutor_card_uri !== null) &&
  (!needs.required_compliance.includes("ferpa") || tutor.data_privacy.ferpa_compliant) &&
  (!needs.required_compliance.includes("coppa") || tutor.data_privacy.coppa_compliant) &&
  (!needs.required_content_filter_strength_min ||
    ordinal(tutor.safety.content_filter_strength) >=
    ordinal(needs.required_content_filter_strength_min))
);
```

Three JSON documents, two joins, one answer.

## 8. Security and policy considerations

- **Authoritative source.** AUPs **SHOULD** be served from the institution's primary domain. Consumers MAY treat AUPs served from third-party CDNs with suspicion unless the institution publishes a redirect.
- **Signing.** v0.1 does not require cryptographic signing. v0.2 may add an optional `signature` field similar to the AEO Protocol's audit block.
- **Versioning.** Bumping `version` and `effective_at` is the normative way to update a policy. Consumers MUST NOT silently apply unannounced changes; the change-log lives at `audit_log_uri`.
- **Refusal to comply.** Consumers (LMS, vendors, students) MAY refuse to operate under an AUP they consider unlawful or unethical. The spec is a disclosure mechanism, not an enforcement mandate.

## 9. Open questions for v0.2

- **Per-assessment-type granularity.** v0.1 has `scope.type=assignment` but doesn't distinguish formative vs. summative assessments. v0.2 may add `scope.assessment_type`.
- **Localization.** v0.2 may add a `translations[]` block so a multilingual district can publish a single AUP with parallel-language `policy_name` / `prohibited_uses[]` text.
- **Cryptographic signing.** Detached signatures over canonical JSON.
- **Programmatic exemptions.** A `temporary_exemption[]` block for individual learner-level carve-outs (IEP-driven accommodations, ELL learners with translation needs).

## 10. Versioning

The `aup_version` field identifies the spec revision. v0.1 is a draft; consumers **SHOULD** treat unknown future versions as an error rather than attempting forward-compatible parsing.
