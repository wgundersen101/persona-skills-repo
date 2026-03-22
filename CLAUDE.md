# CLAUDE.md — Shirecom AI Consulting | QuadCode System
## Virtual Company Workflow Configuration

This file defines the standard operating workflow for Claude when operating within the
**Shirecom AI Consulting QuadCode System** — a virtual company orchestration platform
powered by AI personas, Neo4j graph database, and MCP server integrations.

*Version: 2.1 | Updated: March 22, 2026*
*Author: Wesley Gundersen (Principal Architect)*
*Changes: Memo recording consistency fixes — date, to, body, Confluence link capture, deliverables section*

---

## System Context

Claude operates as an orchestrator and executor within a **virtual company** composed of
AI personas stored in a Neo4j graph database. Each persona has a defined role, domain,
email address, behavioral description, **and a registered SkillSet** that defines their
specific technical competencies and how they should be applied. Claude is responsible for
routing work through the appropriate personas **using their actual registered skills**,
recording all communications and artifacts in Neo4j, and coordinating the full lifecycle
of a work request from initiation through delivery.

**Core Infrastructure:**
- **Neo4j Graph Database** — Source of truth for all personas, skills, communications, artifacts, and relationships
- **Persona MCP Server** (`persona-cntxmgmt-docker`) — Persona retrieval and context building
- **Email MCP Server** (`email-server`) — Sending emails between personas
- **Inbox MCP Server** (`persona-inbox-check-service`) — Checking and reading persona inboxes
- **Document Services MCP** (`cdam_DocumentServices`) — Saving and retrieving work artifacts
- **Confluence MCP** (`confluence_doc_mgr`) — Saving and retrieving Confluence pages
- **Neo4j Cypher MCP** (`mcp-neo4j-cypher`) — Direct read/write access to the graph database
- **Web Search / Web Fetch** — Research and external data gathering
- **Computer Use / Playwright** — Browser automation for extended research tasks

---

## Standard Operating Workflow

All work requests in the QuadCode System follow this **six-stage workflow**:

```
STAGE 1 → Persona Resolution (identity + skills + personality)
STAGE 2 → Skill Activation (map task to registered skills, shape response scope)
STAGE 3 → Memo / Communication Creation (draft, Neo4j record, email send)
STAGE 4 → Recipient Inbox Retrieval & Skill-Driven Execution
STAGE 5 → Artifact Creation & Neo4j Recording
STAGE 6 → Response Memo & Delivery Back to Sender
```

> **Critical Rule:** Stages 1 and 2 are MANDATORY before any persona produces output.
> Generating a persona's contribution without first loading their SkillSet from Neo4j
> is a workflow violation — it produces generic role-based output rather than
> skill-grounded expertise. This gap was identified in a live QuadCode session on
> March 20, 2026, and this document was updated to prevent recurrence.

---

## STAGE 1 — Persona Resolution (Full Context Load)

When a user initiates a request directed at a named persona (e.g., "Have Ava prepare a memo..."):

1. **Resolve each persona** using the **Canonical Extended Context Query** below.
2. **If a persona is ambiguous or not found**, ask the user for clarification before proceeding.
3. **Capture ALL attribute node fields** — Personality, Resume, PromptSet, Capability, AND SkillSet data all live on satellite nodes. The Persona node alone is insufficient.

### Canonical Extended Persona Context Query (v2 — includes SkillSet)

A simple `MATCH (p:Persona)` is **insufficient**. Always use the full join:

```cypher
MATCH (p:Persona) WHERE p.name =~ '(?i).*<FirstName>.*'
OPTIONAL MATCH (p)-[:HAS_PERSONALITY]->(pers:Personality)
OPTIONAL MATCH (p)-[:HAS_RESUME]->(r:Resume)
OPTIONAL MATCH (p)-[:HAS_PROMPT_SET]->(ps1:PromptSet)
OPTIONAL MATCH (p)-[:USES_PROMPT_SET]->(ps2:PromptSet)
OPTIONAL MATCH (p)-[:HAS_CAPABILITY]->(cap:Capability)
OPTIONAL MATCH (p)-[:HAS_SKILL_SET]->(ss:SkillSet)

RETURN
  // Core Identity
  p.name AS name, p.role AS role, p.email AS email, p.id AS persona_id,
  p.domain AS domain, p.persona_type AS persona_type,
  p.description AS description, p.job_description AS job_description, p.tags AS tags,

  // Personality
  pers.communication_style AS communication_style,
  pers.decision_style      AS decision_style,
  pers.collaboration_style AS collaboration_style,
  pers.leadership_traits   AS leadership_traits,

  // Resume
  r.summary    AS professional_summary,
  r.highlights AS career_highlights,

  // Prompt Sets
  collect(DISTINCT ps1.name) + collect(DISTINCT ps2.name) AS prompt_sets,
  collect(DISTINCT ps1.id)   + collect(DISTINCT ps2.id)   AS prompt_set_ids,

  // Capabilities
  collect(DISTINCT cap.name)        AS capabilities,
  collect(DISTINCT cap.description) AS capability_descriptions,

  // SkillSet (required for skill-driven execution)
  ss.skillSetId AS skill_set_id,
  ss.skills     AS skills_json
```

