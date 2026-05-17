# Environment Config Management & Package Versioning at Scale

## Quick Reference

| Topic | Key decision | Rule |
|---|---|---|
| Env vars | `.env` vs. build-time injection vs. runtime config | Never put secrets in client bundles |
| Semver | `^` vs. `~` vs. exact | `^` for apps, exact for CI reproducibility |
| Lockfiles | Commit or not? | Always commit for apps, gitignore for libraries |
| Dependency updates | Ad-hoc vs. automated | Automated (Renovate/Dependabot) with CI gates |
| Monorepo versions | Single version vs. independent | Single version for cohesive systems |

---

## What Is This?

Two related concerns that bite teams at scale:

**Environment config management** — how values that differ across environments (local, staging, production) get into an app. API URLs, feature flag values, analytics keys. The challenge is keeping secrets out of the client bundle while making non-secret config available.

**Package versioning** — how npm packages are versioned and pinned so that builds are reproducible, upgrades are controlled, and security vulnerabilities are caught. At scale, unmanaged dependencies cause "it works on my machine" failures, security incidents, and upgrade fatigue.

> **Check yourself:** Why can't you put a secret API key in a `.env` file for a Vite app and call it secure?

---

## Why It Matters

Environment config mistakes leak secrets into bundles (available to anyone in DevTools). Package version mismatches cause production builds to differ from development builds. At team scale, undisciplined dependency management means the codebase silently drifts across developer machines, CI, and production.

---

## Environment Variables

### The fundamental rule: nothing secret in the client bundle

```
Client bundle: sent to the browser, readable by anyone in DevTools → Network tab
               → minification is not obfuscation

Backend: private, not sent to clients, can hold secrets
```

A Vite or webpack app bundle is a plain text file (even when minified) served to every visitor. Any value baked into it — API keys, secret tokens, private URLs — is publicly visible.

### .env files

Standard convention across Vite, Create React App, Next.js:

```bash
# .env               — loaded in all environments
# .env.local         — local overrides, never committed (add to .gitignore)
# .env.development   — only in dev (npm run dev)
# .env.production    — only in prod build (npm run build)
# .env.test          — only during tests

# Vite — prefix VITE_ to expose to client code
VITE_API_URL=https://api.example.com
VITE_ANALYTICS_ID=UA-12345

# Next.js — prefix NEXT_PUBLIC_ to expose to client
NEXT_PUBLIC_API_URL=https://api.example.com

# Without the prefix: server-side only (Next.js API routes / getServerSideProps)
DATABASE_URL=postgresql://...
SECRET_KEY=my-secret-key
```

### How Vite injects env vars

At build time, Vite replaces references to `import.meta.env.VITE_API_URL` with the string value:

```js
// Source:
const url = import.meta.env.VITE_API_URL;

// Bundle output:
const url = "https://api.example.com";
```

This is **static, build-time substitution** — the value is baked into the bundle at build time. Changing the env var requires a rebuild.

```js
// Vite — accessing env vars
const apiUrl = import.meta.env.VITE_API_URL;
const isProd = import.meta.env.PROD;  // boolean
const isDev = import.meta.env.DEV;    // boolean
const mode = import.meta.env.MODE;    // 'development' | 'production' | custom

// webpack — using DefinePlugin
new webpack.DefinePlugin({
  'process.env.API_URL': JSON.stringify(process.env.API_URL),
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
});
```

### Build-time vs. runtime config

**Build-time (baked in at build):**
- Values change with every build
- Requires a full rebuild to change any config value
- Zero overhead at runtime
- Cannot change without redeployment

**Runtime config (fetched after app loads):**
- Config served as a JSON endpoint or a separate small JS file
- App fetches config on startup before initialization
- Enables config changes without redeployment
- Adds a startup fetch (can block initial render)

```js
// Runtime config pattern
// public/config.js — served but not bundled
window.__APP_CONFIG__ = {
  apiUrl: 'https://api.example.com',
  featureFlags: { newDashboard: true },
};

// App entry — read at runtime
const config = window.__APP_CONFIG__ || {};
const API_URL = config.apiUrl;
```

