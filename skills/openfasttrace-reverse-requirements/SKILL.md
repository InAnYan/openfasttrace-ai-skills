---
name: oft-reverse-requirements
description: Reverse-engineer requirements from existing source code and tests into OpenFastTrace specification items, link implementation and test coverage markers, and adapt to the project's requirement guidelines. Use when an AI agent needs to deduce requirements from legacy or untraced code, identify business rules and edge cases, and trace them using OpenFastTrace.
---

# OFT Reverse Requirements from Code

Extract clean, stable domain requirements from existing source code and tests, represent them as OpenFastTrace (OFT) specification items, and establish bidirectional traceability with implementation and test coverage markers.

## OFT Reference and Organizational Guidelines

For basic OpenFastTrace concepts, item ID syntax (`type~name~revision`), coverage markers (`[impl->...]`, `[utest->...]`), and trace verification, consult the main OpenFastTrace skill:

`https://github.com/itsallcode/openfasttrace-ai-skills/blob/main/skills/openfasttrace/SKILL.md`

### Organizational Guidelines and Tracing Chains

OpenFastTrace is an unopinionated, flexible tracing tool. Different organizations and projects establish their own requirement guidelines, artifact types, and tracing chains:

- **Tracing Chains**: Projects typically define chains from unrefined, high-level artifacts to refined, lower-level specifications (e.g., `feat --> req --> impl/utest`, `epic --> story --> impl/utest`, or chains incorporating architecture `arch --> dsn`).
- **Storage Locations**: Each project organizes specifications according to its own structure (e.g., `doc/specs/`, `spec/system/`, `docs/requirements/`).
- **Style and Formatting**: Projects may use Markdown, AsciiDoc, reStructuredText, or Gherkin feature files, each with project-specific phrasing or linting standards.

The AI agent must always search for and follow the target repository's established documentation, requirement guides, and architecture records before writing specifications.

### Inquiring When Guidelines Are Missing

If the project does not have documented requirement guidelines or directory conventions:

1. **Ask the user explicitly**:
   - What traceability chain and artifact types to use (e.g. `feat --> req`).
   - Where the specification files should be stored.
   - What naming conventions or formatting rules should be followed.
2. **Propose a sensible fallback**: If the user has no preference, propose `feat --> req --> impl/utest` stored in a dedicated specifications directory (e.g., `doc/specs/`).

---

## Core Principles

### 1. Abstract Domain Rules, Do Not Codify Implementation Details

- **Separate Business Rules from Internal Mechanics**: Capture *what* the system must do from a domain perspective (functional rules, boundary conditions, business constraints), not *how* the code computes it (algorithms, heuristics, scoring weights, threshold values, internal caches, or data structures).
- **Avoid Encoding API or Class Mechanics**: Do not write requirements for internal class methods, helper utilities, or interface contracts. Requirements describe system behavior, not class signatures.
- **Ensure Specification Stability**: Requirements must remain valid even if the underlying algorithm, scoring model, or class architecture is refactored.

### 2. Follow TDD Inspection Order When Applicable

- **Tests First in TDD Environments**: If the codebase follows Test-Driven Development (TDD) or has comprehensive, expressive unit tests, inspect the **unit tests first**.
- **Tests Reveal Intent**: Well-written unit tests clearly express the intended behavioral contract, boundary conditions, and edge cases without the cognitive noise of low-level implementation details.
- **Implementation Second**: After identifying the domain rules from tests, inspect the implementation to verify rule execution order, guards, and uncovered branches.

### 3. Make Titles Self-Contained and Strictly Omit Redundant Body Text

- **Titles Carry the Full Constraint**: Formulate headings so they clearly express the complete normative domain constraint. Anyone reading only the heading should understand the rule.
- **Strictly Omit Redundant Descriptions**: If the heading expresses the constraint, **do not write explanatory paragraphs under it**. Paraphrasing the heading in prose adds maintenance bloat. Proceed directly from the requirement tag to `Needs:` (or `Rationale:` / priority notes).
- **Use `Rationale:` Strictly for the "Why"**: Use `Rationale:` *only* when an exception, non-obvious constraint, or business reason requires explanation (e.g., explaining why a generic fallback type is allowed to match specific types).
- **Avoid Overstating Capabilities**: State only what the system factually guarantees. Do not promise broader features or normalizations in rationales that the code and tests do not actually implement.

### 4. Tag at the Point of Usage, Not Definition

- **Call-Site Placement for `[impl->...]`**:
  - Place implementation coverage tags where the rule is **evaluated, dispatched, or enforced** in the primary workflow.
  - Avoid placing tags on private helper method declarations, as helper methods are implementation details subject to inlining or refactoring.
  - In compound boolean expressions, place specific tags before each relevant clause or branch.
- **Targeted Placement for `[utest->...]`**:
  - Place test coverage tags directly on the specific test methods or test cases that verify each requirement.

---

## Reverse-Engineering Workflow

### Step 1: Discover Project Guidelines

1. Look for repository documentation regarding requirement tracing (e.g., developer guides, architecture documentation, contributing guides, or existing specification files).
2. If guidelines are absent or ambiguous, ask the user for the preferred tracing chain and destination folder.