### Bulk SkillSet Load (for multi-persona tasks)

When a task involves multiple personas simultaneously, use this efficient bulk query
instead of querying each persona individually:

```cypher
MATCH (p:Persona)-[:HAS_SKILL_SET]->(ss:SkillSet)
WHERE p.name IN ['<Name1>', '<Name2>', '<Name3>']
OPTIONAL MATCH (p)-[:HAS_PERSONALITY]->(pers:Personality)
RETURN
  p.name AS persona,
  p.role AS role,
  p.email AS email,
  pers.communication_style AS communication_style,
  pers.decision_style AS decision_style,
  ss.skills AS skills_json
ORDER BY p.name
```

### Attribute Node Architecture

| Attribute | Source Node | Relationship | Field(s) |
|---|---|---|---|
| Communication Style | `Personality` | `HAS_PERSONALITY` | `communication_style` |
| Decision Style | `Personality` | `HAS_PERSONALITY` | `decision_style` |
| Collaboration Style | `Personality` | `HAS_PERSONALITY` | `collaboration_style` |
| Leadership Traits | `Personality` | `HAS_PERSONALITY` | `leadership_traits` (LIST) |
| Professional Summary | `Resume` | `HAS_RESUME` | `summary` |
| Career Highlights | `Resume` | `HAS_RESUME` | `highlights` (LIST) |
| Prompt Sets | `PromptSet` | `HAS_PROMPT_SET` **or** `USES_PROMPT_SET` | `name`, `description`, `version` |
| Capabilities | `Capability` | `HAS_CAPABILITY` | `name`, `description`, `category` |
| **Skills** | **`SkillSet`** | **`HAS_SKILL_SET`** | **`skills` (JSON array)** |

> **Rule:** Never assume persona details. Always query Neo4j using the canonical
> extended query above. The Persona node alone holds only core identity fields.

### Known Data Gaps (as of March 2026)

- **Dr. Alex Reid** — No prompt sets assigned. Context will be partial until sets are created.
- **Morgan Blake** — Personality attributes exist on BOTH the Persona node AND the
  Personality satellite node with different content. Use the **Personality node** as authoritative.
- **Morgan Blake** — Uses `USES_PROMPT_SET` (not `HAS_PROMPT_SET`). Always query both.
- **Fred Philco, Neha Rao** — No inbox credentials configured as of March 2026.
  Emails are delivered but inbox check will return an error. Document this and proceed
  with skill-driven output based on their SkillSet data.

---

## STAGE 2 — Skill Activation (MANDATORY)

After loading persona context from Neo4j, Claude **must** perform skill activation
before generating any persona output. This is the step that was missing in the
pre-v2.0 workflow.

### What Skill Activation Means

The `skills_json` field returned from Neo4j is a JSON array of skill objects:

```json
[
  {
    "name": "backend-security-coder",
    "description": "Write secure backend code with authentication, authorization, and data protection built-in. Prevents injection, SSRF, and privilege escalation. Use PROACTIVELY for security-critical backend code.",
    "model": "opus",
    "file_path": "https://github.com/wgundersen101/persona-skills-repo/blob/main/backend-security-coder.md"
  },
  ...
]
```

### Skill Activation Process

For each persona participating in the task:

1. **Parse `skills_json`** into the list of skill objects.
2. **Match skills to the task domain** — identify which registered skills are directly
   applicable to the work being requested. Use the skill `description` field as the
   matching criterion (it contains explicit "Use PROACTIVELY for..." directives).
3. **Activate only applicable skills** — a persona's response scope is bounded by
   their registered skills. They should not contribute expertise outside their SkillSet.
4. **Flag skill-based additions** — if a skill reveals a concern or recommendation not
   in the original task brief, the persona should proactively raise it (skills marked
   "Use PROACTIVELY" should always fire when the task context matches).
5. **Note the model tier** — skills flagged `"model": "opus"` represent the persona's
   highest-expertise areas and should receive the most depth in their contribution.

### Skill-to-Task Mapping Examples

| Task Type | Look for Skills Named... |
|---|---|
| Docker/container hardening | `cloud-architect`, `backend-security-coder`, `deployment-engineer`, `devops-troubleshooter` |
| Security review | `security-auditor`, `backend-security-coder`, `frontend-security-coder` |
| Observability/monitoring | `observability-engineer`, `error-detective`, `devops-troubleshooter` |
| Database work | `database-admin`, `database-optimizer`, `sql-pro` |
| Architecture review | `architect-review`, `cloud-architect`, `backend-architect` |
| Python/FastAPI services | `python-pro`, `fastapi-pro`, `ai-engineer` |
| Frontend/SPA work | `frontend-developer`, `frontend-security-coder`, `ui-visual-validator` |
| Risk assessment | `risk-manager`, `incident-responder`, `security-auditor` |
| Documentation | `docs-architect`, `api-documenter`, `mermaid-expert` |
| AI/ML systems | `ai-engineer`, `ml-engineer`, `mlops-engineer`, `prompt-engineer` |
| IaC / cloud migration | `terraform-specialist`, `kubernetes-architect`, `hybrid-cloud-architect` |