This pattern lets ops teams change config values (not secrets) without triggering a frontend redeploy. Useful for environments where rebuild cycles are slow.

### Secrets: the right place

Secrets (API keys for external services, database URLs, JWT signing keys) belong on the server:

```
Browser → your API server → external service (with secret key)
                ↑
         Secret lives here, in server env vars
         Never reaches the browser
```

If your frontend directly calls an external API with a key (OpenAI, Stripe, etc.), that key is in the bundle. Route those calls through your backend, which holds the secret.

---

## Semantic Versioning (Semver)

npm packages use `MAJOR.MINOR.PATCH`:

```
1.4.2
│ │ └─ PATCH — bug fixes, no API changes
│ └─── MINOR — new features, backward compatible
└───── MAJOR — breaking changes
```

### Version range operators in `package.json`

```json
{
  "dependencies": {
    "react": "^18.2.0",     // ^ — compatible with 18.2.0, accepts 18.x.x but not 19.0.0
    "lodash": "~4.17.21",   // ~ — accepts patch updates only: 4.17.x but not 4.18.0
    "vite": "5.4.0",         // exact — only exactly 5.4.0
    "jest": ">=29.0.0",      // >= — anything 29 or higher
    "typescript": "*"        // * — any version (avoid)
  }
}
```

### `^` (caret) — the default

`^` accepts any version compatible with the same major. When you run `npm install`, npm installs the latest version within the caret range. This is why two developers running `npm install` a month apart may get different patch versions.

### Why exact versions exist

Some teams use exact versions in `package.json` for maximum reproducibility:

```json
{
  "dependencies": {
    "react": "18.2.0"   // only ever exactly this
  }
}
```

Combined with a lockfile, this is redundant — the lockfile already pins exact versions. Exact versions in `package.json` prevent the lockfile from ever upgrading without an explicit change to `package.json`. This provides maximum control but requires more manual upgrade work.

For **libraries**, `package.json` should use `^` ranges for `peerDependencies` to be compatible with consumer versions, but exactness matters less since libraries don't have lockfiles in the package itself.

---

## Lockfiles

Lockfiles record the exact version of every installed package (and all transitive dependencies) at install time.

| File | Package manager |
|---|---|
| `package-lock.json` | npm |
| `yarn.lock` | Yarn |
| `pnpm-lock.yaml` | pnpm |
| `bun.lockb` | Bun |

### Always commit lockfiles for apps

```bash
# DO: commit the lockfile
git add package-lock.json

# DON'T: add it to .gitignore
```

Without a lockfile, every `npm install` may resolve different transitive dependency versions — a new security vulnerability in a transitive dep could silently change your production build. "It works on my machine" is caused by lockfile drift.

### Don't commit lockfiles for libraries

When publishing a library to npm, the lockfile is ignored by consumers — npm uses the consumer's `package.json` ranges. Including a lockfile in a published library adds noise. Libraries should `.gitignore` their lockfile (or rather, the published package excludes it via `.npmignore`/`package.json` `files`).

### `npm ci` vs. `npm install`

```bash
# npm install — reads package.json, may update lockfile
npm install

# npm ci — reads lockfile ONLY, fails if lockfile is missing or inconsistent
npm ci   # for CI/CD — guarantees reproducibility
```

Always use `npm ci` (or `yarn install --frozen-lockfile` / `pnpm install --frozen-lockfile`) in CI pipelines. `npm install` in CI is a sign of lockfile discipline problems.

---

## Dependency Update Strategies

### Automated updates (Renovate / Dependabot)

Renovate (or GitHub Dependabot) opens PRs to update dependencies automatically:

```json
// renovate.json
{
  "extends": ["config:base"],
  "schedule": ["every weekend"],
  "grouping": [
    {
      "matchPackagePatterns": ["^eslint"],
      "groupName": "ESLint packages"
    }
  ],
  "automerge": true,
  "automergeType": "pr",
  "labels": ["dependencies"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "automerge": true    // auto-merge security fixes
  }
}
```

