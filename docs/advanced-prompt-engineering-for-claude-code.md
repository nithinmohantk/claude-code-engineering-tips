# Advanced Prompt Engineering for Claude Code: Best Practices for Developers

> **Audience:** Software engineers, tech leads and architects using Claude Code for design, implementation, debugging and refactoring.
> **Scope:** Model-agnostic techniques that apply to current Claude models in Claude Code (CLI, IDE extensions, desktop and web).

---

## Table of Contents

1. [Core Mental Model](#1-core-mental-model)
2. [Solution Design & Architecture](#2-solution-design--architecture)
3. [Code Generation](#3-code-generation)
4. [Problem Solving & Debugging](#4-problem-solving--debugging)
5. [Context Management](#5-context-management)
6. [Iterative Refinement](#6-iterative-refinement)
7. [Structural Markers Reference (XML Tags)](#7-structural-markers-reference-xml-tags)
8. [Prompt Templates](#8-prompt-templates)
9. [Anti-Pattern Checklist](#9-anti-pattern-checklist)

---

## 1. Core Mental Model

### 1.1 How Claude Code actually works

Claude Code is an **agent running a loop**: read your prompt → decide on a tool (read a file, grep, run a command, edit) → observe the result → repeat until the task is done or it needs you. Four properties follow from that design and should shape every prompt you write.

| Property | What it means | How to adjust |
|---|---|---|
| **Finite context window** | Everything (your prompts, file contents it reads, command output, its own reasoning) shares one working memory. When it fills, older content is summarised (compacted) and detail is lost. | Point at files instead of pasting them. Start fresh sessions per task. Keep long logs out of the conversation. |
| **It only knows what it has read** | It does not "see" your repo up front. It discovers code by searching and reading. Anything it hasn't opened is an assumption. | Name the files, symbols and directories that matter. Tell it where *not* to look. |
| **Sequential, tool-grounded reasoning** | Quality comes from investigate → plan → act → verify. Skipping steps (e.g., "just fix it") makes it act on guesses. | Ask for exploration and a plan before edits on non-trivial tasks. Give it a way to verify (tests, linters, a command). |
| **Literal instruction following** | Current Claude models do what you ask, precisely. Vague requests get conservative, generic output; unstated constraints are not inferred. | State the outcome, the constraints and the definition of done explicitly. |

### 1.2 Persistent context: `CLAUDE.md`

`CLAUDE.md` files are loaded automatically at session start. They are your highest-leverage prompt because they apply to *every* request.

- `./CLAUDE.md` (checked in): build/test commands, architecture map, conventions, "never do X" rules.
- `~/.claude/CLAUDE.md`: personal preferences across all projects.
- Nested `CLAUDE.md` in subdirectories: module-specific rules, loaded when Claude works in that area.
- Run `/init` to bootstrap one, then edit it ruthlessly — short, specific, imperative.

**Bad `CLAUDE.md`:**

```markdown
This is a great project. Please write good, clean code and follow best practices.
```

**Optimized `CLAUDE.md`:**

```markdown
# Commands
- Build: `dotnet build src/Prescribing.sln`
- Unit tests: `dotnet test tests/Prescribing.UnitTests --no-build`
- Single test: `dotnet test --filter "FullyQualifiedName~<TestName>"`
- Lint/format: `dotnet format --verify-no-changes`

# Architecture
- Clean Architecture: `Domain` → `Application` → `Infrastructure` → `Api`. Domain has NO external package references.
- FHIR R4 resources are mapped in `src/Infrastructure/Fhir/Mappers/`. Never expose Firely SDK types from `Application`.

# Rules
- IMPORTANT: Never log patient identifiers (NHI, name, DOB). Use `PatientRef` (hashed) in logs.
- All new endpoints require an `[Authorize(Policy = ...)]` attribute and an audit event.
- Prefer `Result<T>` over exceptions for expected business failures.
- Run unit tests before declaring a task complete.
```

### 1.3 The adjustment in one sentence

> Treat Claude Code like a highly capable senior engineer who has just joined the team today: brilliant, fast, and literal — but with zero tribal knowledge until you give it, and a working memory that you must manage.

---

## 2. Solution Design & Architecture

The most expensive mistakes are architectural, and they are made before the first line of code. Force a design phase.

### 2.1 Separate thinking from doing

- Use **Plan Mode** (`Shift+Tab` to cycle permission modes in the CLI) so Claude can read and reason but not edit.
- Or say explicitly: *"Do not write or modify any code yet."*
- Ask it to **explore the existing codebase first** so the design fits what exists rather than an imagined greenfield.
- For hard trade-offs, ask it to reason deeply (e.g., "think hard about the failure modes") — this allocates more reasoning before it answers.

### 2.2 Give it the full problem frame

A design prompt should carry: the business problem, functional requirements, non-functional requirements (with numbers), constraints, the existing landscape, and the expected output format.

**Bad Prompt:**

```text
Design a notification system for our app.
```

**Optimized Prompt:**

```xml
<task>
Propose an architecture for a prescription-status notification capability.
Do NOT write implementation code. Produce a design for review.
</task>

<context>
- Existing services: see `docs/architecture/c4-containers.md` and `src/Services/`.
- Events already published to Azure Service Bus topic `prescription-events` (schema: `contracts/PrescriptionEvent.v2.json`).
- Patients opt in to SMS/email via the Consent service (`src/Services/Consent`).
</context>

<requirements>
<functional>
- Notify patients when a prescription is dispensed, cancelled, or ready for collection.
- Respect consent and channel preference per patient.
</functional>
<non_functional>
- 50k notifications/hour peak, p95 end-to-end latency < 60s.
- At-least-once delivery; no duplicate SMS to the patient (idempotency required).
- RTO 1h / RPO 5 min. Data residency: Australia East only.
</non_functional>
</requirements>

<constraints>
- Must run on our existing AKS cluster; no new PaaS products without justification.
- No PHI in notification payload bodies beyond first name and pharmacy name.
</constraints>

<output_format>
1. C4 context and container diagrams in Mermaid.
2. Sequence diagram for the "dispensed" flow, including failure and retry paths.
3. Two or three alternatives in a trade-off table (security, scalability, resilience, cost, operational complexity).
4. A recommended option written as an ADR (Context, Decision, Consequences).
5. Open questions and assumptions you made — list them explicitly, don't bury them.
</output_format>
```

### 2.3 Technology stack decisions

Ask for **evaluation against your criteria**, not opinions. Force it to state assumptions and when the recommendation would change.

**Bad Prompt:**

```text
Should we use Kafka or RabbitMQ?
```

**Optimized Prompt:**

```xml
<decision>Choose a message broker for inter-service domain events.</decision>

<options>Apache Kafka (Confluent Cloud), RabbitMQ (self-hosted on AKS), Azure Service Bus Premium</options>

<criteria weight="ordered">
1. Operational burden for a team of 4 platform engineers
2. Ordering guarantees per aggregate (prescription ID)
3. Replay capability for 7 days (audit reconstruction)
4. Cost at ~20M messages/day
5. Ecosystem fit with .NET 8 and Spring Boot 3
</criteria>

<instructions>
- Score each option per criterion with a one-line justification.
- State any assumption that materially affects the score.
- Recommend one option and describe the conditions under which you would change the recommendation.
- If information is missing to decide, say "Requires clarification" and list the question instead of guessing.
</instructions>
```

### 2.4 Component planning before implementation

Once the design is accepted, ask for a **file-level implementation plan** and review it before any edits:

```text
Based on the accepted ADR, produce an implementation plan:
- List every file you will create or modify, with a one-line purpose each.
- Order the steps so the build stays green after each step.
- Identify which steps need tests and what each test will assert.
- Stop after the plan. I will approve or amend it.
```

Save accepted plans to a file (e.g., `docs/plans/notifications.md`). It survives `/clear` and context compaction, and you can hand it to a fresh session.

---

## 3. Code Generation

### 3.1 The five constraints every code prompt should carry

1. **Location** — exact file(s), layer, and the pattern to copy (`"follow the structure of src/Api/Endpoints/OrdersEndpoints.cs"`).
2. **Contract** — signatures, inputs/outputs, status codes, schema.
3. **Conventions** — naming, error-handling model, logging style, libraries allowed/forbidden.
4. **Security and NFRs** — authz, input validation, secrets, PII handling, performance limits.
5. **Definition of done** — which tests must pass, which command verifies it.

**Bad Prompt:**

```text
Write an API endpoint to get a patient's medications.
```

**Optimized Prompt:**

```xml
<task>
Add GET /api/v1/patients/{patientId}/medications returning the patient's active MedicationRequests.
</task>

<reference_implementation>
Follow the exact structure and style of `src/Api/Endpoints/AllergiesEndpoints.cs`
and its handler `src/Application/Allergies/GetAllergiesQueryHandler.cs`.
</reference_implementation>

<contract>
- Response: FHIR R4 Bundle (type=searchset) of MedicationRequest, status=active only.
- 200 with empty bundle if none; 404 if patient does not exist; 403 if caller lacks the relationship.
- Pagination: `_count` (default 20, max 100), opaque continuation token.
</contract>

<conventions>
- MediatR query + handler in `Application`; repository interface in `Application`, implementation in `Infrastructure`.
- Return `Result<T>`; map to HTTP in the endpoint only.
- Structured logging via `ILogger<T>` with `PatientRef`, never raw patientId.
</conventions>

<security>
- Policy `Patient.Read.Clinical`; enforce care-relationship check via `ICareRelationshipService`.
- Validate `patientId` format; reject `_count` > 100 with 400 (OperationOutcome).
- Emit audit event `MedicationListViewed` (who, whom, when, purpose-of-use header).
</security>

<definition_of_done>
- Unit tests for the handler: active-only filter, 404, 403, pagination bounds.
- `dotnet build` and `dotnet test tests/Prescribing.UnitTests` pass.
- No new package references.
</definition_of_done>
```

### 3.2 Test-driven development with Claude

TDD is the most reliable way to get correct code from an agent because it gives Claude an **objective, executable target**.

**Bad Prompt:**

```text
Implement the dose calculator and add some tests.
```

**Optimized Prompt (three explicit phases):**

```xml
<phase_1>
Write failing unit tests ONLY for `DoseCalculator.CalculatePaediatricDose` in
`tests/Clinical.UnitTests/DoseCalculatorTests.cs`, covering:
- weight-based dosing (mg/kg) rounded down to 0.1 mg
- maximum adult dose cap
- weight <= 0 or > 150 kg returns a validation failure
- unit mismatch (g vs mg) handled explicitly
Do NOT implement the calculator. Run the tests and confirm they fail for the right reason.
</phase_1>

<phase_2>
After I confirm, implement the minimum code to make the tests pass.
Do NOT modify the tests. If a test seems wrong, stop and tell me why.
</phase_2>

<phase_3>
Refactor for clarity while keeping tests green. Run the full test suite at the end.
</phase_3>
```

> **Key line:** *"Do NOT modify the tests."* Without it, an agent under pressure to make tests pass may weaken the tests instead of fixing the code.

### 3.3 Security by default

State security requirements as constraints, not hopes. Name the specific threats.

**Bad Prompt:**

```text
Make sure it's secure.
```

**Optimized Prompt:**

```xml
<security_requirements>
- Parameterised queries only; no string concatenation into SQL (OWASP A03).
- Validate and bound every input (length, range, allowed characters) at the API boundary.
- Object-level authorisation on every resource access (OWASP API1: BOLA).
- No secrets in code or config; read from Key Vault via the existing `ISecretProvider`.
- Errors returned to clients must not include stack traces or internal identifiers.
</security_requirements>

After implementing, list each requirement above and point to the exact line(s) that satisfy it.
```

The final line forces **self-verification with evidence**, which surfaces gaps instead of assertions.

### 3.4 Idiomatic code: show, don't describe

A pointer to one good example in your codebase beats a paragraph of style rules.

- *"Match the style of `X`"* → consistent naming, error handling, DI patterns.
- *"Use the same libraries already used in this module; do not add dependencies"* → prevents dependency sprawl.
- *"Explain any deviation from the existing pattern"* → makes divergence visible.

### 3.5 Prefer explicit scope over open-ended "improve"

Claude follows instructions precisely. If you want **only** the change requested, say so; if you want it to go further (add tests, update docs, fix adjacent issues), say that too.

```text
Implement only what is described. Do not refactor unrelated code, rename existing symbols,
or add features I didn't ask for. If you notice other problems, list them at the end instead of fixing them.
```

---

## 4. Problem Solving & Debugging

### 4.1 The debugging prompt anatomy

Every debugging prompt needs: **symptom, expected behaviour, evidence, reproduction, what you've already ruled out, and environment.** Then ask for diagnosis before a fix.

**Bad Prompt:**

```text
The app is broken, it's throwing a null reference. Fix it.
```

**Optimized Prompt:**

```xml
<symptom>
POST /api/v1/prescriptions intermittently returns 500 (≈2% of requests) since release 4.12.0.
</symptom>

<expected>201 Created with the prescription resource.</expected>

<evidence>
Stack trace (trimmed):
System.NullReferenceException at PrescriptionFactory.Create(...) line 88
  at CreatePrescriptionHandler.Handle(...) line 41
Full log excerpt saved to `scratch/logs/incident-2291.log` — read only the lines with correlationId=7f3a….
</evidence>

<reproduction>
Not reproducible locally. Correlates with prescriber records migrated from the legacy system (no `HpiNumber`).
</reproduction>

<ruled_out>
- Not a DB timeout (no SQL errors in the window).
- Not the new FHIR validator (disabled via flag, still occurs).
</ruled_out>

<environment>.NET 8, EF Core 8, AKS, release 4.12.0 (diff vs 4.11.3: `git diff v4.11.3..v4.12.0 -- src/Domain`)</environment>

<instructions>
1. Investigate first. Read the relevant code paths and the release diff. Do not edit anything yet.
2. Give me your top 3 hypotheses ranked by likelihood, with the evidence for and against each.
3. Propose a test that would confirm the leading hypothesis.
4. Only after I confirm, write the failing test, then the fix.
</instructions>
```

### 4.2 Chain-of-thought reasoning for hard problems

For logic errors, concurrency bugs, and "it should work" problems, ask Claude to **reason explicitly before concluding**. This reduces premature fixes that address symptoms.

Techniques:

- **Request extended thinking** for genuinely hard problems: *"Think hard about this before answering"* (Claude Code allocates more reasoning budget when you ask it to think more deeply).
- **Ask for structured reasoning output** so you can audit it:

```xml
<instructions>
Before proposing a fix, reason through the problem inside <analysis> tags:
- Trace the data flow from the request to the failure point, step by step.
- At each step, state what values you expect and what could make them differ.
- Identify the earliest point where actual and expected behaviour diverge.
Then give your conclusion inside <root_cause> and the fix inside <fix>.
</instructions>
```

- **Hypothesis-driven debugging:** ask for multiple hypotheses and a discriminating experiment for each — not a single guess.
- **Instrument, don't speculate:** *"Add temporary logging at points A, B, C, run the reproduction, and report the actual values."* Grounded evidence beats reasoning about code in the abstract.

### 4.3 Tracebacks and logs: signal, not volume

**Bad Prompt:**

```text
[pastes 4,000 lines of log output]
What's wrong?
```

**Optimized Prompt:**

```text
The full log is at `scratch/logs/build-failure.log` (4k lines).
Grep it for the first ERROR and the 30 lines preceding it, then explain the root cause.
Ignore the deprecation WARN lines — they are pre-existing and unrelated.
```

Let Claude use tools (`grep`, `tail`, `jq`) to extract the relevant section. Pasting huge logs consumes context and dilutes attention.

### 4.4 Give it a feedback loop

Claude debugs best when it can **run something and observe**:

- Provide the exact reproduction command: `npm test -- -t "renders expired token banner"`.
- Ask it to *"run the reproduction after the fix and show the output."*
- For UI issues, provide screenshots (paste or drag images into Claude Code) and expected vs actual.

### 4.5 Defend against the "fix the symptom" trap

```text
Do not suppress the exception, add a null-check that hides the problem, or catch-and-ignore.
Find out WHY the value is null and fix it at the source. If the root cause is in data
rather than code, say so and propose both a code guard and a data remediation.
```

---

## 5. Context Management

Context is the scarcest resource in an agent session. Most hallucinations come from **missing context** (Claude fills gaps with plausible guesses) or **polluted context** (stale or irrelevant content that misleads).

### 5.1 Reference, don't paste

| Instead of… | Do this |
|---|---|
| Pasting a 500-line file | `@src/Services/OrderService.cs` or "read `src/Services/OrderService.cs`" |
| Pasting the entire schema | "Read the `prescriptions` and `dispense_events` tables in `db/migrations/V42__*.sql`" |
| Pasting library docs | Give the URL (Claude can fetch it) or a local path to the specific doc page; or connect an MCP server for the source |
| Describing a UI bug in prose | Paste a screenshot |

Use `@` mentions in Claude Code to include a file or directory directly in the prompt.

### 5.2 Scope the search space

**Bad Prompt:**

```text
Find where we handle authentication and add MFA.
```

**Optimized Prompt:**

```xml
<scope>
Authentication lives in `src/Identity/` and the gateway policy in `infra/apim/policies/`.
Ignore `legacy/` and `samples/` — they are not deployed.
</scope>

<task>
Explain the current login flow (sequence diagram, Mermaid) using only files in scope.
Cite file:line for each step. Do not modify anything.
</task>
```

### 5.3 Schemas, contracts and docs: the minimal sufficient set

Give Claude the **interfaces it must conform to**, not everything that exists:

- The relevant table DDL, not the whole database.
- The OpenAPI fragment / FHIR profile (StructureDefinition) for the resource you're touching.
- The one reference implementation it should mimic.
- Version numbers for frameworks and libraries (APIs differ across versions — this is a common hallucination source).

```xml
<versions>
.NET 8, EF Core 8.0.x, Firely SDK 5.x (R4), xUnit 2.x, FluentAssertions 6.x
</versions>
<instructions>
If you are unsure whether an API exists in these versions, check the package source or
documentation rather than assuming. Say so if you can't verify it.
</instructions>
```

### 5.4 Session hygiene

| Command / practice | When to use |
|---|---|
| `/clear` | Between unrelated tasks. A clean context beats a long one. |
| `/compact [focus]` | Mid-task when context is large; tell it what to preserve: `/compact keep the migration plan and failing test names`. |
| `/context` | Inspect what's consuming context. |
| `Esc` | Interrupt a wrong direction immediately rather than letting it run. |
| `Esc` `Esc` / `/rewind` | Roll back the conversation (and checkpointed edits) to an earlier point. |
| Write state to files | Plans, decisions and progress notes in `docs/plans/*.md` survive any context reset. |
| Subagents | Ask Claude to "use a subagent to investigate X" so heavy exploration happens in a separate context and only conclusions return. |

### 5.5 Anti-hallucination guardrails

Explicitly give Claude permission to not know:

```text
If a file, function, config key or API you need doesn't exist or you can't find it,
stop and tell me. Do not invent names, endpoints, environment variables or package versions.
Mark any assumption with "ASSUMPTION:" so I can verify it.
```

And require grounding:

```text
For every claim about how the current code behaves, cite the file and line number.
```

---

## 6. Iterative Refinement

### 6.1 Steer early, steer specifically

Course corrections are cheapest at the start. Review the first plan or first file before letting Claude proceed through the rest.

**Bad Prompt:**

```text
This isn't right. Try again.
```

**Optimized Prompt:**

```xml
<feedback>
The structure of `ConsentEvaluator` is good — keep it.
Two problems:
1. `Evaluate()` queries the DB once per rule (N+1). Load all consent directives for the patient in one query.
2. Expired directives are treated as "deny"; they must be treated as "not applicable" so the next rule in precedence applies.
</feedback>

<constraints>
Change only `ConsentEvaluator.cs` and its tests. Add a test that proves fix #2.
</constraints>
```

Specific feedback names **what to keep**, **what is wrong**, **why**, and **the boundary of the change**.

### 6.2 Refactoring with safety rails

**Bad Prompt:**

```text
Refactor this module to be cleaner.
```

**Optimized Prompt:**

```xml
<goal>
Refactor `src/Billing/InvoiceService.cs` (1,200 lines) into cohesive classes by responsibility.
</goal>

<invariants>
- Public API of `IInvoiceService` must not change.
- Behaviour must not change: all existing tests must pass unchanged.
- No new dependencies.
</invariants>

<process>
1. Before editing, run the existing tests and record the baseline.
2. If coverage of a method you will move is missing, add characterisation tests first.
3. Propose the target class breakdown (names + responsibilities) and wait for approval.
4. Refactor in small steps; run tests after each step.
5. Summarise each step with the test result.
</process>
```

### 6.3 Performance optimisation: measure first

**Bad Prompt:**

```text
Make this faster.
```

**Optimized Prompt:**

```xml
<problem>
`GET /reports/dispensing?month=…` has p95 of 8.2s; target is < 1s at current data volume (12M rows).
</problem>

<evidence>
Query plan attached in `scratch/perf/plan.txt`. App Insights shows 92% of time in `DispensingReportRepository.GetMonthly`.
</evidence>

<instructions>
1. Analyse the query plan and the repository code. Identify the top bottleneck with evidence.
2. Propose up to 3 optimisations, each with expected impact, risk and trade-off (e.g., index write cost, cache staleness).
3. Do not change behaviour or result ordering.
4. Implement only the option I choose, and add a benchmark or test that demonstrates the improvement.
</instructions>
```

### 6.4 Use review-mode prompts on your own output

After a feature is done, switch Claude into reviewer mode — ideally in a **fresh session** so it isn't anchored on its own reasoning:

```xml
<role>You are reviewing this change as a security-focused senior engineer.</role>
<scope>`git diff main...HEAD`</scope>
<checklist>
- Correctness and edge cases
- Authorisation and input validation
- Error handling and logging (no PII in logs)
- Test coverage of the new branches
- Consistency with conventions in CLAUDE.md
</checklist>
<output>
Findings classified CRITICAL / HIGH / MEDIUM / LOW with file:line and a concrete fix.
Do not report style preferences unless they violate a stated convention.
</output>
```

Claude Code also ships review commands (e.g., `/review`, `/security-review`) you can run directly.

### 6.5 Converge, don't loop

If you've corrected the same issue twice and it's still wrong, the context is likely polluted with failed attempts. **`/clear` and restart with a better prompt** that incorporates what you learned. This is almost always faster than a third correction.

---

## 7. Structural Markers Reference (XML Tags)

Claude is trained to pay close attention to XML-style tags. They separate instructions from data, make long prompts unambiguous, and let you reference sections by name ("using the constraints in `<constraints>`…"). Tag names are not magic — consistency and clarity are what matter.

### 7.1 Recommended tag vocabulary

| Tag | Purpose |
|---|---|
| `<task>` | The single, clear outcome you want. |
| `<context>` | Background: business problem, existing system, relevant files. |
| `<requirements>` / `<functional>` / `<non_functional>` | What must be true of the solution. |
| `<constraints>` | Hard limits: tech, scope, files that may/may not change. |
| `<conventions>` | Coding standards and patterns to follow. |
| `<security>` | Explicit security requirements. |
| `<reference_implementation>` | The example to mimic. |
| `<evidence>` | Logs, traces, metrics, screenshots. |
| `<ruled_out>` | Hypotheses already eliminated. |
| `<instructions>` / `<process>` | Ordered steps, including stop points. |
| `<definition_of_done>` | Objective completion criteria and verification commands. |
| `<output_format>` | Exactly what the response should contain. |
| `<example>` | A sample of desired input/output. |

### 7.2 Rules of thumb

- **Instructions outside or after data.** When you include large data (logs, specs), put it in its own tag and put the question/instructions after it.
- **Nest for hierarchy:** `<requirements><functional>…</functional><non_functional>…</non_functional></requirements>`.
- **Request tagged output** when you want to parse or review sections: `<analysis>`, `<plan>`, `<root_cause>`, `<fix>`.
- **Explain the "why" of a constraint.** "Never log patient identifiers *because logs are shipped to a third-party SIEM outside our data-residency boundary*" generalises better than a bare rule.
- **Don't over-structure trivial asks.** "Rename `getUsr` to `getUser` across `src/`" needs no tags.

---

## 8. Prompt Templates

Copy these into `.claude/commands/` as custom slash commands (or into skills) so the whole team uses them consistently. `$ARGUMENTS` is replaced with whatever follows the command.

### 8.1 `/design` — architecture before code

```markdown
<task>Design a solution for: $ARGUMENTS. Do not write implementation code.</task>
<instructions>
1. Explore the relevant parts of the codebase first and summarise what exists.
2. List functional and non-functional requirements; mark inferred ones as ASSUMPTION.
3. Provide C4 context + container diagrams (Mermaid) and one sequence diagram.
4. Compare 2–3 alternatives in a trade-off table (security, scalability, resilience, cost, operations).
5. Recommend one option as an ADR. List open questions.
</instructions>
```

### 8.2 `/implement` — plan, test, build, verify

```markdown
<task>$ARGUMENTS</task>
<process>
1. Read CLAUDE.md and the files relevant to the task. Identify the reference pattern to follow.
2. Produce a file-level plan and wait for my approval.
3. Write failing tests first; confirm they fail.
4. Implement until tests pass. Do not modify tests to make them pass.
5. Run build, lint and the full unit test suite. Report results.
6. Summarise: files changed, decisions made, anything left for follow-up.
</process>
```

### 8.3 `/debug` — hypothesis-driven investigation

```markdown
<problem>$ARGUMENTS</problem>
<instructions>
1. Investigate before editing: read code paths, recent changes (`git log -p` on suspect files), and logs.
2. Reason step by step inside <analysis>. Give top 3 hypotheses with evidence for/against.
3. Propose a discriminating test or instrumentation for the leading hypothesis and run it.
4. Once confirmed, write a regression test, then fix the root cause (not the symptom).
5. Show the passing test output.
</instructions>
```

### 8.4 `/review` (team variant)

```markdown
<scope>`git diff main...HEAD`</scope>
<instructions>
Review for correctness, security (OWASP ASVS / API Top 10), performance, maintainability and test coverage.
Classify findings CRITICAL/HIGH/MEDIUM/LOW/INFORMATIONAL with file:line, rationale and a concrete fix.
If something cannot be assessed from the diff, mark it "Requires clarification".
</instructions>
```

---

## 9. Anti-Pattern Checklist

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| "Fix it" / "Make it better" | No target; Claude guesses scope and success criteria. | State the outcome and definition of done. |
| Coding before designing | Locks in architecture by accident. | Plan Mode or "do not write code yet". |
| Pasting whole files/logs | Burns context; dilutes attention. | Reference paths; let Claude grep. |
| No verification step | Claude can't tell if it succeeded. | Provide test/build/lint commands. |
| Unstated constraints | Claude doesn't infer them. | Write them in the prompt or `CLAUDE.md`. |
| One giant session for everything | Stale context causes drift and errors. | `/clear` between tasks; persist plans to files. |
| Repeated vague corrections | Context fills with failed attempts. | Specific feedback; restart after two misses. |
| Letting tests be edited to pass | Hides real defects. | "Do not modify tests." |
| Accepting unverified claims | Plausible ≠ correct. | Require file:line citations and command output. |
| Ignoring versions | APIs hallucinated across versions. | Pin framework/library versions in the prompt. |

---

### Summary: the five habits that matter most

1. **Invest in `CLAUDE.md`.** It's the prompt you write once and benefit from forever.
2. **Explore → Plan → Implement → Verify.** Don't let any step be skipped on non-trivial work.
3. **Structure prompts with XML tags** — task, context, constraints, definition of done, output format.
4. **Manage context deliberately.** Reference, scope, clear, compact, and persist state to files.
5. **Make success executable.** Tests and commands turn "looks right" into "is right".