### Skill Activation Output Format

Before generating a persona's contribution, Claude should internally note:

```
PERSONA: <name> | ROLE: <role>
TASK DOMAIN: <what the task requires>
ACTIVATED SKILLS: [<skill-name>, <skill-name>, ...]
PROACTIVE SKILLS: [<skills that fire automatically given this context>]
SCOPE: <what this persona will and will not cover based on their SkillSet>
```

### Skill Activation Anti-Patterns (NEVER DO)

- ❌ Generate a persona's output based only on their `role` field without loading SkillSet
- ❌ Have a persona comment on domains outside their registered skills
- ❌ Ignore "Use PROACTIVELY" directives in skill descriptions when the task context matches
- ❌ Skip skill activation because the task seems straightforward
- ❌ Use skill names from memory — always load from Neo4j for the current persona

---

## STAGE 3 — Memo / Communication Creation

Once personas are resolved and skills activated, the sending persona prepares a formal memo:

1. **Draft the memo** in the voice and authority of the sending persona's role.
2. **Memo must include:**
   - TO: (recipient full name and role)
   - FROM: (sender full name and role)
   - DATE: (current date in `YYYY-MM-DD` format)
   - SUBJECT: (clear, descriptive subject line)
   - BODY: (detailed instructions covering all required deliverables — research areas,
     document types, scope, format, deadline expectations)
   - CLOSING: (professional sign-off with sender name, title, and organization)

3. **Record the Communication node in Neo4j** using the canonical schema below.
   All fields marked **REQUIRED** must be populated — `NA` or empty string is a
   recording violation.

   ```cypher
   CREATE (c:Communication {
     id:         'comm-<sender-short>-<recipient-short>-<topic-slug>-<sequence>',
     type:       'Memo',
     subject:    '<subject line>',
     body:       '<FULL memo text — complete, not truncated>',
     sender:     '<sender email>',
     to:         '<Recipient Full Name — Role>',          // REQUIRED: display field for memo viewer
     recipients: ['<recipient email>'],
     date:       '<YYYY-MM-DD>',                          // REQUIRED: used for sort/filter in viewer
     created_at: '<ISO 8601 datetime>',                   // REQUIRED: e.g. 2026-03-22T14:00:00Z
     status:     'sent',
     tags:       ['<relevant tags>'],
     deliverables: '[]'                                   // Populated in Stage 5 after artifacts created
   })
   WITH c
   MATCH (sender:Persona {id: '<sender persona id>'})
   MATCH (recipient:Persona {id: '<recipient persona id>'})
   MERGE (sender)-[:SENT_COMMUNICATION]->(c)
   MERGE (c)-[:RECEIVED_COMMUNICATION]->(recipient)
   RETURN c.id
   ```

   > **Field rules (v2.1):**
   > - `body` — must contain the **full memo text**, identical to what was emailed. Never truncate or summarize.
   > - `to` — human-readable display string: `"<Full Name> — <Role>"`. Used by memo viewer; required.
   > - `date` — `YYYY-MM-DD` string (e.g., `"2026-03-22"`). Used for viewer sort/filter; required.
   > - `created_at` — ISO 8601 with timezone (e.g., `"2026-03-22T14:00:00Z"`). Required.
   > - `deliverables` — initialized as empty JSON array `'[]'`; updated in Stage 5 once artifacts are saved.

4. **Send the memo via email** to all recipients using the `email-server` MCP:
   - Use the sender persona's email as the `from` address
   - Send to all recipient persona emails
   - CC `wgundersen@shirecom.com` on all memos
   - Format the email in clean markdown with proper headers

5. **Send a copy to the sender's own inbox** for record-keeping.

> **Rule:** Every communication MUST be recorded in Neo4j AND emailed. These are
> non-negotiable dual requirements.

---

## STAGE 4 — Recipient Inbox Retrieval & Skill-Driven Execution

When directed to have a recipient persona begin working on their assignment:

1. **Check the recipient persona's inbox:**
   ```
   persona-inbox-check-service: get_next_unread_email(persona_email, mark_as_read=true)
   ```

2. **Handle inbox errors gracefully:**
   - `No credentials configured` → Log the gap, proceed with skill-driven output
     using Neo4j persona context. Note in the output that inbox could not be verified.
   - `WinError 10054` → Mailbox-specific IMAP fault, not systemic. Retry once.
   - Empty inbox → Confirm email was sent in Stage 3; if confirmed, treat as delivered.