### Step 2: Analyze Tests and Code Flow

1. **If TDD or thorough test suite exists**:
   - Read test classes, parameterized test datasets, and assertion messages.
   - Group tests by the behavioral rules and edge cases they validate.
2. **Examine Implementation Logic**:
   - Identify entry point methods and primary decision paths.
   - Trace conditional guards, early exits, and fallback behavior.
   - Note rule precedence: in sequential evaluations, record which rules take priority over others.

### Step 3: Extract Features and Requirements

1. **Do not limit to a single feature**: Identify as many high-level features (`feat` or project equivalent) and detailed requirements (`req` or project equivalent) as necessary to accurately capture the analyzed scope.
2. Ensure each requirement represents an independent, cohesive domain rule with a self-contained title and no redundant body narrative.

### Step 4: Illustrative Example (Entity Comparison)

#### 1. Untraced Source Code and Tests

Start with the existing untraced implementation code:

```java
public boolean isDuplicate(final Entity recordOne, final Entity recordTwo) {
    if (haveSameIdentifier(recordOne, recordTwo)) {
        return true;
    }

    if (haveIncompatibleTypes(recordOne, recordTwo) ||
        haveDifferentEditions(recordOne, recordTwo)) {
        return false;
    }
    ...
```

And its existing untraced unit tests:

```java
@Test
void recordsWithIdenticalIdentifiersMatch() {
    ...
}

@Test
void recordsWithDifferentTypesDoNotMatchUnlessGeneric() {
    ...
}

@Test
void recordsWithDifferentEditionsDoNotMatch() {
    ...
}
```

#### 2. Derived Requirements

From the code logic and test cases, deduce the feature and domain requirements. Note that requirements with self-evident headings contain **no redundant body paragraphs**:

```markdown
# User can detect duplicate records
`feat~entity-deduplication~1`

The system detects whether two records represent the same entity.

Needs: req

## Records with identical unique identifiers are considered duplicates
`req~entity-deduplication.same-identifier~1`

This rule has priority over subsequent field comparisons.

Needs: impl, utest

Covers:

- feat~entity-deduplication~1

## Records with differing editions are not duplicates
`req~entity-deduplication.different-editions~1`

Needs: impl, utest

Covers:

- feat~entity-deduplication~1

## Records with incompatible entity types are not duplicates unless one is a generic type
`req~entity-deduplication.type-compatibility~1`

Rationale: Generic fallback types may represent unclassified records of any type and are allowed to match more specific types.

Needs: impl, utest

Covers:

- feat~entity-deduplication~1
```

#### 3. Tagged Implementation at Usage Points

Add `[impl->...]` tags directly where each rule is evaluated:

```java
public boolean isDuplicate(final Entity recordOne, final Entity recordTwo) {
    // [impl->req~entity-deduplication.same-identifier~1]
    if (haveSameIdentifier(recordOne, recordTwo)) {
        return true;
    }

    if (
        // [impl->req~entity-deduplication.type-compatibility~1]
        haveIncompatibleTypes(recordOne, recordTwo) ||
        // [impl->req~entity-deduplication.different-editions~1]
        haveDifferentEditions(recordOne, recordTwo)
    ) {
        return false;
    }
    ...
```

#### 4. Tagged Unit Tests

Add `[utest->...]` tags to the corresponding test methods:

```java
// [utest->req~entity-deduplication.same-identifier~1]
@Test
void recordsWithIdenticalIdentifiersMatch() {
    ...
}

// [utest->req~entity-deduplication.type-compatibility~1]
@Test
void recordsWithDifferentTypesDoNotMatchUnlessGeneric() {
    ...
}

// [utest->req~entity-deduplication.different-editions~1]
@Test
void recordsWithDifferentEditionsDoNotMatch() {
    ...
}
```

### Step 5: Verify Tracing and Project Quality Gates

1. **Run OpenFastTrace Verification**: Run the project's OFT command (e.g., Gradle task, Maven plugin, or `oft` CLI) according to project conventions:
   - Ensure all high-level items have covering requirements.
   - Ensure all requirements have covering `impl` and `utest` tags.
   - Ensure there are no broken links or missing dependencies.
2. **Run Project Linters**: Run the appropriate linters for the specification file formats (e.g. Markdown or reStructuredText linters) and code linters configured in the repository.
3. **Execute Test Suite**: Run the relevant test suite to confirm all tests pass and coverage is active.

### Step 6: Present Results with Draft Disclaimer

When presenting the reverse-engineered requirements to the user, inform them that the generated specifications should be considered a draft proposal and a strong starting point:

- **Potential Over-Coverage**: Reverse-engineering from code and tests may occasionally capture defensive guards, implementation heuristics, or auxiliary utilities that the project may not intend to treat as formal domain requirements.
- **Implementation Artifacts**: The inferred requirements reflect the existing codebase's technical choices and legacy assumptions, which might not represent the desired target architecture.
- **Human Review**: Advise the user to review the generated items to confirm domain boundaries, prune unnecessary specifications, and adapt the phrasing before committing them as definitive project requirements.
