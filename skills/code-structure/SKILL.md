---
name: code-structure
description: Use when adding a file, module, component, server action, zod schema or package to a TypeScript monorepo with Next.js, when moving, renaming or splitting modules, when unsure where code belongs or what a file should be called, when asked whether something needs an interface, port, adapter or wrapper, or when asked to review or audit a repository's structure, naming or duplication.
---

# Skill: code-structure

This skill defines where code goes and what it is called in a TypeScript monorepo with Next.js: layers, file paths, naming, schemas, abstractions, and how to audit a repository against them.

## Overview

Every file has one role and one domain. The role decides the file name, the domain decides the folder. Anyone who knows the two can name the path without searching.

A project `AGENTS.md` or `CLAUDE.md` overrides this skill: package names, the list of domains, test suffix, import alias, linter, exceptions. Read it first. Paths and tool names below are the defaults.

Existing files that break these rules are not precedent. New code takes the form described here, also next to old files that do not, and the old file is reported as a finding instead of copied.

## Layers

Imports point one way only.

| Layer                           | Default location        | May import                       |
| ------------------------------- | ----------------------- | -------------------------------- |
| Domain: zod schemas, pure logic | `packages/core`         | nothing at runtime               |
| Repositories                    | `packages/db`           | domain                           |
| Other packages                  | `packages/<name>`       | domain, and packages below them  |
| App library                     | `apps/<app>/lib`        | packages                         |
| Hooks                           | `apps/<app>/hooks`      | `lib`                            |
| Components                      | `apps/<app>/components` | `lib`, `hooks`, other components |
| Routing                         | `apps/<app>/app`        | everything above                 |

- The domain package may name the type of a repository or provider with `import type`, as long as the package graph stays free of cycles. Where that would close a cycle, the domain package declares an interface instead.
- A repository without a domain or repository package keeps both roles in `lib/<domain>/`.
- A package is reached through the entries in its `exports` field. No path alias or relative import into another package's sources.
- Code that runs in the browser imports no file marked `server-only` and no file that reads secrets.

## Where a file goes

`<domain>` is a singular noun from the project's list of domains, such as `run` or `source`. A new domain is added to that list in the same change.

| Role                                           | Path                                                           | Marker                                |
| ---------------------------------------------- | -------------------------------------------------------------- | ------------------------------------- |
| Page, layout, route handler                    | `app/**/page.tsx`, `layout.tsx`, `route.ts`                    | thin: access check, one call, render  |
| Server action                                  | `lib/<domain>/actions.ts`                                      | `"use server"`                        |
| Read used by several pages, or combining calls | `lib/<domain>/queries.ts`                                      | `server-only`                         |
| Use case with several steps                    | `lib/<domain>/<verb>-<noun>.ts`                                | `server-only`, dependencies passed in |
| Form values and their parsing                  | `lib/<domain>/fields.ts`                                       |                                       |
| Provider: client for an external system        | `lib/<domain>/<capability>.ts`, such as `mail.ts`              | `server-only`                         |
| Schema that belongs to one provider            | next to it, `<name>-schema.ts`, such as `mail-schema.ts`       |                                       |
| Schema of a payload one route sends            | `lib/<domain>/<payload>-schema.ts`                             | safe in the browser                   |
| Environment values read by several files       | `lib/<domain>/config.ts`                                       | `server-only`                         |
| Formatting and view models                     | `lib/<domain>/format.ts`, `view.ts`                            | safe in the browser                   |
| Component                                      | `components/<domain>/<thing>.tsx`                              | `"use client"` only when needed       |
| Hook                                           | `hooks/use-<thing>.ts`                                         | `"use client"`                        |
| Domain model                                   | `packages/core/src/specs/<name>-schema.ts`                     | zod and types only                    |
| Domain logic                                   | `packages/core/src/<topic>.ts`, such as `run-status.ts`        | pure functions                        |
| Repository                                     | `packages/db/src/repositories/<name>-repository.ts`            | one class per entity                  |
| Test                                           | next to the module, `<module>.test.ts` or the project's suffix |                                       |

- A domain folder holds only that domain. Code used by several domains stays flat one level up, in a file named after its content: `lib/date-format.ts`, `components/button-link.tsx`.
- Inside `components/<domain>/` the file drops the domain and the export keeps it: `components/run/sheet.tsx` exports `RunSheet`.
- A page calls a repository method directly, including a single check such as `?? notFound()`. The read moves to `queries.ts` with its second caller.
- A use case lives in the app that needs it. When a second app needs it, it moves to a package above the repository package.
- Logic that a page or route would hold beyond access check, one call and render belongs in `lib/<domain>/`.
- A file that reads environment values for itself alone reads them inline.
- Where an old file already holds a role at a non-conforming path, add to it and report it. One role is never split across an old and a new file.
- `specs/` and `repositories/` each have an `index.ts` that forwards their files. The package root forwards those folders.
- A file whose role is not in the table sits in its domain folder under a name that says what it holds: `lib/run/events.ts`.
- Labels live in components, not in `lib/` or a package. Formatters and the messages returned by actions and schemas are the exception.

