---
name: oft-reverse-requirements
description: Reverse-engineer domain requirements from existing source code and unit tests into OpenFastTrace specification items, link implementation and test coverage markers, and ensure clean separation between domain rules and implementation mechanics. Use when an AI agent needs to deduce requirements from legacy or untraced code, formulate self-evident normative requirements, and trace them using OpenFastTrace.
---

# OFT Reverse Requirements from Code

Extract clean, stable domain requirements from existing source code and tests, represent them as OpenFastTrace (OFT) specification items, and establish bidirectional traceability with `[impl->...]` and `[utest->...]` coverage markers.

## Core Philosophy

When reverse-engineering specifications from an existing codebase, the goal is to capture **what** the system must do from a domain perspective (business rules, functional constraints, boundary conditions), **not how** the code currently computes or organizes it (algorithms, heuristics, scoring weights, internal class APIs, or data structures).

Requirements must remain stable over time. If an internal algorithm, heuristic threshold, or class refactoring changes tomorrow, well-formulated requirements will continue to stand.

---

## Golden Rules

### 1. Abstract Domain Rules, Do Not Codify Implementation Details

- **Avoid Algorithmic Specifics**: Do not encode weights, mathematical formulas, scoring thresholds, or heuristic steps into requirements (e.g., `similarity > 0.8` or `weights map`). These are implementation choices subject to tuning and refactoring.
- **Avoid API / Class Mechanics**: Do not write requirements for internal class methods, helper utilities, or interface contracts (e.g., `compareEntriesStrictly` or `CollectionDuplicateDetector`). Requirements describe system behavior, not class signatures.
- **Focus on the Domain Invariant**: Ask: *What real-world or domain fact makes this condition true or false?* (e.g., "Entries for different parts of the same book must not be considered duplicates").

### 2. Formulate Self-Evident, Normative Headings

- **`feat~...~` (Feature)**: Formulate as **"User can verb"** (user capability, e.g., `# User can detect duplicate entries`).
- **`req~...~` (Requirement)**: Formulate as **"Subject must verb"** (normative constraint, e.g., `## Entries with differing editions must not be considered duplicates`).
- **Heading Carries the Complete Constraint**: Anyone reading only the heading should understand the full normative rule without reading the body text.
- **Avoid Nominalizations**: Write "must be verified" instead of "verification", "must normalize" instead of "normalization".
- **One Item, One Rule**: If a title contains "and", consider splitting it into two separate requirements unless the conditions are strictly inseparable.

### 3. Keep Descriptions Lean and Rationales Factual

- **Do Not Restate the Title**: If the heading is self-evident, omit redundant descriptive paragraphs.
- **Do Not Leak Implementation in Descriptions**: Never describe code paths, variable names, or class structures in the requirement body.
- **Use `Rationale:` Strictly for the "Why"**:
  - Explain why an exception exists (e.g., why a generic `misc` type is permitted to match other types).
  - Explain why a constraint is necessary (e.g., why chapter entries sharing an ISBN need further verification).
- **Be Factually Accurate (Do Not Overstate Capabilities)**:
  - Describe only the exact behavior the code guarantees.
  - *Example*: If the code only strips periods from journal names (`replace(".", "")`) and lowercases them, do **not** claim in the rationale that "journal abbreviations are harmonized". State factually that "journal names (ignoring periods and casing)" are normalized.

### 4. Tag at the Point of Usage, Not Definition

- **Call-Site Placement for `[impl->...]`**:
  - Place implementation coverage tags where the rule is **evaluated, dispatched, or enforced** in the primary business logic flow.
  - **Do NOT** place tags on private helper method declarations. Helper methods are implementation details that may be inlined, renamed, or extracted.
  - In compound conditions (`if (ruleA || ruleB || ruleC)`), place the specific tag directly before each clause or branch.
- **Targeted Placement for `[utest->...]`**:
  - Place unit test tags on the specific test methods or parameterized tests that verify that exact requirement.

---

## Reverse-Engineering Workflow

### Step 1: Trace Code Flow and Decision Points

1. **Locate Primary Entry Points**: Find the public methods that make decisions (e.g., `isDuplicate(entry1, entry2)`).
2. **Identify Decision Trees and Guards**:
   - What causes immediate positive outcomes (early returns, matches)?
   - What causes immediate negative outcomes (guards, exclusions, incompatibilities)?
   - What falls through to deeper checks or similarity heuristics?