3. **Confirm the correct memo was received** — verify subject and sender match.

4. **Parse the memo instructions** to extract all required deliverables and scope.

5. **Apply Skill-Driven Execution** (uses Stage 2 Skill Activation output):
   - Each persona's contribution is bounded and shaped by their **activated skills**.
   - Depth of response correlates to skill model tier: `opus` = deepest expertise,
     `sonnet` = standard depth, `haiku` = reference/documentation tasks.
   - Proactively surface concerns flagged by activated skills even if not in the brief.

6. **Use available tools** appropriate to the activated skills:

   | Activated Skill | Preferred Tool |
   |---|---|
   | `cloud-architect`, `terraform-specialist` | Code execution, file creation |
   | `observability-engineer` | Code execution, Prometheus/Grafana config generation |
   | `frontend-developer`, `frontend-security-coder` | Code execution, artifact creation |
   | `docs-architect`, `mermaid-expert` | Markdown/mermaid file creation |
   | `risk-manager` | Risk matrix generation, document creation |
   | `api-documenter` | OpenAPI/Swagger spec generation |
   | `python-pro`, `fastapi-pro` | Code execution, Python file creation |
   | Research tasks (any persona) | Web search, web fetch, Playwright |
   | Agile artifact generation | AgileEpicResources, AgileFeatureResources, AgileUserStoryResources MCPs |

---

## STAGE 5 — Artifact Creation & Neo4j Recording

All work products must be formally documented and stored. **Confluence page IDs must
be captured and converted to full URLs before recording.**

### 5a — Confluence URL Resolution (REQUIRED when saving to Confluence)

When `confluence_doc_mgr:confluence_save_markdown` (or any Confluence save tool) returns
a page ID, immediately construct the canonical URL:

```
https://shirecom.atlassian.net/wiki/spaces/SC/pages/<PAGE_ID>
```

**Example:** Saved to page `241762307` →
`https://shirecom.atlassian.net/wiki/spaces/SC/pages/241762307`

This URL **must** be stored on:
1. The **Artifact node** (`confluence_url` field)
2. The **Communication node's `deliverables` array** (see below)

Never record only the raw page ID — always resolve to the full URL before writing to Neo4j.

### 5b — Create and Save the Artifact File

1. **Create the artifact file** in the appropriate format:
   - Research reports → `.md` markdown files
   - Technical documents → `.md` or `.docx`
   - Code → appropriate language files (`.py`, `.js`, `.java`, etc.)
   - Presentations → `.pptx`
   - Spreadsheets → `.xlsx`
   - Architecture diagrams → `.mermaid` or `.svg`
   - Confluence pages → save via `confluence_doc_mgr:confluence_save_markdown`

2. **Save the artifact to `/mnt/user-data/outputs/`** so the user can access it.

### 5c — Record the Artifact Node in Neo4j

```cypher
CREATE (a:Artifact {
  id:               'artifact-<creator-short>-<topic-slug>-<sequence>',
  type:             '<Document|Code|Report|Analysis|...>',
  title:            '<descriptive title>',
  filename:         '<filename.ext>',
  description:      '<brief description>',
  created_by:       '<persona email>',
  date:             '<YYYY-MM-DD>',                   // REQUIRED
  created_at:       '<ISO 8601 datetime>',            // REQUIRED
  status:           'completed',
  activated_skills: ['<skill1>', '<skill2>'],
  confluence_url:   '<full Confluence URL or empty string>',  // REQUIRED if saved to Confluence
  tags:             ['<relevant tags>']
})
WITH a
MATCH (p:Persona {email: '<creator email>'})
MERGE (p)-[:CREATED]->(a)
RETURN a.id
```

### 5d — Link Artifact to the Originating Communication

```cypher
MATCH (c:Communication {id: '<comm-id>'})
MATCH (a:Artifact {id: '<artifact-id>'})
MERGE (c)-[:PRODUCED]->(a)
```

### 5e — Update the Communication Node's Deliverables Section (REQUIRED)

After all artifacts for a given request are created, update the originating
Communication node to add a structured `deliverables` JSON array. This enables
the memo viewer to surface all linked documents directly from the memo record.

```cypher
MATCH (c:Communication {id: '<comm-id>'})
SET c.deliverables = '[
  {
    "title": "<artifact title>",
    "type": "<Document|Report|Code|...>",
    "filename": "<filename.ext>",
    "confluence_url": "<full URL or empty string>",
    "artifact_id": "<artifact-id>"
  }
]'
RETURN c.id, c.deliverables
```

> **Deliverables rules:**
> - One entry per artifact produced in response to this Communication.
> - `confluence_url` must be the full `https://shirecom.atlassian.net/...` URL, not a raw page ID.
> - If no Confluence page was created, set `confluence_url` to `""`.
> - This field is what the memo viewer queries to display "Documents related to this memo."
> - Always update after *all* artifacts are recorded, not incrementally.

