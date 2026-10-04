---
name: openfasttrace
description: Work with OpenFastTrace requirement tracing, including specification items, artifact IDs, coverage markers, Markdown and Gherkin syntax, and trace validation. Use when you need to create, edit, review, or validate OpenFastTrace-traced requirements, design, implementation, tests, or documentation.
---

# OpenFastTrace (OFT) Skill

OpenFastTrace is a tool for requirement tracing across various artifacts (specifications, code, tests).

## Core Concepts

- **Specification Items**: Normative pieces of specification or coverage markers.
- **Artifact Types**: Dynamic and not hard-coded. New types exist automatically when used in a document.
  - Common: `feat`, `req`, `arch`, `dsn`, `impl`, `utest`, `itest`, `stest`, `uman`, `oman`.
- **ID Syntax**: `type~name~revision` (e.g., `req~login-feature~1`).
  - `name`: Hierarchical with dots (e.g., `ui.button.save`).
  - `revision`: Integer (starts from 1) used for versioning.
    - Incrementing the revision breaks all incoming links (coverage and dependencies).
    - This forces covering items to be updated and re-verified.
- **Keywords**:
  - `Covers: <ID>`: Current item implements/details the target ID. To specify ID, you need to use the unordered list (inline ID is not supported):
    ```markdown
    Covers:
    - <ID>
    ```
  - `Needs: <types>`: Artifact types required to cover this item, separated by comma.
  - `Status: <status>`: Possible values are `draft`, `proposed`, `approved`, `rejected`. Has to occur before `Description`, `Rationale` or `Comment`.
  - `Depends: <IDs>`: Defines dependencies (no effect on coverage). Write an unordered list like in `Covers` (inline ID is not supported).
  - `Description: <text>`: Optional keyword to start description.
  - `Rationale: <text>`, `Comment: <text>`.
  - `Tags: <tags>`: Optional tags.

## Syntax

### Markdown

```markdown
### Title
`req~id~1`
Description of the requirement.

Rationale: Why this is needed.

Covers:
- feat~parent~1

Needs: dsn, impl, utest
```

The identifier must be placed on the line immediately below the Markdown heading with **no empty line**.

- **Forwarding**: `arch --> dsn : req~id~1` (delegates coverage without repeating).
- **Exclusion**: Use `<!-- oft:off -->` and `<!-- oft:on -->` to skip parsing.

If the project uses `markdownlint`, add this line at the end of the Markdown file with OFT specification items:

```
<!-- markdownlint-disable-file MD022 -->
```

### Coverage Tags (many file formats)

Implementation covering design in a Java/C++ file:

```
// [utest -> dsn~hash-sum-calculation~1]
```

Coverage in a YAML file (e.g., GitHub workflow):

```yaml
# [bld->dsn~create-sbom~2]
```

Coverage in a Markdown file:

```markdown
<!-- [uman->feat~ai-skill~1] -->
```

Require coverage:

```plantuml
' [req -> dsn~hash-sum-calculation~1 >> impl, utest]
```

Multiple coverage:

```Java
// [dsn -> req~local-stability~1,arch~dimensional-input~1]
```

### Gherkin

Gherkin `.feature` files can define OFT scenario items. Put exactly one OFT ID
in the contiguous tag region immediately before a `Scenario` or `Scenario
Outline`. Optional `# Covers:` and `# Needs:` comments belong between the tags
and the scenario header. Multiple `Covers` comments accumulate IDs; `Needs`
may appear once.

```gherkin
@id:scn~user-login~1
# Covers: req~authentication~1
# Needs: dsn, itest
Scenario: User logs in
  Given a registered user
  When valid credentials are entered
  Then access is granted
```

Basic coverage tags are recognized only in Gherkin comments, for example
`# [impl~login~1 -> dsn~authentication~1]`. Executable Gherkin lines are not
evaluated for coverage tags.

Tracing can be performed via CLI, Maven, or Gradle.

## CLI Usage

General form: `oft <command> [options] <files/dirs>`

- **Commands**: `trace` (generate report), `convert` (export format), `help` (usage and version).
- **Options for `convert` and `trace`**:
  - `-o, --output-format`: `plain`, `html`, `aspec` (XML).
  - `-f, --output-file`: File path (default STDOUT).
  - `-a, --wanted-artifact-types`: Filter by type (Partial Tracing).
  - `-t, --wanted-tags`: Filter by tags (Partial Tracing). Use `_` for items without tags (e.g., `-t _,MyTag`).
  - `-v, --report-verbosity`: `quiet`, `minimal`, `summary`, `failures`, `failure_summaries`, `failure_details` (default), `overview`, `all`.
  - `-i, --ignore-artifact-types`: Exclude types from import.

Exit codes:

- `0`: Success.
- `1`: OFT error.
- `2`: Command line error.

## Maven Integration

- **User Guide**: [openfasttrace-maven-plugin](https://github.com/itsallcode/openfasttrace-maven-plugin)

Add the `openfasttrace-maven-plugin` to your `pom.xml`:

```xml
<plugin>
    <groupId>org.itsallcode.openfasttrace</groupId>
    <artifactId>openfasttrace-maven-plugin</artifactId>
    <version>VERSION</version>
    <executions>
        <execution>
            <goals><goal>trace</goal></goals>
        </execution>
    </executions>
    <configuration>
        <reportFormat>html</reportFormat>
        <reportFile>target/site/tracing.html</reportFile>
    </configuration>
</plugin>
```

- **Run**: `mvn openfasttrace:trace`

## Gradle Integration

- **User Guide**: [openfasttrace-gradle](https://github.com/itsallcode/openfasttrace-gradle)

Apply the plugin in `build.gradle`:

```gradle
plugins {
    id "org.itsallcode.openfasttrace" version "VERSION"
}

openfasttrace {
    reportFormat = "html"
}
```

- **Run**: `gradle trace`

## Partial Tracing & Filtering

Partial tracing allows teams to focus on specific layers of the traceability chain, reducing noise and build time.

**Example Scenario:**
- **Product Owner (PO)**: Writes system requirements (`req`). Traces `feat` → `req` to ensure all features are specified.
- **Architect**: Writes design specifications (`dsn`). Traces `feat` + `req` → `dsn` to ensure requirements are architecturally covered.
- **Developer**: Writes implementation (`impl`) and tests (`utest`). Traces `feat`+ … + `dsn` → `impl`, `utest` to verify complete implementation and testing of the design.

```text
[PO] --(feat)--> [req]
                   |
[Architect] -------+--(feat, req)--> [dsn]
                                       |
[Developer] ---------------------------+--(feat, ..., dsn)--> [impl], [utest]
```

**Usage:**
- Filter by artifact types: `oft trace -a req,dsn <dir>`
- Filter by tags: `oft trace -t MyTag <dir>`
- Combine filters to focus on specific components or requirement levels.

## LLM Interaction Guidelines

- When identifying coverage, look for `impl~<ID>`, `utest~<ID>`, `itest~<ID>` and `stest~<ID>` in comments.
- Place markers at the narrowest possible scope (method/class).
- Ensure ID consistency across specifications and code.
- **Semantic Changes**: Increment the revision when the meaning of a requirement changes. This enforces a check of all covering items as their links become invalid.
- Verify changes by running tracing.
- Always follow the project guidelines for writing the requirements. In particular, they should answer:
  - Where to put the requirements.
  - How to structure or group the requirements.
  - Style of the title and description.
  - How the traceability chain looks like.
  - Which artifact types are used in the project.
- If there are no guidelines for requirements, as a fallback, use only `req` type for all requirements and `impl`, `utest` for `Needs`.