Without automation, dependencies stagnate. Teams that update quarterly or annually face large, risky upgrades. Regular automated updates — validated by CI — mean each update is small and safe.

### Security audits

```bash
# Check for known vulnerabilities
npm audit

# Automatically fix non-breaking vulnerabilities
npm audit fix

# See what a fix would change without applying
npm audit fix --dry-run

# Force fix (may introduce breaking changes)
npm audit fix --force
```

Integrate `npm audit --audit-level=high` into CI to block deployments with high-severity vulnerabilities.

---

## Monorepo Package Versioning

In a monorepo (Nx, Turborepo), packages can be versioned independently or together.

### Fixed/synchronized versioning

All packages share one version number. When anything changes, everything bumps to the next version. Used by large projects where all packages release together (Angular, RxJS).

```
@my-company/ui:        1.5.0
@my-company/utils:     1.5.0  ← same version always
@my-company/hooks:     1.5.0
```

### Independent versioning

Each package versions independently. Used when packages have very different release cadences (a UI component vs. an analytics utility).

```
@my-company/ui:        2.1.4
@my-company/utils:     0.8.1  ← independently versioned
@my-company/hooks:     1.0.0
```

Tooling (Changesets, Lerna) manages the changelog and version bump process:

```bash
# Changesets workflow:
# 1. Developer adds a changeset when making a change
npx changeset add
# 2. CI creates a "Version Packages" PR that bumps affected packages
# 3. Merging releases all changed packages
```

---

## Package Manager Choices

| Package manager | Key differentiator |
|---|---|
| npm | Default, ships with Node.js |
| Yarn Classic (v1) | Deterministic lockfile, workspaces |
| pnpm | Symlinked, content-addressable store — fastest installs, disk-efficient |
| Bun | JS runtime + package manager, fastest installs overall |

For large monorepos, **pnpm** is increasingly the default — its content-addressable store means packages are stored once on disk and symlinked, rather than copied into every `node_modules`. This dramatically reduces disk usage and install time for workspaces.

```bash
# pnpm workspace setup
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'

# Install all workspace packages
pnpm install

# Run script in specific workspace
pnpm --filter @my/ui build

# Add dep to specific workspace
pnpm --filter @my/app add react
```

---

## Gotchas

**1. `VITE_` prefix doesn't make a var "safe" — it makes it public**
A common misunderstanding: `VITE_SOME_KEY=mysecret` exposes `mysecret` in the bundle. The prefix is required to make the var accessible in client code — it's not a security mechanism. Never use the prefix for secrets.

**2. `npm install` in CI updates the lockfile silently**
If `package.json` uses `^` ranges and a new patch is released between your last manual install and the CI run, `npm install` installs the newer version and updates the lockfile — without you noticing. Use `npm ci` which fails if the lockfile is out of sync with `package.json`, making inconsistencies visible.

**3. Lockfile merge conflicts**
When two branches both update dependencies, the lockfile merge conflict is usually safe to resolve by running `npm install` after accepting either version of `package.json`. Manually resolving lockfile conflicts is error-prone. Most teams use `npm install` to regenerate the lockfile post-merge.

**4. Peer dependency warnings are not errors (but matter)**
When you see "peer dependency not satisfied" warnings, npm still installs. But mismatched peer dependencies cause subtle bugs — a component library expecting React 17 running in a React 18 app may have type errors, unexpected behavior in concurrent mode, or hook rule violations. Take peer dependency warnings seriously.

**5. Private packages and `.env` in CI**
CI environments need environment variables too. Store them in your CI system's secrets (GitHub Actions secrets, Doppler, AWS Secrets Manager) — not in committed `.env` files. The `.env.local` convention exists specifically so local overrides aren't committed.

---

## Interview Questions

**Q (High): Why can't you put a private API key in a `.env` file for a Vite application?**