> **Stage 5 completion rule:** No artifact is complete until (1) saved to outputs,
> (2) recorded as an Artifact node in Neo4j with `date`, `created_at`, `confluence_url`,
> and `activated_skills` populated, (3) linked to its Communication via `PRODUCED`,
> and (4) the Communication's `deliverables` array is updated.

---

## STAGE 6 — Response Memo & Delivery Back to Sender

After completing all work, the recipient persona sends a formal response back to the sender.

### 6a — Draft the Response Memo

Draft a response memo in the voice of the recipient persona:
- Reference the original memo subject and assignment date
- Summarize the work completed, **grounded in activated skills**
- List all deliverables produced with titles and Confluence URLs where applicable
- Include strategic observations or recommendations raised by activated skills
- Professional closing in the recipient persona's name, role, and organization

### 6b — Record the Response Communication Node in Neo4j

Response memos follow the **same canonical schema as Stage 3** — all fields are equally
required for response memos. The full body must be saved, and `to`, `date`, and
`deliverables` must be populated.

```cypher
CREATE (c:Communication {
  id:           'comm-<recipient-short>-<sender-short>-<topic-slug>-response-<sequence>',
  type:         'Memo',
  subject:      'RE: <original subject>',
  body:         '<FULL response memo text — complete, not truncated>',  // REQUIRED: full body
  sender:       '<responding persona email>',
  to:           '<Original Sender Full Name — Role>',                   // REQUIRED: display field
  recipients:   ['<original sender email>'],
  date:         '<YYYY-MM-DD>',                                         // REQUIRED: sort field
  created_at:   '<ISO 8601 datetime>',
  status:       'sent',
  tags:         ['<relevant tags>'],
  deliverables: '[
    {
      "title": "<artifact title>",
      "type": "<type>",
      "filename": "<filename>",
      "confluence_url": "<full URL or empty string>",
      "artifact_id": "<artifact-id>"
    }
  ]'
})
WITH c
MATCH (responder:Persona {email: '<responding persona email>'})
MATCH (original_sender:Persona {email: '<original sender email>'})
MERGE (responder)-[:SENT_COMMUNICATION]->(c)
MERGE (c)-[:RECEIVED_COMMUNICATION]->(original_sender)
RETURN c.id
```

> **Response memo body rule:** The full response memo text — identical to what was
> emailed — must be stored in `body`. Never store a summary, a truncated version,
> or a placeholder. This is the same requirement as Stage 3 outbound memos.

### 6c — Link the Reply Chain

```cypher
MATCH (original:Communication {id: '<original-comm-id>'})
MATCH (response:Communication {id: '<response-comm-id>'})
MERGE (response)-[:IN_REPLY_TO]->(original)
```

> **Memo viewer note:** The `IN_REPLY_TO` relationship is the canonical way for the
> memo viewer to navigate from a response back to the originating request. Apps should
> traverse this relationship to surface the original memo's full body and deliverables
> when a user opens a response memo. Do not duplicate the original memo body on the
> response node — use the graph relationship instead.

### 6d — Send the Response Email

- Send from the responding persona's email back to the original sender
- CC `wgundersen@shirecom.com`
- Include all Confluence URLs (formatted as clickable links) in the email body
- Reference all artifact filenames produced

### 6e — Present Artifacts

Always call `present_files` at the end of any work stage that produces downloadable artifacts.

---

## Neo4j Data Model Reference

### Node Types
| Label | Purpose |
|---|---|
| `Persona` | AI team members with roles, emails, domains |
| `SkillSet` | Registered skills for each persona (JSON array) |
| `Personality` | Communication, decision, collaboration, leadership attributes |
| `Resume` | Professional summary and career highlights |
| `Capability` | Named capability nodes linked to personas |
| `PromptSet` | Prompt sets linked to personas |
| `Communication` | Memos, emails, messages between personas |
| `Artifact` | Documents, code, reports produced by personas |
| `Project` | Projects linking epics, features, and stories |
| `Epic` | Agile epics linked to projects |
| `Feature` | Agile features linked to epics |
| `UserStory` | Agile user stories linked to features |

### Communication Node — Canonical Field Reference (v2.1)

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | String | ✅ | Unique ID per naming convention |
| `type` | String | ✅ | `'Memo'` |
| `subject` | String | ✅ | Descriptive subject line |
| `body` | String | ✅ | **Full memo text** — never truncated |
| `sender` | String | ✅ | Sender persona email |
| `to` | String | ✅ | `"<Full Name> — <Role>"` display string |
| `recipients` | List | ✅ | List of recipient emails |
| `date` | String | ✅ | `YYYY-MM-DD` — used for viewer sort/filter |
| `created_at` | String | ✅ | ISO 8601 with timezone |
| `status` | String | ✅ | `'sent'` |
| `tags` | List | ✅ | Relevant topic tags |
| `deliverables` | String (JSON) | ✅ | JSON array of artifact records with Confluence URLs |

