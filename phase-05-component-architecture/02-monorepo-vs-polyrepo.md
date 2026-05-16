# Monorepo Management vs. Polyrepo

## Quick Reference

| Dimension | Monorepo | Polyrepo |
|---|---|---|
| Dependency management | One `node_modules`, workspace hoisting | Each repo manages its own |
| Atomic commits | Yes — one commit can span packages | No — changes require coordinated PRs |
| Build performance | Needs caching (Turborepo/Nx) to stay fast | Per-repo CI is fast but doesn't share computation |
| Team autonomy | Shared infrastructure, own package scope | Full autonomy: own toolchain, own cadence |
| Versioning | Internal packages are always at-head | Published packages require semver coordination |

---

## What Is This?

A **monorepo** is a single version control repository that contains multiple packages or applications. A **polyrepo** is an organizational pattern where each package or application lives in its own repository.

These are organizational decisions, not tooling decisions. The tooling (Turborepo, Nx, pnpm workspaces) exists to make the monorepo pattern tractable at scale, but the core choice is about where version history lives and how teams coordinate changes across package boundaries.

A monorepo does not mean a monolith. A large monorepo can contain dozens of independently deployable packages — a shared UI library, several applications, a set of utilities — each with clean boundaries. The distinction is that they share one Git history, one CI configuration root, and one dependency graph.

> **Check yourself:** Can a monorepo contain independently deployable micro-frontends? What does the monorepo give you that separate repos do not, even when the apps are independently deployed?

---

## Why Does It Exist?

### The dependency problem at scale

Imagine three packages: `app-shell` depends on `@company/ui`, which depends on `@company/tokens`. You change a color token in `@company/tokens`. In a polyrepo:

1. Bump `@company/tokens` version, open PR, merge, publish to npm.
2. Update `@company/ui`'s `package.json` to consume the new token version, open PR, merge, publish.
3. Update `app-shell`'s `package.json` to consume the new UI version, open PR, merge, deploy.

Three separate PRs, three separate CI runs, potentially days of elapsed time for a one-line change. This is the coordination tax that monorepos eliminate.

### The diamond dependency problem

```
        app
       /   \
  pkg-a   pkg-b
       \   /
       pkg-c
```

`pkg-a` requires `pkg-c@1.x`, `pkg-b` requires `pkg-c@2.x`. In a polyrepo, `app` ends up with two copies of `pkg-c` in its dependency tree, and if `pkg-c` exports a class, `instanceof` checks fail across the two versions. In a monorepo with workspace-hoisted dependencies, the version conflict is a hard error at install time — you must resolve it — rather than a silent runtime bug.

### Why polyrepos persist

Polyrepos exist because they are the natural starting point (a new project starts as a new repo) and because they offer genuine autonomy: each team owns their CI configuration, their deploy pipeline, their toolchain version, and their release schedule entirely. There is no "merge to main" gating from another team's broken tests. This autonomy has real value — but it comes at the cost of cross-repo coordination once the packages need to talk to each other.

---

## How It Works

### Workspaces: the foundation

npm, Yarn, and pnpm all support **workspaces** — a way to declare that a repository contains multiple packages and that they should share a single `node_modules` installation with symlinked local packages.

```json
// package.json at root
{
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ]
}
```

```
my-monorepo/
  package.json          ← root workspace config
  apps/
    web/                ← apps/web/package.json { "name": "web" }
    mobile/
  packages/
    ui/                 ← packages/ui/package.json { "name": "@company/ui" }
    tokens/
    utils/
```

Running `npm install` at the root:
1. Resolves all packages' dependencies together into one unified dependency tree.
2. Hoists shared dependencies to `node_modules/` at the root.
3. Creates symlinks: `node_modules/@company/ui → packages/ui` — so when `web` imports `@company/ui`, it resolves to the local source code, not an npm-published version.

This means a change to `packages/ui/src/Button.js` is immediately visible to `apps/web` without a publish step. This is the core productivity gain of a monorepo.

> **Check yourself:** How does workspace symlinking make cross-package development faster? What would you have to do in a polyrepo to get the same "live" feedback loop?

### pnpm workspaces

pnpm is the preferred workspace tool for large monorepos because its content-addressable store means packages are never duplicated on disk, and its strict hoisting prevents packages from accidentally importing unlisted dependencies (a common npm/Yarn workspace footgun).

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