Answer: Vite performs build-time substitution — at `vite build`, every reference to `import.meta.env.VITE_*` is replaced with the string value. The resulting bundle contains the literal string in plain text. The bundle is then served to every visitor. Even though the file is "minified," there is no obfuscation — anyone who opens the Network tab in DevTools and reads the bundle source can extract any value baked into it. The `.env` file itself is not shipped, but its contents become part of the bundle. For private API keys, the correct approach is to call your own backend server, which holds the key as a server-side environment variable (not in any client-deployed artifact) and proxies the request to the external service.

**Q (High): What is the difference between `npm install` and `npm ci`, and when should you use each?**

Answer: `npm install` reads `package.json`, resolves the ranges, and installs the latest compatible versions. If it finds an existing `package-lock.json`, it tries to use it but may update it when ranges have newer compatible versions available. `npm ci` (CI stands for clean install) reads only `package-lock.json` — it installs exactly what's recorded there and fails if `package-lock.json` is absent or if it's inconsistent with `package.json`. `npm ci` also always starts by deleting `node_modules`. Use `npm install` in local development when you want to update deps. Use `npm ci` in CI/CD pipelines — it guarantees that every CI run installs exactly the same packages, making builds reproducible and making lockfile drift visible as a build failure rather than a silent behavior difference.

**Q (High): When should you commit a lockfile and when should you not?**

Answer: Commit lockfiles for applications — anything you deploy. The lockfile ensures that everyone on the team, CI, and production all install the identical dependency tree. Without it, a transitive dependency that released a patch between two `npm install` runs could cause "works on my machine" behavior. Don't commit lockfiles for libraries you publish to npm. The lockfile of a library is not used by consumers — npm ignores it and resolves the consumer's own tree from the library's `package.json` ranges. Including a lockfile in a library repository is fine for development consistency, but it should be excluded from the published package (via `package.json` `files` field or `.npmignore`).

**Q (Medium): What is the dual-concern with build-time vs. runtime config injection, and when would you choose runtime?**

Answer: Build-time injection (Vite's `import.meta.env`, webpack's DefinePlugin) bakes config values into the bundle at build time. The bundle must be rebuilt and redeployed to change any value — there's no way to update config without redeployment. This is fine for most config, but becomes a problem when you need to change a non-secret config value (like a feature flag or environment-specific URL) without a full frontend deploy cycle. Runtime config solves this: a separate `config.json` file or an API endpoint serves config that the app fetches on startup. Changing config requires only updating the config file, not rebuilding the JS bundle. The tradeoff: a startup fetch adds latency and a potential failure point. Runtime config is appropriate for: ops-managed config, multi-tenant apps where config differs per customer, and feature flags that change frequently.

**Q (Medium): What is Renovate and why is automated dependency management preferable to manual updates?**

Answer: Renovate is a tool that watches your `package.json`, detects when new versions of dependencies are released, and automatically opens PRs to update them. Each PR targets one dependency (or a configured group), runs your CI test suite, and can be configured to auto-merge if tests pass. The alternative — manual updates — leads to dependency stagnation: teams defer updates until they're forced (security vulnerability, breaking feature requirement), at which point they face a large, risky upgrade spanning months of API changes. Automated updates keep dependencies current in small, low-risk increments. Security patches get applied promptly. Each update is a contained, reviewed, CI-validated change. Teams using Renovate tend to be running near-latest versions of all dependencies, making major framework upgrades incremental rather than disruptive.

---

## Self-Assessment

- [ ] Explain why `VITE_MY_SECRET=foo` does not keep a secret secure
- [ ] Describe the build-time vs. runtime config tradeoff and name a use case for each
- [ ] Explain the difference between `npm install` and `npm ci` and where each belongs
- [ ] Describe when to commit a lockfile vs. when to exclude it
- [ ] Explain what Renovate does and why automated updates reduce risk compared to quarterly manual updates

---
*Phase 10 complete. Next: Phase 11 — Web Components & Native Browser Primitives, starting with the Custom Elements lifecycle.*