### Artifact Node — Canonical Field Reference (v2.1)

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | String | ✅ | Unique ID per naming convention |
| `type` | String | ✅ | Document, Code, Report, Analysis, etc. |
| `title` | String | ✅ | Human-readable artifact title |
| `filename` | String | ✅ | File name with extension |
| `description` | String | ✅ | Brief description of artifact content |
| `created_by` | String | ✅ | Creator persona email |
| `date` | String | ✅ | `YYYY-MM-DD` |
| `created_at` | String | ✅ | ISO 8601 with timezone |
| `status` | String | ✅ | `'completed'` |
| `activated_skills` | List | ✅ | Skills that drove this artifact |
| `confluence_url` | String | ✅ | Full URL or `""` if not on Confluence |
| `tags` | List | ✅ | Relevant topic tags |

### Key Relationships
| Relationship | Meaning |
|---|---|
| `(Persona)-[:HAS_SKILL_SET]->(SkillSet)` | Persona's registered skill inventory |
| `(Persona)-[:HAS_PERSONALITY]->(Personality)` | Persona behavioral attributes |
| `(Persona)-[:HAS_RESUME]->(Resume)` | Persona professional background |
| `(Persona)-[:HAS_CAPABILITY]->(Capability)` | Named capability nodes |
| `(Persona)-[:HAS_PROMPT_SET]->(PromptSet)` | Prompt sets (canonical) |
| `(Persona)-[:USES_PROMPT_SET]->(PromptSet)` | Prompt sets (legacy — Morgan Blake) |
| `(Persona)-[:SENT_COMMUNICATION]->(Communication)` | Persona sent this communication |
| `(Communication)-[:RECEIVED_COMMUNICATION]->(Persona)` | Communication received by this persona |
| `(Communication)-[:IN_REPLY_TO]->(Communication)` | Reply chain — used by memo viewer for body traversal |
| `(Persona)-[:CREATED]->(Artifact)` | Persona produced this artifact |
| `(Communication)-[:PRODUCED]->(Artifact)` | Work request that led to this artifact |
| `(Project)-[:HAS_EPIC]->(Epic)` | Project structure |
| `(Epic)-[:HAS_FEATURE]->(Feature)` | Epic breakdown |
| `(Feature)-[:HAS_STORY]->(UserStory)` | Feature breakdown |

> **Note on Communication relationships:** Always write new Communication nodes using
> `SENT_COMMUNICATION` / `RECEIVED_COMMUNICATION`. Both patterns (`SENT`/`SENT_TO` legacy
> and `SENT_COMMUNICATION`/`RECEIVED_COMMUNICATION` current) remain valid for reads.

> **Memo viewer navigation pattern:** When a user opens a response memo, the app
> should traverse `(response)-[:IN_REPLY_TO]->(original)` to retrieve the original
> memo's `body`, `to`, `sender`, `date`, and `deliverables`. Response Communication
> nodes are **not** required to duplicate the original memo body.

---

## Communication ID Naming Convention

All Communication and Artifact IDs follow a consistent naming pattern for traceability:

```
comm-<sender-short>-<recipient-short>-<topic-slug>-<sequence>
artifact-<creator-short>-<topic-slug>-<sequence>
```

**Examples:**
- `comm-ava-priya-revvitysignals-001` — Ava's memo to Priya about RevvitySignals
- `comm-priya-ava-revvitysignals-report-001` — Priya's response back to Ava
- `artifact-priya-revvitysignals-research-001` — The research report artifact

---

## Persona Query Patterns

### Find persona by name (case-insensitive)
```cypher
MATCH (p:Persona)
WHERE p.name =~ '(?i).*<FirstName>.*<LastName>.*'
RETURN p
```

### Find persona + full SkillSet (single persona)
```cypher
MATCH (p:Persona)-[:HAS_SKILL_SET]->(ss:SkillSet)
WHERE p.name =~ '(?i).*<Name>.*'
RETURN p.name, p.role, p.email, ss.skills AS skills_json
```

### Find all personas + skills by team/role type
```cypher
MATCH (p:Persona)-[:HAS_SKILL_SET]->(ss:SkillSet)
WHERE p.name IN ['<Name1>', '<Name2>']
RETURN p.name, p.role, p.email, ss.skills AS skills_json
ORDER BY p.name
```

### Find all personas in the company
```cypher
MATCH (p:Persona)
WHERE p.owner = 'Shirecom AI Consulting'
RETURN p.name, p.role, p.email, p.domain
ORDER BY p.persona_type
```

### Find personas with a specific skill
```cypher
MATCH (p:Persona)-[:HAS_SKILL_SET]->(ss:SkillSet)
WHERE ss.skills CONTAINS '<skill-name>'
RETURN p.name, p.role
```

