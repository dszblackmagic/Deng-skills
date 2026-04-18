---
name: highton-weaver
description: Generate or update Weaver OA integrations from Weaver open-doc links or textual API docs for layered Java projects. Use when the task is to convert a Weaver API document into project-native code across provider, service, implementation, and test layers, including request/response DTOs, service contracts, implementation-layer HTTP encapsulation, constants/enums, and project-specific adaptation rules.
---

# Highton Weaver

## Overview

Use this skill to turn a Weaver OA API document into layered Java encapsulation for the current project.

Read the document first, then map it into the target project's layering.

Common target chain:

- `Provider`
- `Service`
- `Impl`
- `Test`
- request/response DTOs
- constants/enums if the project uses them

For the detailed repository rules, read:

- [references/layered-weaver-rules.md](references/layered-weaver-rules.md)
- [references/architecture-detection.md](references/architecture-detection.md)
- [references/similar-interface-strategy.md](references/similar-interface-strategy.md)
- [references/delivery-report.md](references/delivery-report.md)
- [references/opendoc-link-parsing.md](references/opendoc-link-parsing.md)

## Workflow

### 1. Detect the target architecture

Before writing code, inspect the current project and decide which delivery chain is correct.

Use [references/architecture-detection.md](references/architecture-detection.md).

Prefer the existing architecture over the default chain.

Typical outcomes:

- `Provider -> Service -> Impl -> Test`
- `Facade -> Service -> Impl`
- `Controller -> Service -> Client`
- `Controller -> Service -> Gateway -> Impl`
- direct `Service -> Impl` when no outer provider layer exists

Decide these items first:

- what is the outermost callable layer
- where OA HTTP code should live
- whether DTOs belong to API, service, or web modules
- which test class should carry the new interface method tests
- whether tests should be integration tests or unit tests
- whether constants/enums belong in shared infrastructure or feature-local files

### 2. Read the OA document

If the user gives a Weaver open-doc URL, first try to resolve the actual JSON document instead of reading only the HTML shell page.

Use [references/opendoc-link-parsing.md](references/opendoc-link-parsing.md).

Extract these items before writing code:

- API path
- HTTP method
- whether `access_token` is in query or body
- whether `userid` is required
- request fields, nested objects, arrays, and paging fields
- response envelope shape
- response `data` shape
- whether the document separates `count` and `list` endpoints

### 3. Rebuild the contract in project terms

Map the OA document into the target project's standard abstraction.

Before creating new files, look for the closest existing Weaver-style or third-party integration as a template.

Use [references/similar-interface-strategy.md](references/similar-interface-strategy.md).

First inspect the existing codebase and identify:

- whether the project exposes `Provider`, `Facade`, `Controller`, or direct `Service`
- whether implementations live in `Impl` classes
- whether DTOs are split into `request/response` packages
- whether constants and enums are centralized
- whether tests already exist for similar interfaces and where they live

Then adapt the output to that style.

Prefer these options in order:

1. extend an existing similar interface chain
2. reuse existing DTOs or nested DTO patterns
3. copy the nearest implementation pattern and only change the document-driven parts
4. create new files only when no good existing carrier exists

Default assumptions when the project has no stronger convention:

- external caller passes project-facing identity fields, not OA `userid`
- service layer converts project-facing identity to OA `userid`
- outer delegation layer is a pure pass-through
- service implementation performs the actual OA HTTP request
- response is always wrapped in `Response<T>`
- generate a matching test class or extend the closest existing test class for the new method

Use [references/layered-weaver-rules.md](references/layered-weaver-rules.md).

### 4. Add or update the required files

For a new Weaver API, update the smallest complete set of files required by the current project.

Typical set:

- provider or facade interface
- service interface
- implementation class
- request DTOs
- response DTOs
- constants/enums if the project uses them
- corresponding test class and test methods

Default expectation: after finishing the interface chain, also generate the corresponding test class or test methods according to the project's existing test style.

Only skip test generation when one of these is true:

- the repository rules explicitly prohibit adding or running tests
- the user explicitly says not to add tests
- the project truly has no meaningful place to carry the test and adding one would be misleading

### 5. Produce a delivery summary

After code generation, output a compact delivery summary that explains what was generated and why.

Use [references/delivery-report.md](references/delivery-report.md).

The summary should explicitly state:

- which architecture chain was chosen
- which existing interface or integration was used as the closest template
- which files were created or modified
- which request and response structures were mapped directly from the document
- which fields were adapted from project-facing semantics to OA-facing semantics
- whether `access_token` is in query or body
- whether count/list dual endpoints were implemented
- which test class and test methods were added
- whether tests were intentionally skipped and why

## Implementation Rules

- Follow the existing Weaver integration style in the target project before inventing a new style.
- Keep the outer layer thin. A `Provider`, facade, controller, or gateway entrypoint should usually delegate downward instead of owning OA HTTP details.
- Keep HTTP orchestration in the implementation layer.
- Preserve OA response field names in DTOs unless the repository already normalizes that field family.
- Prefer strong typed nested DTOs for `condition` or other structured filters.
- Decide `access_token` placement strictly from the document. Do not assume all Weaver APIs use the same placement.
- If the OA capability is naturally split into `count` and `list`, implement both methods when total count is needed by the project-facing response.
- Reuse existing shared DTOs when the response shape already matches an existing class.
- When generating a new Weaver API, also generate the corresponding method tests based on the project's prevailing test style.
- Prefer extending the nearest existing test class before creating a brand-new test style.
- If tests cannot be added because of repository or user constraints, call that out explicitly instead of silently omitting them.

## Quick Checklist

- Did you read the actual OA document content instead of only the page shell?
- Did you inspect the project first and choose the right chain instead of forcing `Provider -> Service -> Impl`?
- Did you find the closest existing integration and reuse its structure before creating new shapes?
- Did you determine whether `access_token` belongs in query or body?
- Did you keep the project-facing input model separate from OA `userid` when the project already uses its own identity model?
- Did you add constants and enum entries if the project architecture uses them?
- Did you make outer layer, service, implementation, and DTO signatures consistent?
- Did you add or update the corresponding test class and test methods for the new interface?
- Did you keep the outer layer free of HTTP details?
- Did you produce a short delivery summary that explains the chosen chain, reused template, files changed, and key document mappings?
- Did you avoid running Maven tests unless the repository rules and user instructions allow it?
