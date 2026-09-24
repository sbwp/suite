# Apps
A suite of personal web apps that share one platform: the same AWS accounts, Mongo Atlas cluster, sign-in, build tooling, and deployment pipeline. Each app is a distinct application with its own UI and services. Apps may call each other's APIs, but they never share UIs or read each other's data directly.

**Planned apps, in build order:**
1. **Budget:** a budgeting tool, used only by me.
2. **Events:** scheduling for events I host with friends. Invited friends sign in to see events and RSVP. It's the most representative app, since it has multiple users with different roles.
3. **Admin:** a small page for managing who can access which apps (inviting users, adding and removing groups). Until it exists, an invite script does this job (see [Granting access](#granting-access)).

**Guiding constraints:**
- **Audience:** most apps are just for me, and some are for a small group of invited friends. Public-facing apps aren't ruled out, but they might be built outside this suite.
- **Cost:** $0 wherever a free tier allows, until something has a public audience. Anything that costs money is called out explicitly. Current fixed cost: about $1/month for DNS.
- **Everything is public:** every repo is public on GitHub (the account is on GitHub Pro), so nothing sensitive ever goes in a repo, and CI is locked down accordingly (see [Public repo safeguards](#public-repo-safeguards)).
- **Strong typing and thorough testing:** types come from one source wherever possible, so mismatches fail at compile time. Unit tests have 100% coverage plus RACC (see [Tests](#tests)).

## Contents
1. [Repositories](#1-repositories)
2. [Architecture overview](#2-architecture-overview)
3. [Backend](#3-backend)
4. [Frontend](#4-frontend)
5. [Authentication](#5-authentication)
6. [Infrastructure](#6-infrastructure)
7. [CI/CD and releases](#7-cicd-and-releases)
8. [Local development](#8-local-development)
9. [Standards](#9-standards)
10. [Working with Claude](#10-working-with-claude)
11. [Decisions and alternatives considered](#11-decisions-and-alternatives-considered)
12. [Open questions](#12-open-questions)

---

## 1. Repositories
There are three kinds of repos, plus this folder. Separate repos give each one its own commit history and one release model. Apps pin a platform version, so nothing gets redeployed unless I'm working on it.

| Repo | Contains | Release model |
|---|---|---|
| `platform` | Shared npm packages, Terraform modules, and reusable CI workflows | **Published:** semantic versions via release-please |
| `platform-infra` | Shared live infrastructure | **Deployed:** dev on merge, prod on approval, date tags |
| One per app (`budget`, `events`, `admin`) | The app's frontend, backend, and IaC | **Deployed:** dev on merge, prod on approval, date tags |
| `suite` (this folder, `sbwp/suite`) | This README, the suite-wide `CLAUDE.md` | Not released |

### `platform`
Everything apps share as code. Nothing in it is deployed. A release publishes new versions, and nothing changes anywhere until an app upgrades. That upgrade is an ordinary app PR, so a new platform version is always tested in an app's dev environment before prod. **This repo has no AWS credentials**, only an npm publish token.

- **npm packages**, published publicly to **npmjs.com** under the `@sbwp` scope (`@sbwp/runtime`, `@sbwp/build`, `@sbwp/web`), with no token needed to install them. The repo is a pnpm workspace with three packages. They're kept separate so, for example, Lambda bundles never include React and the esbuild tooling never ships at runtime.
    - **`runtime`**: the backend library. It wraps handlers, parses and validates requests, builds the `User`, and provides `db`, `secrets`, and `log` without app code knowing it runs on AWS.
    - **`build`**: the CLI. It reads an app's routes manifest, bundles the Lambdas, runs the local dev environment, and checks the Node version.
    - **`web`**: the frontend package. It has the auth client, the typed API client, test helpers, and the base theme.
- **Terraform modules**, e.g. an app API, a static site, and `app-auth`. App repos reference them by pinned git tag and never apply them on their own.
- **Reusable GitHub Actions workflows** for PR checks and the app deploy pipeline (see [Shared CI](#shared-ci)).
- **`local-dev/`**: the Caddy config and the app port list (see [Local development](#8-local-development)).

**One shared version.** The packages, modules, and workflows are released together under one tag (e.g. `v1.4.0`), and an app pins that one version everywhere. The main reason is that `build`'s output and the Terraform module's input have to match, and a single version makes it impossible to pin incompatible ones.

**Grow only as needed.** The platform starts with what Budget needs. Anything else gets pulled in once a second app needs the same thing.

### `platform-infra`
Shared live infrastructure, `terraform apply`d the same way apps deploy (see [Release flows](#release-flows)):
- AWS account setup and GitHub-to-AWS OIDC roles
- Terraform state buckets and the build artifacts bucket
- Atlas projects and clusters
- the Cognito user pool, its pre-token-generation Lambda, and the invite script
- each environment's DNS zone, wildcard certificate, and SES identity

It holds the most powerful AWS roles in the suite, and it is the only repo that has them.

`platform-infra/management/` is the exception to automation. It's a tiny Terraform config for the main DNS zone that I apply by hand and CI never runs (see [Domains and DNS](#domains-and-dns)).

**The contract between repos is SSM parameters.** `platform-infra` writes shared values (the Atlas project ID, the user pool ID, certificate ARNs, the artifacts bucket) to well-known SSM parameters such as `/platform/atlas/project-id`, and the platform's Terraform modules read them. No repo reads another repo's Terraform state. A change that needs both sides takes two PRs: deploy the infra first, then release the module.

### App repos
Each app repo contains that app's frontend, backend, and IaC. CI uses path filters, so a frontend-only change doesn't redeploy the backend, for example. Apps depend on a pinned platform version and upgrade on their own schedule. Renovate opens the upgrade PRs.

### The `suite` folder
This `apps/` folder is its own small repo. It holds this README and the suite-wide `CLAUDE.md`. Its `.gitignore` excludes the other repos, which are cloned inside it, and the local-only folders:
- **`local-resources/`**: material I provide for working on the suite
- **`memory/`**: Claude's handoff notes (see [Working with Claude](#10-working-with-claude))

---

## 2. Architecture overview

```
Browser ──► CloudFront (budget.app.sabrinabea.com)
              ├─ /*      ──► S3 (React app)
              └─ /api/*  ──► API Gateway HTTP API
                               └─ JWT authorizer (Cognito token + route scope)
                                    └─► Lambda: runtime wrapper ──► handler(request, ctx)
                                                                       └─► Atlas (own database, IAM auth)
Sign-in ──► Cognito managed login (auth.app.sabrinabea.com)
```

- **Language:** TypeScript on Node.js everywhere.
- **Frontend:** React built with Vite, hosted on S3 behind CloudFront. Mantine for components.
- **Backend:** one Lambda per route behind an API Gateway HTTP API, bundled with esbuild and defined by a TypeScript routes manifest.
- **UI and API share one domain.** CloudFront sends `/api/*` to API Gateway and everything else to S3. The local proxy does the same. There's no CORS, frontend code is identical in every environment, and the auth cookie works.
- **Auth:** one Cognito user pool per environment, shared by all apps. Access to each app is granted per user through scopes.
- **Data:** Mongo Atlas free-tier clusters. Each app has its own database.
- **IaC:** Terraform. **CI/CD:** GitHub Actions.

---

## 3. Backend

### Routes manifest
Each app has one TypeScript manifest. It is the single source of truth for the app's routes, scheduled jobs, scopes, and Cognito groups:

```ts
export default defineApp({
    scopes: ['events:guest', 'events:host'],
    baseScope: 'events:guest',
    routes: [
        route('GET', '/events', './handlers/listEvents'),
        route('POST', '/events', './handlers/createEvent', { scopes: ['events:host'] }),
        route('GET', '/health', './handlers/health', { public: true })
    ],
    schedules: [
        // e.g. Events reminders: same handler pattern, triggered on a schedule
    ]
});
```

- **Typed throughout.** `defineApp` uses a `const` type parameter, so literal types are inferred without `as const`. `baseScope`, route scopes, and `user.has(...)` in handlers only accept declared scopes, so a typo fails to compile.
- **Checked at build time.** The build fails if a handler file is missing, or if a handler's input types don't match its route (e.g. a `/events/:id` handler that doesn't expect `id`).
- Routes can later carry per-route settings such as memory and timeout.
- Scope rules are covered under [Scopes](#scopes).

### Build and deploy
The `build` CLI reads the manifest and, for each route:
1. Generates an entry file that wraps the handler with the `runtime` library and bakes in the route's full scope set.
2. Bundles and zips it with esbuild. Builds are deterministic, so unchanged code produces identical zips.

It also writes a **routes JSON** describing every route and scope. The platform's Terraform module reads it and creates one Lambda, one log group (with short retention, to keep costs down), and one API route per entry. It also configures the JWT authorizer for each route. The `app-auth` module creates one Cognito group per scope. Terraform only updates functions whose zip changed.

The zips, the routes JSON, and the frontend bundle together are the **build artifact** that gets promoted from dev to prod.

### Handler contract
A handler file exports one function, `handler(request, ctx)`. Handlers never see raw Lambda events or AWS SDKs.

**`request`** contains only parsed, validated data:

```ts
type Request<Params, Query, Body, Scope extends string> = {
    params: Params; // path params
    query: Query; // query string
    body: Body; // JSON body
    user: User<Scope>; // absent on public routes (enforced by the types)
};

type User<Scope extends string> = {
    id: string; // stable ID from the auth server
    email: string;
    name?: string;
    scopes: Scope[];
    has: (scope: Scope) => boolean;
};
```

- **Validation:** each handler declares zod schemas for `params`, `query`, and `body`. The wrapper validates each request and returns 400 on failure. The handler's types come from the same schemas, which is also what powers the build-time route check and the typed API client.
- **`User`:** the wrapper builds it from token claims, so provider-specific claim names never reach app code. Raw claims aren't exposed. Fields are added only when an app needs them.

**`ctx`** provides the handler's dependencies: `db`, `secrets`, and `log`. The `runtime` library supplies real implementations in AWS and local ones in development, and unit tests pass in fakes. App code knows it's talking to MongoDB, but not how it's connected or where secrets live.

**Output:**
- A returned value becomes a 200 JSON response. Helpers like `created(x)` and `noContent()` cover other successes, and they keep type information so the API client knows each route's response type.
- Thrown platform errors (`NotFoundError`, `ForbiddenError`, …) become their status codes.
- Anything else becomes a 500. It's logged in full, and no details are sent to the caller.

### Scopes
- **Every non-public route requires the app's base scope.** A route's extra scopes are added to the base, never substituted, so `POST /events` requires `events:guest` and `events:host`. Forgetting one can only make a route stricter.
- **Public routes** have no authorizer and no scope checks, and their handlers get no `user`.
- **Privileged users also hold the base scope** (a host is also a guest). The invite script adds the base group automatically.
- **Handlers can check scopes for finer-grained behavior**, e.g. `user.has('events:host')` to show hosts every RSVP but guests only their own.
- How scopes are enforced is covered under [Checking tokens](#checking-tokens).

---

## 4. Frontend
Apps don't need to look like one suite: there's no shared navigation, and each app has its own primary color. The rule is **share logic, not appearance.**

### The `web` package
- **Auth client:** works with the platform's built-in auth routes. It keeps the access token in memory only, refreshes it through the HttpOnly cookie, and handles sign-in and sign-out. It's exposed as `useUser()` and `<RequireAuth>` (see [Sign-in flow](#sign-in-flow)).
- **Typed API client:** calls `/api/...`, attaches the access token, refreshes and retries once on a 401, and turns error responses back into the backend's error types. `createClient<typeof routes>()` types it from the app's own manifest:
    - paths autocomplete from the real routes
    - `params`, `query`, and `body` are checked against each handler's zod schemas
    - response types come from what each handler returns

  For example, `api.get('/events/:id', { params: { id } })`. Any backend change that breaks a frontend call fails type checking. The generic types get type-level tests with Vitest's `expectTypeOf`. Types only cover calls within one app. It's built in two steps: the basic client alongside Budget's first routes, then the typed layer on top. The typed layer needs no handler changes.
- **Test helpers:** a fake auth client and a fake API client for Vitest and React Testing Library, plus a Playwright fixture that signs in through the local dev sign-in page with chosen scopes.
- **Base Mantine theme (contrast tuning only):**
    - a near-black dark background (`#121212`, like Material UI)
    - brighter text and dimmed text
    - a darker primary shade in dark mode
    - `autoContrast`

  Each app extends it with `mergeThemeOverrides` and sets its own primary color. Text meets WCAG AA contrast (4.5:1).

### UI components
Each app uses **Mantine** directly, and no shared components are built up front. If the same component ends up in two apps and really is identical, it moves into `web` then. Why Mantine:
- It's a dependency, so its code doesn't count toward coverage.
- It includes forms, modals, notifications, and date pickers.
- It styles with plain CSS modules, which apps use too.
- It has built-in color scheme support.

### Color scheme
Apps follow the system setting (`defaultColorScheme="auto"`). To avoid a white flash before the JavaScript loads, each `index.html` sets `<meta name="color-scheme" content="light dark">` and gives the body background a `prefers-color-scheme` media query.

---

## 5. Authentication
One shared sign-in for the whole suite. Access is invite-only and granted per app by me. There is no public sign-up.

### Cognito
The auth server is **AWS Cognito**: one user pool per environment on the **Essentials** plan, defined in `platform-infra`.
- **Cost:** $0 up to 10,000 monthly active users (a permanent free tier), then $0.015 per user.
- **Why Essentials:** passkeys, email one-time codes, managed login pages, and access-token customization. At this scale it costs the same as Lite ($0).
- **Changing plans:** the plan is one setting (`user_pool_tier`, which must be set explicitly), changeable in place with no user migration. If usage ever passes 10,000 users and the Essentials features aren't in use, Lite ($0.0055 per user) is an option.
- **Portability:** app code never sees Cognito. It only sees the `User` built from claims.

### Sign-in methods
- **Email one-time codes** are the base method, with no passwords. Self sign-up is off, so an invitation is just a user created with the friend's email.
- **Passkeys** are an optional upgrade. After signing in with a code, a user can add a passkey and skip codes on that device.
- **Email is sent through SES** from each environment's own subdomain (see [Domains and DNS](#domains-and-dns)), not Cognito's built-in sender, which is capped at about 50 emails a day and looks like spam. SES costs about $0.10 per 1,000 emails.

### Granting access
- **Cognito groups are named after scopes** (`budget:owner`, `events:guest`, `events:host`). Being in a group grants the scope with the same name. The manifest's `scopes` list creates the groups.
- **Each app owns its auth resources** through the `app-auth` Terraform module: its own app client in the shared pool (each app has its own callback URL and client secret) and its groups. The user pool ID comes from SSM. Adding a role never requires a `platform-infra` change.
- **Users are managed with an invite script** in `platform-infra`, run locally with my AWS credentials:

  ```sh
  pnpm invite friend@example.com --group events:guest --env prod
  ```

  It creates the user if needed, adds the group plus the app's base group, and sends a short invitation email through SES with the app's link. Cognito's own invitation email is built around temporary passwords, so it's turned off. A matching `revoke` command removes a group.
- **Users are never listed in code.** The repos are public, so the user list lives only in Cognito.
- The **Admin** app will eventually replace the script with a UI.

### Checking tokens
- **API Gateway's JWT authorizer verifies every token:** signature, issuer, expiry, the app's own client, and the route's scope. The Terraform module configures it from the routes JSON. It's free, and invalid requests never invoke a Lambda. Each app accepts only tokens issued to its own client. Calls between apps, if they ever happen, will need a deliberate design (e.g. machine-to-machine tokens).
- **A pre-token-generation Lambda** on the user pool copies group names into the standard `scope` claim, which is the only claim API Gateway can check scopes against.
- **API Gateway can only check "any of these scopes."** So the module gives it each route's most specific scope, and the handler wrapper checks the full set. Privileged users always hold the base scope, so the two layers agree, and the wrapper is the strict check.
- **The wrapper trusts verified claims and fails closed.** Lambdas can only be invoked by their API, so the wrapper doesn't re-verify signatures. It returns 401 if a non-public route receives no claims and 403 if required scopes are missing, so a misconfigured deployment can't expose a route. It runs identical code locally and in AWS.
- **Locally, the dev server does API Gateway's job.** It verifies JWTs with `jose` and builds the same event shape. Verification code exists only there, and it's tested there.

### Sign-in flow
**Sign-in pages are Cognito's managed login** at `auth.app.sabrinabea.com` (prod) and `auth.dev.sabrinabea.com` (dev), branded per app client. Every app signs in through the same domain, so signing into one app signs you into the others.

**The refresh token lives in an HttpOnly cookie, and the access token lives only in memory.** The platform adds these public routes to every app:

| Route | Does |
|---|---|
| `/api/auth/login` | Redirects to managed login with PKCE and a `state` check |
| `/api/auth/callback` | Exchanges the code using the app client's secret (from SSM), then sets the refresh token in an `HttpOnly`, `Secure`, `SameSite` cookie scoped to `/api/auth` |
| `/api/auth/refresh` | Returns a short-lived access token from the cookie. It's a POST that requires a custom header, as protection against cross-site request forgery. |
| `/api/auth/logout` | Revokes the refresh token, clears the cookie, and signs out of Cognito |

The auth client calls `/refresh` on page load, so a valid session signs in instantly. Otherwise, `<RequireAuth>` sends the user to sign in. With this design, an XSS bug can at most use a short-lived access token while the page is open. It can never steal the refresh token, as it could if the refresh token were kept in `localStorage`.

---

## 6. Infrastructure

### AWS accounts
- **dev** and **prod**: where apps and shared infrastructure run.
- **management**: the AWS Organization's management account. It holds the main DNS zone and nothing automated.

Terraform state lives in S3.

### Domains and DNS
The suite uses `sabrinabea.com`. The main site stays at the root domain (GitHub Pages) for now.

| Environment | App | Auth | Hosted zone |
|---|---|---|---|
| Prod | `budget.app.sabrinabea.com` | `auth.app.sabrinabea.com` | `app.sabrinabea.com`, delegated to **prod** |
| Dev | `budget.dev.sabrinabea.com` | `auth.dev.sabrinabea.com` | `dev.sabrinabea.com`, delegated to **dev** |
| Local | `budget.local.sabrinabea.com` → `127.0.0.1` | Fake sign-in, or the dev pool | Wildcard record in the main zone |

**The main `sabrinabea.com` zone is in the management account, and no automation ever writes to it**, because it also holds my Gmail records. Its few suite records come from `platform-infra/management/`, which I apply locally with my own credentials, never from CI:
- NS records delegating `app.` to prod and `dev.` to dev
- the `*.local.sabrinabea.com` → `127.0.0.1` wildcard

Adding an app never touches the main zone. Everything else lives in each environment's delegated zone, managed by `platform-infra`:
- **Certificates:** one wildcard per environment (`*.app.sabrinabea.com`, `*.dev.sabrinabea.com`), in us-east-1 for CloudFront and Cognito, validated automatically in the environment's own zone. They must be in the same account as CloudFront and Cognito, because public ACM certificates can't be shared across accounts.
- **Parent `A` records:** Cognito requires the parent of its custom domain to resolve, so `app.sabrinabea.com` and `dev.sabrinabea.com` each get one. For now it redirects to the main site. Later it could host a landing page for the suite.
- **SES:** prod sends from its own zone (e.g. `invites@notify.app.sabrinabea.com`), and dev from `dev.sabrinabea.com`. The root domain's email records (MX, SPF, DKIM, DMARC) are never touched, and app email can't affect its sending reputation. New SES accounts start in a sandbox that only sends to verified addresses. Prod needs a one-time, free production-access request before invites reach friends. Dev stays in the sandbox and sends only to my verified test addresses.

**Cost:** about $1/month for the two delegated zones.

### Database
- **Mongo Atlas free tier (M0).** Atlas allows one free cluster per project, so dev and prod are separate Atlas projects, each with its own M0 cluster.
- **Each app gets its own database.** Each app's Lambda IAM role maps to an Atlas database user with a built-in `readWrite` role on that database only. That costs nothing extra (Atlas bills per cluster) and needs no custom roles. Apps never read another app's database. They call its API instead.
- **IAM auth, not passwords.** M0 has no private networking, and Lambdas outside a VPC have no fixed IP, so the Atlas IP access list allows all addresses, and IAM authentication is the real access control. A NAT gateway for a fixed IP (about $30+/month) isn't worth it.

### Secrets
- **Storage:** SSM Parameter Store, as standard-tier `SecureString` parameters, under `/<app>/<env>/<name>`. Each app's IAM role can read only its own path.
- **What goes there:** app client secrets and any third-party API keys. The database needs no secret.
- **Access:** apps read secrets only through `ctx.secrets`. A secret that ever needs rotation can move to Secrets Manager without changing app code.

---

## 7. CI/CD and releases

### Branching and PRs
- Feature branches are named `description-of-feature` (kebab-case). They're squash-merged into main via PR and then deleted.
- Branch protection on main requires a PR, passing checks, and squash merges.
- **PR titles follow Conventional Commits** (`feat: add RSVP form`, `fix: rounding in totals`, `feat!: …` for breaking changes), enforced by a CI check. With squash merges, the title becomes the commit on main, so branch commits can be messy. Versions and changelogs are generated from these titles.

### PR checks
Every PR builds, lints, and runs unit tests with coverage thresholds enforced. For IaC, `terraform plan` runs and is posted to the PR. Merging requires all of it to pass.

### Release flows
**Apps** have one workflow run per merge to main:
1. **build:** builds once, then uploads the Lambda zips, routes JSON, and frontend bundle to the S3 artifacts bucket, keyed by commit. GitHub Actions artifacts expire, so they aren't used.
2. **deploy-dev:** automatic.
3. **deploy-prod:** waits for my approval on the GitHub `prod` environment. Approving works from the GitHub mobile app. It deploys the same artifact, never a rebuild, then tags the commit and creates a GitHub Release.

   Environment settings:
    - "Prevent self-review" is off, since I'm the only reviewer.
    - Only one prod deploy runs at a time.
    - A pending approval expires after 30 days.

GitHub's deployment history shows every prod deploy with its commit, approver, time, and release.

**`platform-infra`** follows the same flow without a build job: `plan` on PR, `apply` to dev on merge, and `apply` of the same commit to prod after approval.

**`platform`** is released with **release-please**, as are any future libraries:
- It keeps a release PR open that updates every `package.json` version, `CHANGELOG.md`, and `.release-please-manifest.json`. Versions are never edited by hand.
- Merging that PR tags the release, creates a GitHub Release, and publishes to npm.
- The version bump is the highest one among unreleased commits. For example, a breaking change plus a fix since `1.4.2` produces `2.0.0`.
- While the version is 0.x, breaking changes bump only the minor version.

### Versioning deployed repos
Apps and `platform-infra` don't use semantic versions, since nothing depends on them the way apps depend on the platform:
- Each prod release gets a **date tag**, e.g. `2026.09.24`, with `.2` for a second release that day.
- **Release notes** are generated from Conventional Commit titles since the last release (e.g. with `git-cliff`).
- **Every build has its commit SHA and build time built in**, for footers, logs, and error reports. The release tag points at that commit.
- If apps ever call each other's APIs, breaking changes are handled with versioned API paths (`/api/v2/...`) or backward-compatible changes, not app version numbers.

### Shared CI
- **Reusable workflows:** `platform` provides the PR checks and the deploy pipeline. Each app's workflow files are a few lines that call them at the pinned platform version, e.g. `uses: sbwp/platform/.github/workflows/app-deploy.yml@v1.4.0`. They run with the calling app's own OIDC role, so `platform` still needs no AWS credentials.
- **Tools:** `jdx/mise-action` installs the exact versions from `mise.toml`, and the pnpm store is cached.
- **Renovate** (free for public repos) opens PRs for npm packages, platform versions, Terraform module tags, GitHub Actions, and `mise.toml` tools. Low-risk updates merge automatically once checks pass.

### Public repo safeguards
- The OIDC deploy roles trust only `main` and the named environments, checked through the token's `sub` claim (e.g. `repo:sbwp/budget:environment:prod`). Forks and other branches can't get AWS credentials.
- AWS-connected PR jobs (like `terraform plan`) run only for PRs from branches in the same repo. GitHub's "require approval for fork PR workflows" setting stays on.
- `terraform plan` output is public, so sensitive variables and outputs are marked `sensitive`.
- AWS account IDs are visible in code. AWS doesn't treat them as secret.

---

## 8. Local development
Goal: local work should feel like the deployed app, with almost no setup per app.

**`pnpm dev` is the one command for any app.** It runs the `build` CLI's `dev` command, which:
- starts the local Mongo container if needed
- starts the local backend server from the routes manifest
- starts Vite

Improvements arrive with platform upgrades.

### Local domains
- **`*.local.sabrinabea.com` → `127.0.0.1`**, a wildcard record set up once, so every app has a real-looking address like `budget.local.sabrinabea.com` with nothing installed locally.
- **Caddy** is the local proxy:
    - It sends `/api/*` to the app's local backend and everything else to Vite, the same split CloudFront makes in AWS.
    - It serves HTTPS from its own local certificate authority, which my Mac trusts. That keeps secure cookies and sign-in redirects working like prod, and meets Cognito's HTTPS callback requirement.
    - If testing on other devices is ever needed, a real wildcard certificate via DNS challenge is an option.
- **Ports:** each app has a fixed backend/frontend port pair (e.g. 4101/5101 for Budget, 4102/5102 for Events), listed in `platform/local-dev/` next to the Caddy config. Adding an app means one block and one port pair. Later, the `build` CLI could generate this.

### Local data and config
- **MongoDB runs in Docker on OrbStack**, a lighter Docker alternative for macOS, free for personal use. It uses the standard `docker` CLI, so switching to Docker Desktop would change nothing in the repos.
- The `runtime` library's database abstraction hides the difference between local Mongo and Atlas with IAM auth.
- The `runtime` library defaults to local settings (e.g. `mongodb://localhost:27017/<app>`), so most apps need no local config. The rare local secret goes in a gitignored `.env.local`, which mise loads.

### Local sign-in
- **Fake sign-in is the default.** Locally, `/api/auth/login` shows a dev sign-in page. I pick a user (e.g. `alice@test`) and tick scopes from the manifest's list. The local server signs a JWT with a local key, using Cognito's claim shape (`sub`, `email`, `scope`, `cognito:groups`), and verifies it the way it would a real one. It works offline, and switching roles is instant. The real claims-to-`User` mapping, scope checks, and cookie flow all still run.
- **It's safe by construction.** The fake issuer exists only in the `build` CLI's local server, never in deployed code, and API Gateway would reject its tokens anyway, since the issuer is wrong.
- **Real Cognito on demand.** `AUTH_MODE=cognito` in `.env.local` switches to the dev user pool. Each app's dev app client allows its local callback URL.
- **Playwright runs against the local stack with fake sign-in**, both locally and in CI (CI starts the same Mongo container). Each test picks exactly the scopes it needs. The real Cognito flow is checked manually in dev, or with a smoke test later if that becomes worthwhile.

---

## 9. Standards

### Tooling
- **pnpm** is the package manager. It's strict (code can only import packages listed in its own `package.json`, which keeps the platform's dependencies honest), and it skips dependency install scripts unless they're allowlisted (esbuild is).
- **mise** pins tool versions (Node, pnpm, Terraform) per repo in `mise.toml`. It's also used to load `.env.local`. mise tasks are not used.
- **`package.json` scripts** are the standard way to run things everywhere: `pnpm test`, `pnpm lint`, `pnpm build`, `pnpm dev`.
- **The platform decides the Node major version**, starting with **Node 24** (current LTS with a Lambda runtime; check Lambda's supported runtimes when scaffolding). The esbuild target, the Lambda runtime, and `engines.node` all come from the platform. Each app's `mise.toml` pins the same major version, and the `build` CLI fails clearly on a mismatch. A new Node major version is a major platform release. The frontend's browser target is a separate Vite setting.

### Code style
- **Prettier owns all formatting:** single quotes, semicolons always, 4-space indentation, `printWidth: 100`, `trailingComma: 'none'`, and `bracketSpacing: true` (spaces inside object braces, none inside array brackets). K&R braces.
- **ESLint** handles code quality only, with `eslint-config-prettier` so it never fights Prettier.
- **Arrow functions** are always preferred over `function`, enforced with `prefer-arrow-callback` and `func-style`.
- **husky + lint-staged** format and lint staged files on commit.

### Tests
- **Vitest** for unit tests and **Playwright** for UI and regression tests. Why Vitest:
    - It runs TypeScript and ES modules natively and reuses the Vite config.
    - Its Jest-style API matches Playwright's `expect`.
    - It can test types with `expectTypeOf`.
- **Coverage:** unit tests need 100% line and branch coverage, enforced by thresholds in CI before merging. Playwright tests don't count toward coverage.
- **RACC (Restricted Active Clause Coverage):** for each clause in a decision, there's a pair of tests where that clause determines the outcome, and all other clauses keep the same values across the pair. No tool measures it, so tests are designed for it deliberately and checked in review.
- **Testability by design:** handlers get dependencies through `ctx`, and the frontend gets fake auth and API clients, so unit tests use fakes instead of mocked SDKs.

---

## 10. Working with Claude
- **`CLAUDE.md` files hold Claude's instructions:**
    - `apps/CLAUDE.md` has the suite-wide rules and points here. Every repo is inside `apps/`, so it loads in every session.
    - Each repo's own `CLAUDE.md` has its commands and specifics.
    - General personal preferences live in `~/.claude/CLAUDE.md`.
- **`memory/`** (not in git) holds Claude's handoff notes, one file per repo (e.g. `memory/budget.md`, `memory/suite.md`): what's in progress, what's next, and open threads. Claude reads the relevant file at the start of a session and updates it before stopping. Decisions go in this README or a repo's docs, not in `memory/`.
- **Claude's auto memory** (machine-local, under `~/.claude/projects/`) is only for how I like to work. Anything important gets moved into a `CLAUDE.md`, where I can see it.

---

## 11. Decisions and alternatives considered

| Decision | Chosen | Rejected, and why |
|---|---|---|
| Repo layout | Platform + infra + one repo per app | A monorepo for the whole suite: tangled commit history. Four repos per app: no home for shared pieces. |
| Lambda layout | One Lambda per route, from a manifest | One Lambda per app: simpler, but per-route functions give smaller bundles and per-endpoint deploys. Routes listed in Terraform: duplicated definitions. |
| Platform versioning | One shared version | Separate versions: could pin incompatible `build` and module versions. |
| Library registry | npmjs.com | GitHub Packages: requires a token even to install public packages. |
| App versioning | Date tags + generated notes | Semantic versions: nothing depends on apps, so the numbers would mean nothing, and releases would need an extra step. |
| Prod promotion | GitHub environment approval | A manual release workflow: works, but is less direct. |
| Branch merging | Squash + Conventional Commit titles | Merge or rebase: every commit would have to follow the convention. |
| Unit test runner | Vitest | Jest: extra setup for TypeScript and ESM. Mocha: more pieces, and further from Playwright's style. |
| Package manager | pnpm | npm: not strict about undeclared dependencies. |
| Tool versions | mise | Volta: no longer maintained. |
| Component library | Mantine | shadcn/ui: its components are copied into the repo, so they'd count toward coverage. MUI: heavier, with a strong Material look. |
| Shared UI | Logic only, plus a contrast-tuning theme | A shared component library: slows every UI tweak, and consistency isn't a goal. |
| Auth provider | Cognito Essentials | Auth0: whether RBAC is on the free plan is unclear even from Auth0's own sources, and beyond the free and self-serve tiers, pricing requires a sales quote. |
| Sign-in | Email codes + optional passkeys | Passwords: weakest and hardest to migrate. Google: needs extra Lambdas to stay invite-only, plus account linking. |
| Token checking | API Gateway verifies; wrapper fails closed | Re-verifying in the Lambda: no real benefit. Wrapper only: loses the outer layer. |
| Token storage | Refresh token in an HttpOnly cookie, access token in memory | Everything in the browser: the refresh token would sit in `localStorage`, exposed to XSS. |
| Secrets | SSM Parameter Store | Secrets Manager: $0.40 per secret per month, and rotation isn't needed. |
| Database layout | A database per app on a shared M0 | Collection-level roles in one database: needs custom roles. |
| Atlas networking | Open IP list + IAM auth | NAT gateway: about $30+/month. |
| DNS | Delegated `app.` and `dev.` zones | Automation writing to the main zone: it holds my email records. |
| Local auth | Fake sign-in by default, real Cognito on demand | Cognito only: slow, needs internet, and makes switching roles painful. |
| Containers | OrbStack | Docker Desktop: fine, just heavier. |

---

## 12. Open questions
None right now. New questions go here until they're decided. Once decided, they move into the relevant section above.