## Naming

- File and folder names are kebab-case.
- One concept per file, and the name says which. A file that needs "and" to describe it is two files.
- A provider sits in the domain folder of its only caller, and flat in `lib/` once several domains call it.
- A provider file is named after what it does, not after the vendor: `mail.ts`, not `brevo.ts`. Switching vendors keeps the path.
- These names are never used: `utils.ts`, `helpers.ts`, `common.ts`, `misc.ts`, `shared.ts`, and `types.ts` outside a folder whose siblings share those types.
- One word per thing across all layers. If the repository says `request`, the action, the use case and the component do not say `start`, `launch` and `trigger` for the same step.
- A function that writes says so in its name. No write hides behind `get` or `find`.
- No filler words: `Data`, `Info`, `Manager`, `Handler`. `Schema` and `Repository` are the only suffixes that name a kind.
- A name is not reused for a second thing in the same scope: no variable named like an import, no parameter named like a schema.
- An `index.ts` only forwards: `export * from "./module"`. The module decides through `export` what leaves it.

## Schemas

All domain models are zod schemas in the `specs/` folder of the domain package. A schema that only one provider or route uses stays next to it. Types are inferred, never written by hand next to a schema.

```ts
import { z } from "zod";

export type ProfileInput = z.infer<typeof ProfileInputSchema>;
export const ProfileInputSchema = z.object({
  name: z.string().trim().min(1),
  tags: z.array(z.string()),
});

export type Profile = z.infer<typeof ProfileSchema>;
export const ProfileSchema = ProfileInputSchema.extend({
  id: z.string(),
  createdAt: z.date(),
});
```

- The const is PascalCase with the suffix `Schema`. The type has the same name without the suffix.
- Use `z.input` instead of `z.infer` only when the schema transforms its input and the type describes what goes in.
- The type comes first, the const directly below it, no blank line between them, one blank line above and below the pair.
- Leave the type out when nothing uses it.
- A `-schema.ts` file holds schemas and types only. A schema may call a pure function from another file. Logic never lives in a schema file.
- Props, form values and return types are derived from the schema type (`Pick`, `Omit`, indexed access). They never repeat its fields.

## Repositories and use cases

A repository is a class with one method per query. A use case receives its repositories and providers as a parameter, typed by the concrete class, and returns the function that does the work. Server actions, route handlers and pages do the wiring.

```ts
// packages/db/src/repositories/run-repository.ts
export class RunRepository {
  constructor(private readonly session?: ClientSession) {}

  async find(id: Run["id"]) {}
}

// apps/dashboard/lib/run/request-run.ts
export function requestRun({ runRepo }: { runRepo: RunRepository }) {
  return async (step: StepName) => {};
}

// apps/dashboard/lib/run/actions.ts
await requestRun({ runRepo: new RunRepository() })(step);
```

- The class is the type. No interface stands beside a repository or provider that has one implementation.
- With the second real implementation, the type becomes an interface in the domain package and the use cases stay unchanged. Test doubles do not count as an implementation.
- Only use cases take dependencies as a parameter. Actions, pages and components import what they call.
- A use case holds a rule or several steps. One that only forwards to a repository method is deleted, and the caller uses the repository.

## Abstractions

- An interface or a factory exists only when the code has two real implementations today. Test doubles do not count. The one exception is the interface the domain package declares to avoid a package cycle.
- A provider is plain functions or one class, without an interface. Replacing the vendor means rewriting that file.
- A function is extracted when it has two callers or hides a detail of a lower level. Giving a block a name is not a reason.
- A function that only renames or forwards a call, and has one caller, is inlined. The test: if deleting it removes code instead of moving it to the caller, it was a pass-through.
- No parameter, config field, type or export for a case that does not exist yet.
- Use what the library ships (error codes, constants, types) before writing your own.

## Reuse

- A rule, a schema or a constant exists once. Two identical copies of it become one function or component.
- Code that is only similar stays apart. With the third similar block, extract what all three share. A shared helper that grows a branch per caller is inlined back.
- Converting between two shapes of the same data, such as flat form values and a nested schema, is one generic helper in one place. No set of helpers per schema.
- Before writing a helper, search for it. A third formatter next to two existing ones is a finding, not a feature.
- Reusing a function from the wrong place includes moving it to the right one and updating its importers. Any other restructuring is a separate PR.
- An export nothing imports loses its `export` or is deleted. An export that only its own test imports stays.
- A value that never varies is written inside the component, not passed as a prop.
- A component is split in two only when the parts are used separately or one of them must run on the server.

## Style