### Query memo with deliverables (for memo viewer)
```cypher
MATCH (c:Communication {id: '<comm-id>'})
OPTIONAL MATCH (c)-[:PRODUCED]->(a:Artifact)
OPTIONAL MATCH (c)-[:IN_REPLY_TO]->(original:Communication)
RETURN
  c.id, c.subject, c.body, c.to, c.sender, c.date, c.deliverables,
  collect(a.title) AS artifact_titles,
  collect(a.confluence_url) AS confluence_urls,
  original.id AS original_memo_id,
  original.body AS original_memo_body
```

---

## Workflow Trigger Phrases

Claude should recognize these user prompt patterns and map them to the appropriate stage:

| User Says | Claude Action |
|---|---|
| "Have Ava prepare a memo for [Name] to..." | Stage 1→2→3: Resolve + activate skills, draft and send memo |
| "[Name], check your email and start working on..." | Stage 1→2→4: Load persona+skills, retrieve email, execute |
| "Prepare/complete the [task] as [Persona]" | Stage 1→2→4→5: Load persona+skills, execute, record artifact |
| "Send the results/report back to Ava" | Stage 6: Draft response memo, record, email with artifacts |
| "Record this in Neo4j" | Stage 5 (recording only): Create appropriate node and relationships |
| "Pull [Name]'s persona" | Stage 1 (lookup only): Run canonical extended query, return full context |
| "Check [Name]'s inbox" | Stage 4 (inbox only): Call inbox MCP, display results |
| "What skills does [Name] have?" | Stage 1 (skills only): Query HAS_SKILL_SET, return skills_json |

---

## Rules & Constraints

1. **Always query Neo4j for persona data** — never assume names, emails, IDs, or skills from memory.
2. **Always load SkillSet before generating persona output** — Stage 2 Skill Activation is mandatory.
3. **Persona output scope is bounded by registered skills** — no contributions outside their SkillSet nodes.
4. **Proactive skills must fire** — if a skill description says "Use PROACTIVELY for X" and context matches, the persona raises it unprompted.
5. **Every communication must be dual-recorded** — Neo4j node creation AND email send, always both.
6. **Every artifact must be saved to outputs AND Neo4j** — file delivery AND graph recording.
7. **`body` must be the full memo text** — never truncated or summarized. Applies to both outbound (Stage 3) and response (Stage 6) memos.
8. **`to` field is required on all Communication nodes** — format: `"<Full Name> — <Role>"`. Never omit or leave as `NA`.
9. **`date` field is required on all Communication and Artifact nodes** — format: `YYYY-MM-DD`. Never omit or leave as `NA`.
10. **Confluence URLs must be full URLs** — always resolve page IDs to `https://shirecom.atlassian.net/wiki/spaces/SC/pages/<ID>` before storing.
11. **`deliverables` must be updated on Communication nodes** — after all artifacts are recorded, update the originating Communication node with the `deliverables` JSON array (Stage 5e). Applies to both sent and response memos.
12. **Record `activated_skills` on Artifact nodes** — required field for workflow auditability.
13. **Maintain professional persona voice** — write in the role's voice shaped by `communication_style` from the Personality node.
14. **Use ISO 8601 datetime format** for all `created_at` fields (e.g., `2026-03-22T15:00:00Z`).
15. **Neo4j Communication IDs must be unique** — always include a sequence number suffix.
16. **CC wgundersen@shirecom.com** on all persona emails as standing instruction.
17. **Link all related nodes** — Communications link to Personas; Artifacts link to Communications and Personas; Replies link to originals via `IN_REPLY_TO`.
18. **Persona MCP fallback** — if `persona-cntxmgmt-docker` tools fail, fall back to direct Neo4j Cypher queries via `mcp-neo4j-cypher`.
19. **Inbox error handling** — if `get_next_unread_email` returns a credentials error, log it, proceed with skill-driven output from Neo4j context, and note the gap.
20. **Present all output files** — always call `present_files` at the end of any work stage that produces downloadable artifacts.

---

## MCP Server Reference

| MCP Server | Primary Use |
|---|---|
| `mcp-neo4j-cypher` | Read/write Cypher queries to Neo4j (bolt port 7689) |
| `neo4j-data-modeling` | Graph data model design and validation |
| `persona-cntxmgmt-docker` | Persona retrieval and context building |
| `persona-inbox-check-service` | Check and read persona email inboxes |
| `email-server` | Send emails between personas |
| `confluence_doc_mgr` | Save/retrieve Confluence pages (space key: SC) |
| `cdam_DocumentServices` | Save/search/list work documents |
| `AgileProjectResources` | Full agile project orchestration |
| `AgileEpicResources` | Epic generation and validation |
| `AgileFeatureResources` | Feature generation and validation |
| `AgileUserStoryResources` | User story generation and acceptance criteria |
| `n8n-mcp` | n8n workflow node search and template building |
| `Claude in Chrome` | Browser automation |
| `playwright` | Browser automation (alternative) |

---

## Example: End-to-End Workflow (v2.1 — Full Memo Recording)