In pnpm, each package's `node_modules` only contains symlinks to packages it explicitly lists in its `package.json`. This enforces correct dependency declarations and prevents phantom dependencies — imports that work locally because they happen to be hoisted but would fail if the package were published.

---

### Turborepo: build caching and orchestration

Workspaces solve the dependency graph. They do not solve build performance. In a monorepo with 30 packages, a naive CI run that rebuilds and re-tests everything on every commit takes the same time regardless of what changed. Turborepo solves this.

Turborepo is a build system orchestrator for monorepos. It:
1. Reads the package dependency graph from workspace config.
2. Runs tasks (build, test, lint) in dependency order, with maximum parallelism.
3. Caches task outputs keyed on the input hash (source files + dependencies + env vars).
4. Skips any task whose inputs have not changed since the last cached run.

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],   // ^ means: run dependents' build first
      "outputs": [".next/**", "dist/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,            // dev servers are not cacheable
      "persistent": true
    }
  }
}
```

The `^build` syntax means "before running this package's `build`, run `build` in all packages this package depends on." This guarantees correct topological ordering without manual configuration.

### Remote caching: the primary performance argument

Local caching (`.turbo/` directory) helps individual developers, but CI runs on fresh machines and has no local cache. Turborepo's remote cache stores task outputs in a shared storage backend (Vercel's cache, or self-hosted via the open protocol). When a CI job runs:

1. Turborepo computes the hash for each task based on inputs.
2. It queries the remote cache: "Has this exact hash been built before?"
3. On a cache hit, it downloads the output artifact and replays the task as if it ran — no actual build or test execution.
4. On a cache miss, it runs the task normally and uploads the result.

**Why this is transformative:** If a developer pushes a commit that only changes `packages/ui`, only `packages/ui`'s build and test run from scratch. The `apps/web` build, which depends on `@company/ui`, gets its output from the remote cache if `apps/web`'s source and its resolved dependency hash haven't changed. In a 30-package monorepo, a targeted change might only invalidate 3 packages. CI goes from 20 minutes to 2 minutes.

The cache key includes: the package's own source files, the content of all its transitive dependencies in the workspace, the relevant env vars listed in `turbo.json`, and the task name. This makes the cache semantically correct — a hit guarantees the output is identical to what would have been produced by running the task.

---

### Nx: structured monorepo with generators

Nx is a more opinionated monorepo framework. Where Turborepo is a thin caching/orchestration layer that works on top of existing tooling, Nx provides:

- **Generators:** scaffolding commands that create new packages, components, or features following team conventions (`nx generate @nx/react:library my-lib`).
- **Executors:** wrappers around build tools that normalize configuration across the repo.
- **Module boundary enforcement:** lint rules that detect imports violating declared package boundaries (e.g., a feature library should not import from another feature library directly).
- **Affected computation:** `nx affected --target=test` uses the Git diff to compute exactly which projects are affected by the current change and runs only those.

Nx's module boundary enforcement is particularly valuable in large organizations: it makes the intended architecture enforceable rather than advisory.

---

### Internal packages: referencing and publishing

Internal packages in a monorepo are referenced by other packages in the workspace via workspace symlinks. The `package.json` `name` field is the import name:

```json
// packages/ui/package.json
{
  "name": "@company/ui",
  "version": "0.0.0",   // version is irrelevant for internal packages
  "main": "./src/index.js",
  "exports": {
    ".": "./src/index.js",
    "./button": "./src/Button/index.js"
  }
}
```

```json
// apps/web/package.json
{
  "dependencies": {
    "@company/ui": "*"   // * = use whatever version is in the workspace
  }
}
```

The `*` (or `workspace:*` in pnpm) tells the package manager to resolve this from the workspace symlink, not from npm. In TypeScript monorepos, `paths` in `tsconfig.json` maps `@company/ui` to the source directory so the IDE resolves types from source without a compile step.

**When packages need to be published to npm** (shared with external consumers), add `publishConfig`:

```json
{
  "name": "@company/ui",
  "version": "1.0.0",
  "main": "./src/index.js",
  "publishConfig": {
    "main": "./dist/index.js",
    "exports": { ".": "./dist/index.js" }
  }
}
```

`publishConfig` fields override the corresponding root fields only in the published package. During development, `main` points to `./src/index.js` (no build required). In the published tarball, it points to `./dist/index.js`.

> **Check yourself:** Why does pnpm's phantom dependency prevention matter? What bug does it catch that npm workspaces silently allow?

---

## The Polyrepo Trade-offs in Detail

Polyrepo is not a failure mode. It has genuine advantages:

**Full autonomy:** Each team owns their entire stack — Node version, Webpack vs. Vite vs. esbuild, React version, test framework, lint rules. No cross-team negotiation when upgrading. A team can adopt React 19 without waiting for three other teams to validate their code.

**Isolated CI:** A flaky test in one repo does not block deploys in another. CI queues are independent. A runaway dependency update in one service cannot break another service's pipeline.

**Blast radius containment:** A bad dependency update, security issue, or misconfiguration is scoped to one repo. Patching it does not require coordination with other teams.

**The scaling pain:** These advantages erode as packages need to interact. At a certain point — typically when more than 2–3 packages share core dependencies and change frequently together — the polyrepo overhead becomes:

- Publishing: every internal change requires semver bump + publish + downstream update PRs.
- Version drift: `app-a` is on `@company/ui@2.1`, `app-b` is on `@company/ui@1.9`. Fixing a bug in `@company/ui` must be backported or each consumer must be upgraded.
- Refactoring: an API change in `@company/utils` requires PRs across every consuming repo, often merged days apart, creating a window where the codebase is inconsistent.
- Discovery: knowing what exists and what has been deprecated requires cross-repo search. Documentation and tooling must be centralized separately.

---

## Code Owners and Atomic Commits

**CODEOWNERS** in a monorepo lets each package or directory be owned by a different team, while still allowing cross-cutting changes:

```
# .github/CODEOWNERS
/packages/ui/           @company/design-system-team
/packages/tokens/       @company/design-system-team
/apps/web/              @company/web-team
/apps/mobile/           @company/mobile-team
/packages/utils/        @company/platform-team
```

A PR that touches `packages/ui` and `apps/web` requires approval from both `design-system-team` and `web-team`. In a polyrepo, this same change would be two separate PRs that must be coordinated — and there is a window between the two merges where the system is in an inconsistent state.

**Atomic commits** are the monorepo's strongest refactoring advantage. A breaking API change to `Button` in `packages/ui` — renaming `color` prop to `variant` — can be made in a single commit that simultaneously updates every consumer in every app. Git history shows the intent: "rename Button color to variant" is one commit across 12 files in 4 packages. In a polyrepo, this is four PRs, four CI runs, four merges, and a coordination window.

---

## When Each Approach Is Right

**Choose a monorepo when:**

- Multiple packages or apps share code that changes frequently together.
- Teams want atomic refactors across package boundaries.
- You want a single source of truth for internal tooling, lint config, and CI setup.
- The team works on a coherent product with shared domain concepts.
- You have or are willing to invest in build tooling (Turborepo or Nx) to maintain CI performance.

**Choose a polyrepo when:**

- Teams have genuinely different tech stacks or upgrade cadences that should not be coupled.
- Packages have different security/compliance postures and should not share access.
- Teams are largely independent and the code rarely changes together.
- The organization is acquired or merged from separate companies with separate histories.
- A package is truly a standalone open-source library with external consumers who need a clean versioned release.

**Team size as a proxy:** Monorepos tend to pay off when a team of 15+ engineers share a significant common library surface. Below that, the operational investment in build caching and workspace configuration often exceeds the coordination savings. Above 50 engineers across multiple product areas, the atomic commit and dependency management benefits of a monorepo are usually decisive.

---

## Gotchas

**Phantom dependencies are a monorepo footgun.** With npm or Yarn workspaces, hoisting puts all packages in the root `node_modules`. Package A can `import 'lodash'` without listing it in its own `package.json` — it works because the root installed it for Package B. When Package B removes lodash, Package A breaks. The package was never a declared dependency. pnpm's isolated `node_modules` makes this a hard error at install time instead of a runtime surprise in CI.

**Remote cache invalidation is not magic.** If your task's inputs are not fully declared in `turbo.json` — for example, an environment variable that changes behavior but is not listed in `env` — Turborepo will serve a stale cache hit. Always declare env vars that affect output. `NODE_ENV`, `API_URL`, and similar variables that are baked into a build artifact must be in the cache key.

**Workspace `*` and `workspace:*` are not the same across package managers.** npm uses `"*"`, pnpm uses `"workspace:*"`, Yarn Berry uses `"workspace:^"` for different semantic guarantees. Publishing a package from a pnpm monorepo with `workspace:*` references will fail unless you run `pnpm publish` which rewrites `workspace:*` to the actual version. Never publish with raw workspace protocol references.

**CODEOWNERS only works if PRs cannot be merged without approval.** Branch protection rules must require CODEOWNERS review or the file is advisory only. In large monorepos where CI is slow, teams are tempted to merge PRs without waiting for cross-team review — enforce it at the GitHub/GitLab level, not by convention.

**Monorepos scale CI horizontally, not vertically.** Turborepo's affected computation reduces the work; it does not speed up individual task execution. If your test suite for one package takes 20 minutes, it still takes 20 minutes. Sharding (splitting tests across parallel runners) must be configured separately per package. Turborepo parallelizes across packages; you still need parallel test runners within a package.

---

## Interview Questions

**Q (High): Explain how workspace hoisting works and what problem it solves. What bug does it introduce?**
Answer: Workspace hoisting takes dependencies declared across all packages in a monorepo and resolves them into a single, unified `node_modules` at the repo root. Packages that share a dependency version get one copy on disk — installed once, symlinked to each consumer. This solves both disk space (no duplicate installations) and the diamond dependency problem (version conflicts are caught at install time rather than producing duplicate instances at runtime). The bug it introduces is phantom dependencies: because all packages end up in the root `node_modules`, any package can import any other package even without declaring it as a dependency. The import works locally but fails when the package is published or when the providing package removes that dependency. pnpm prevents this by using a strict, non-flat `node_modules` structure where each package's `node_modules` only contains its declared dependencies.
The trap: Knowing hoisting but not knowing phantom dependencies. Hoisting is a well-known concept; phantom dependencies are the subtlety that separates experienced monorepo practitioners.

**Q (High): How does Turborepo's remote cache work, and what must be true for a cache hit to be correct?**
Answer: Turborepo computes a hash for each task based on: the source files in that package, the content of all transitive workspace dependencies, the task configuration, and any env vars declared as inputs in `turbo.json`. Before running a task, it queries the remote cache with that hash. On a hit, it downloads the cached output artifacts and restores them, skipping execution entirely. On a miss, it runs the task and uploads the result. For the cache to be correct, the hash must cover every input that affects the output. The common failure modes: an env var that is baked into the build output but not listed in `env` (the cache serves a build with the wrong API URL), generated files that are inputs but not tracked by Turborepo's file hashing, or external side effects like network calls during the build (caching a build that fetched live data, replaying that result when the data has changed). The rule is: if omitting it would produce a different output, it must be in the cache key.
The trap: Thinking remote caching is automatic and infallible. Cache correctness depends entirely on correctly declaring all inputs.

**Q (High): What is the diamond dependency problem, and how does a monorepo solve it differently from a polyrepo?**
Answer: The diamond dependency problem occurs when two packages depend on different versions of a shared package. In Node.js, this typically results in two copies of the module in the dependency tree. If the module exports a class or uses module-level state, cross-instance operations fail: an `instanceof` check across versions returns false, or singleton state is duplicated. In a polyrepo, this conflict is invisible at development time — `app` depends on `pkg-a` and `pkg-b`, which each bring their own copy of `pkg-c`. The conflict only surfaces as a runtime bug. In a monorepo with workspaces, all packages' dependencies are resolved together at install time. If `pkg-a` requires `pkg-c@1.x` and `pkg-b` requires `pkg-c@2.x`, the install fails with a clear version conflict error. You are forced to resolve it upfront: align versions, or use a peer dependency and let the consumer decide. This makes dependency contracts explicit rather than accidental.
The trap: Describing the diamond problem without connecting it to why monorepos expose it as a build-time error rather than a runtime bug.

**Q (Medium): What is `publishConfig` in a monorepo package and why is it useful?**
Answer: `publishConfig` is a `package.json` field whose key-value pairs override the corresponding root-level fields in the published npm tarball only. During development in a monorepo, you want `"main": "./src/index.js"` so that workspace consumers import directly from source — no build step required, instant feedback. But if you publish this package to npm, external consumers need `"main": "./dist/index.js"` pointing to the compiled output. With `publishConfig: { "main": "./dist/index.js", "exports": { ".": "./dist/index.js" } }`, the package's source-pointing fields stay in place for local development, and npm (or pnpm publish) substitutes the `publishConfig` fields in the tarball. Without this, you would either need to run a build and mutate `package.json` before publishing (fragile, easy to forget to revert) or maintain separate fields manually.
The trap: Not knowing `publishConfig` exists, or thinking you must change `package.json` before publishing.

**Q (Medium): When would you choose a polyrepo over a monorepo for a team shipping multiple frontend applications?**
Answer: Polyrepo is the right choice when the applications have genuinely different technology stacks or upgrade cadences that should not be coupled — for example, a greenfield React 19 app and a legacy jQuery app that the organization is maintaining in parallel. It is also right when teams have different security postures (an internal admin tool and a public-facing consumer app that should not share repository access), when packages are truly standalone with external consumers (an open-source library with its own community and release schedule), or when the organization is composed of teams that were previously separate companies with separate infrastructure. The key signal is whether the code changes together: if Package A never needs to change when Package B changes, the monorepo's cross-package tooling provides no benefit and only adds infrastructure overhead. The mistake is defaulting to monorepo as "best practice" without asking whether the packages actually have the shared change patterns that monorepos are built to serve.
The trap: Treating monorepo as universally superior. Strong candidates can articulate conditions under which polyrepo is the correct choice.

**Q (Medium): How do CODEOWNERS and branch protection rules work together in a monorepo to preserve team autonomy?**
Answer: A `CODEOWNERS` file maps directory paths to GitHub teams or individual users. Any PR that modifies files in a listed path automatically requires a review approval from the owning team before merge. In a monorepo, this means a PR that touches `packages/ui` (owned by the design system team) and `apps/web` (owned by the web team) requires both teams to approve. Combined with branch protection rules that enforce "required reviewers" and "require CODEOWNERS review," this makes the rule machine-enforced rather than convention-based. The limitation: CODEOWNERS is advisory unless branch protection is configured to block merges without approval. In large monorepos with slow CI, teams feel pressure to bypass this — which is why the protection must be at the repository settings level, not the honor system. The benefit is that teams retain meaningful veto power over changes to their owned packages even when a cross-cutting refactor touches their code.
The trap: Thinking CODEOWNERS alone prevents unauthorized merges. It must be paired with branch protection rules.

**Q (Low): What is the difference between Turborepo and Nx in terms of what each provides?**
Answer: Turborepo is a thin caching and task orchestration layer. It reads your existing workspace configuration, understands the dependency graph, and adds content-hash-based caching with remote cache support and parallel task execution. It does not impose opinions about how packages are structured or what tools they use — it works on top of whatever build tooling each package already has. Nx is a more opinionated framework that additionally provides generators (code scaffolding for new packages, components, and features following enforced conventions), executors (normalized wrappers around build tools so all packages use consistent configuration), and architectural lint rules that enforce declared module boundaries — for example, preventing a `feature` library from importing directly from another `feature` library, enforcing that only `app` packages may do that. Nx's stronger opinions make it better suited for large organizations that want to enforce consistent patterns; Turborepo's lighter touch makes it easier to adopt incrementally without restructuring existing code.
The trap: Saying Turborepo and Nx are interchangeable or that Nx is just "Turborepo with more features." The distinction is philosophical: orchestration layer vs. structured framework.

---

## Self-Assessment

Before moving on, check off each item you can do WITHOUT looking at the file.

- [ ] Explain how workspace hoisting works, what phantom dependencies are, and why pnpm's stricter isolation prevents them
- [ ] Walk through the coordination tax that drives monorepo adoption using the diamond dependency problem as a concrete example
- [ ] Describe how Turborepo's remote cache key is computed and name two categories of inputs that, if missing, produce incorrect cache hits
- [ ] Explain what `publishConfig` does and why it is useful for packages that are used both as workspace internals and as published npm packages
- [ ] State the conditions under which a polyrepo is the correct choice, not just a legacy pattern to be migrated away from
- [ ] Explain how CODEOWNERS and branch protection combine to preserve per-team autonomy within a monorepo

---
*Next: Design System Architecture — once you decide on a monorepo or polyrepo, the shared UI component library lives at the heart of it; design systems are how teams scale UI consistency.*