- Every `if`, `for` and `while` has braces.
- Guard clauses with an early return replace `else` and nested conditions.
- A multi-line block is separated from the statements around it by one blank line. No blank line directly after an opening brace or before a closing one.
- No nested ternary. In JSX, move the choice into a component or a lookup object.
- A function that returns JSX is a component: `<ConnectionAlert events={events} />`, not `{connectionAlert(events)}` or an inner `content()`. It takes what it reads as props.
- In a file, constants and lookup objects come first, then the exported function, then the helpers it calls.
- A function takes at most three parameters, then an options object. Signatures a framework dictates are exempt.
- A function takes no boolean parameter that selects its path. Write two functions: `formatDate` and `formatDateTime`.
- A boolean prop describes state: `disabled`, `open`, `live`. A choice between two layouts or behaviours is a `variant` union, a slot (`children`, `render`) or a second component, not `asLink` or `isEdit`. Two booleans that never hold together become one union.
- Props are typed inline in the signature. A named type exists only when two functions share it or another file imports it.
- A type lives next to the code that produces its values. It moves to `types.ts` only when several files of one folder share it.
- Files in the same folder import each other relatively (`./fields`). Every other import goes through the alias (`@/lib/run/format`).
- Comments state only what the code cannot show and what comes from another system. Design reasoning goes into the commit or an ADR.

## Enforcement

A rule a tool can check is configured in that tool and fixed across the whole repository in one `refactor:` PR. The review covers only what is left.

Biome is the recommended linter and formatter: one tool and one config file cover every linter rule below. It is a recommendation, not a requirement. A project that already has a linter, or names one in its project file, keeps it. Where none is configured, propose Biome and set it up once the user agrees.

| Rule                                                          | Checked by                             | With Biome                                                               |
| ------------------------------------------------------------- | -------------------------------------- | ------------------------------------------------------------------------ |
| Braces, no nested ternary, no useless `else`, parameter count | linter                                 | `useBlockStatements`, `noNestedTernary`, `noUselessElse`, `useMaxParams` |
| kebab-case file names                                         | linter                                 | `useFilenamingConvention`                                                |
| Import direction between `lib`, `hooks`, `components`, `app`  | linter                                 | `noRestrictedImports` in an `overrides` entry per folder                 |
| No import cycles                                              | linter                                 | `noImportCycles`                                                         |
| Unused exports, files and dependencies                        | knip                                   |                                                                          |
| Package boundaries                                            | `exports` field of each `package.json` |                                                                          |
| Everything else in this skill                                 | review                                 |                                                                          |

With another linter, map each rule to its equivalent. A rule without an equivalent stays with the review. Any rule that forbids barrel files or `export *` contradicts the `index.ts` rule and stays off: in Biome, `noBarrelFile` and `noReExportAll`. Check each rule name and its options against the installed version before writing it down.

## Placing a new module

1. Read the project file for package names, domains and exceptions.
2. Name the role and the domain of the code.
3. Look up the path in "Where a file goes". If the file exists, add to it.
4. If the code has two roles, write two files.
5. Search for an existing function before adding a helper.
6. Run the project's checks.

## Auditing a repository

The audit produces a list of findings. It changes no code.

1. List every file under `lib`, `components`, `hooks` and the packages with its role and domain. A file with two roles or no clear domain is a finding.
2. Compare each path and name with "Where a file goes" and "Naming".
3. List every zod schema with its name, the order of type and const, and its package.
4. Run knip, or grep the importers of each export. List exports without importers.
5. Count the real implementations behind each interface, function-typed parameter and factory.
6. List duplicated blocks and one-caller wrappers.
7. Check the import direction and which browser code reaches server files.
8. Report each finding as `file:line`, the rule it breaks, and one proposed change. A rule broken in many places is one finding with a count. Sort: secrets and wrong import direction first, then by the number of files the change touches. Group the findings into refactorings that can each be merged alone.

## Red flags

| Thought                                                 | Reality                                                                |
| ------------------------------------------------------- | ---------------------------------------------------------------------- |
| "The neighbouring file does it this way"                | Old files are not precedent. Follow the table, report the neighbour.   |
| "We will switch providers later, so I add an interface" | One implementation, one file. The interface comes with the second one. |
| "The use case can import the repository itself"         | Pass it in. The signature lists what the use case touches.             |
| "Every repository call should go through a use case"    | A use case that only forwards is deleted.                              |
| "It's a small helper, `utils.ts` is fine"               | Name the file after what the helper does.                              |
| "This wrapper makes the call site read nicer"           | One caller and no logic of its own: inline it.                         |
| "I'll add the option now, we will need it"              | Build it when the second case exists.                                  |
| "The types are almost the same, I'll write a new one"   | Derive it from the schema type.                                        |
| "I'm already in here, I'll fix the structure too"       | An audit lists findings. Refactorings are their own PRs.               |
| "The file fits two domains"                             | Then it is generic and stays flat, or it is two files.                 |