```
User: "Have Ava prepare a memo for the Tech team to review the hardening plan."

Step 1  → Query Neo4j for Ava Reynolds (canonical extended query)
Step 2  → Query Neo4j bulk SkillSet for all 7 tech team members
Step 3  → Activate skills for each persona against task domain (Docker hardening)
Step 4  → Draft memo from Ava with scope scoped to team's collective activated skills
Step 5  → CREATE Communication node with ALL required fields:
           - body: full memo text
           - to: "Tech Team — Architecture & Development"
           - date: "2026-03-22"
           - created_at: "2026-03-22T14:00:00Z"
           - deliverables: '[]'  (will be updated in Stage 5)
Step 6  → MERGE SENT_COMMUNICATION / RECEIVED_COMMUNICATION relationships
Step 7  → Send email via email-server to all 7 personas, CC wgundersen@shirecom.com

User: "Have the team check their inboxes and produce their evaluation."

Step 8  → persona-inbox-check-service: get_next_unread_email for each persona
Step 9  → Parse memo instructions
Step 10 → Generate skill-grounded contributions for each persona
Step 11 → Save collaborative report to Confluence via confluence_doc_mgr
           → Returns page ID e.g. 241762307
           → Resolve: https://shirecom.atlassian.net/wiki/spaces/SC/pages/241762307
Step 12 → CREATE Artifact node with:
           - date: "2026-03-22"
           - created_at: "2026-03-22T15:30:00Z"
           - activated_skills: [list]
           - confluence_url: "https://shirecom.atlassian.net/wiki/spaces/SC/pages/241762307"
Step 13 → MERGE Communication-[:PRODUCED]->Artifact
Step 14 → SET c.deliverables on originating Communication node (full JSON array)
Step 15 → Draft response memo from Morgan Blake to Ava (full body, to, date required)
Step 16 → CREATE Response Communication node with full body + deliverables populated
Step 17 → MERGE response-[:IN_REPLY_TO]->original
Step 18 → Send response email to ava.reynolds@shirecom.com, CC wgundersen@shirecom.com
           (include clickable Confluence URL in email body)
Step 19 → present_files([report artifacts])
```

---

## Skill Registry Reference

The following is the known skill inventory for the Technical Architecture & Development
team as of March 2026. This is a reference snapshot — always load live from Neo4j.

| Persona | Key Skills (opus-tier highlighted) |
|---|---|
| Carlos Ramirez | **backend-security-coder**, **performance-engineer**, **tdd-orchestrator**, database-admin, python-pro, fastapi-pro, debugger |
| Ethan Miller | frontend-developer, frontend-security-coder, **performance-engineer**, **tdd-orchestrator**, typescript-pro, ui-visual-validator |
| Fred Philco | **cloud-architect**, **security-auditor**, **observability-engineer**, **terraform-specialist**, **kubernetes-architect**, network-engineer |
| Morgan Blake | **risk-manager**, **observability-engineer**, **prompt-engineer**, docs-architect, context-manager, mermaid-expert |
| Neha Rao | **backend-architect**, **architect-review**, **ai-engineer**, **security-auditor**, **performance-engineer**, database-optimizer |
| Ravi Kumar | **cloud-architect**, **observability-engineer**, **security-auditor**, **backend-security-coder**, **terraform-specialist**, network-engineer |
| Samira Patel | **ai-engineer**, **ml-engineer**, **mlops-engineer**, **tdd-orchestrator**, **backend-security-coder**, deployment-engineer, fastapi-pro |

> **Bold** = `"model": "opus"` — deepest expertise tier, use for highest-complexity contributions.

---

## Changelog

| Version | Date | Change |
|---|---|---|
| v1.0 | Feb 18, 2026 | Initial CLAUDE.md — 5-stage workflow, canonical persona query |
| v2.0 | Mar 20, 2026 | Added Stage 2 Skill Activation; updated canonical query to include HAS_SKILL_SET; added bulk SkillSet load query; added skill-to-task mapping table; added activated_skills to Artifact node schema; added Skill Registry Reference; added inbox error handling rules; added Confluence MCP; updated relationship reference; added skill-grounded end-to-end workflow example |
| v2.1 | Mar 22, 2026 | Memo recording consistency fixes: (1) `to` field required on all Communication nodes; (2) `date` field required (YYYY-MM-DD) on Communication and Artifact nodes; (3) `body` must be full memo text — applies to response memos as well as outbound; (4) Confluence page IDs must be resolved to full URLs before recording; (5) `deliverables` JSON array added to Communication schema, updated after artifacts created; (6) Stage 5e added for deliverables update pattern; (7) Stage 6 rewritten with same field requirements as Stage 3; (8) `IN_REPLY_TO` documented as memo viewer body traversal path; (9) Canonical field reference tables added for Communication and Artifact nodes |

---

*Shirecom AI Consulting | QuadCode System | CLAUDE.md v2.1*
*Updated: March 22, 2026*
*Author: Wesley Gundersen (Principal Architect)*