3. **Identify Rule Precedence**:
   - Imperative code evaluates sequentially. If one rule takes absolute priority (e.g., identical persistent identifiers override entry type checks), explicitly document that precedence.

### Step 2: Analyze Tests for Domain Intent

1. Examine unit test method names and assertions (e.g., `@Test void bookAndInBookWithSameTitleIsNotDuplicate()`).
2. Tests often express domain rules and edge cases far more clearly than the implementation code.
3. Group tests by the behavioral rule they validate.

### Step 3: Extract and Formulate Requirements

1. Define the parent feature:

   ```markdown
   # User can detect duplicate entries
   `feat~duplicate-check~1`

   JabRef can detect whether two bibliographic entries represent the same publication.

   Needs: req
   ```

2. Derive each specific requirement covering the feature:

   ```markdown
   ## Entries with identical persistent identifiers must be considered duplicates
   `req~duplicate-check.same-identifier~1`

   This rule has the highest priority over the others listed below.

   Needs: impl, utest

   Covers:

   - feat~duplicate-check~1
   ```

3. Add rationale where domain context or exceptions need explanation:

   ```markdown
   ## Entries with different entry types must not be considered duplicates unless one is misc
   `req~duplicate-check.entry-type-compatibility~1`

   Rationale: Entries with the generic `misc` type may represent unclassified records of any type and can therefore match entries of other types.

   Needs: impl, utest

   Covers:

   - feat~duplicate-check~1
   ```

### Step 4: Link Implementation (`[impl->...]`) at Usage Sites

Annotate the code where the rule is evaluated in the primary workflow:

```java
public boolean isDuplicate(final BibEntry one, final BibEntry two) {
    // [impl->req~duplicate-check.same-identifier~1]
    if (haveSameIdentifier(one, two)) {
        return true;
    }

    if (
        // [impl->req~duplicate-check.entry-type-compatibility~1]
        haveDifferentEntryType(one, two) ||
        // [impl->req~duplicate-check.different-editions~1]
        haveDifferentEditions(one, two) ||
        // [impl->req~duplicate-check.book-parts-different-chapters-or-pages~1]
        haveDifferentChaptersOrPagesOfTheSameBook(one, two)
    ) {
        return false;
    }
```

### Step 5: Link Tests (`[utest->...]`)

Annotate the corresponding test methods:

```java
// [utest->req~duplicate-check.same-identifier~1]
@Test
void sameDoiMeansDuplicate() {
    ...
}

// [utest->req~duplicate-check.entry-type-compatibility~1]
@Test
void differentTypesAreNotDuplicatesUnlessMisc() {
    ...
}
```

### Step 6: Validate Tracing and Code Quality

1. **OFT Tracing Verification**: Run the project's OFT task (e.g. `./gradlew traceRequirements` or `oft check`) and verify:
   - All `feat` items have covering `req` items.
   - All `req` items have covering `impl` and `utest` items.
   - Zero broken or dangling references.
2. **Linter & Style Verification**:
   - Check Markdown formatting (`npx markdownlint-cli2`).
   - Run code linters (`checkstyle`, `spotless`, etc.).
   - Run unit tests to confirm all tests pass.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | What to Do Instead |
| :--- | :--- | :--- |
| **Encoding Thresholds** (e.g., similarity score must exceed 0.8) | Ties requirement to a specific mathematical formula or weight matrix. Breaks if heuristic changes. | **State Domain Rule** (e.g., Field comparison must normalize specialized fields before similarity matching). |
| **Specifying API Signatures** (e.g., DuplicateCheck must provide compareEntriesStrictly) | Interface mechanics are code architecture, not business requirements. | Focus on the domain behavior or functional result. |
| **Tagging Method Definitions** (e.g., tagging `private boolean helper(...)`) | Helper methods can be refactored or reused. The rule is executed at the call site. | Tag at the **usage / decision site** where the rule is evaluated in the workflow. |
| **Overstating Capabilities** (e.g., stating abbreviations are harmonized when only periods are dropped) | Misleads reviewers and users; promises behavior that is not tested or implemented. | State accurately: `journal names (ignoring periods and casing) are normalized`. |
| **Verbose Repetition** (e.g., paragraph in body repeating the heading in different words) | Adds maintenance noise without adding value. | If heading is self-evident, omit body text. Use body only for `Rationale:` or rule precedence. |
