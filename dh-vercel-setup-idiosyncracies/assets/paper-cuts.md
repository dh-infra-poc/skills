# Paper cuts — Delivery Hero × Vercel POC

Every paper cut hit during the POC setup sessions (24h window, 2026-08-10 → 2026-08-11), verbatim from the session logs, grouped by area. Format: what we were doing, what broke, how we got past it.

Sources: CSV dashboard demo app, Eve PR-review agent build, demo-day diagram generation, time-off app (DSQL layer + rollback-demo bug), team onboarding Slack thread (2026-08-11). (POC-alignment session sweep pending — added via PR when extracted.)

---

## Vercel CLI

### `vercel link --yes` is not actually non-interactive when the account has multiple teams
- **Context:** Linking a fresh project to Vercel before the first deploy, using `vercel link --yes` to avoid prompts.
- **Symptom:** Instead of linking, the CLI returned a JSON "pick a scope" response listing `vercel link --yes --scope <team>` variants for every team the account belongs to, with the hint "Run one of the commands in next[] to complete without prompting." The `--yes` flag couldn't choose a default, so the flow stalled on a human decision.
- **Workaround:** Re-run with the team explicit: `vercel link --yes --scope dh-infra`.

### Non-interactive CLI needs explicit `--scope`/`--yes` despite prior team switch
- **Context:** Linking a project right after `vercel teams switch dh-infra`.
- **Symptom:** `vercel link --yes --project <name>` returned `action_required` / "Provide --team or --scope explicitly. No default is applied in non-interactive mode." — the just-switched team is not used as a default. Separately, `vercel connect attach` errored "Confirmation required. Use --yes to skip the confirmation prompt."
- **Workaround:** Always pass `--scope <team>` and `--yes` on every command in non-interactive runs; never rely on `teams switch` state.

### First deploy of a new project goes straight to production, not preview
- **Context:** Deploying with plain `vercel` (no `--prod`), explicitly intending a preview deployment.
- **Symptom:** The deployment came back with `"target": "production"` and was aliased to the `.vercel.app` production domain — the CLI's own `next[]` hints even suggested `vercel deploy --prod` "Promote to production" as if it had been a preview. Because it was the project's very first deployment, it became production regardless of the command's preview default.
- **Workaround:** None found — accept that deploy #1 defines production, so make it a commit you'd stand behind.

### `vercel curl` rejects the standard `--scope` flag
- **Context:** Fetching a protected production URL with `vercel curl https://... --scope <team>`, mirroring the `--scope` usage other subcommands required in the same session.
- **Symptom:** After successfully generating the bypass token, the command failed with `curl: option --scope: is unknown` — `vercel curl` (beta) passes unrecognized flags through to curl instead of handling the CLI's own global flags.
- **Workaround:** Run `vercel curl` without `--scope`; the linked project context is enough.

### Vercel CLI shows a BLOCKED deployment as status `UNKNOWN`
- **Context:** Diagnosing a deploy that never came up.
- **Symptom:** `vercel ls` and `vercel inspect` both rendered the deployment as `status UNKNOWN` with `?` duration. The real state (`"readyState": "BLOCKED"` plus an explanatory `errorLink`) was only visible through the deployments API (e.g. MCP `get_deployment`).
- **Workaround:** When the CLI says `UNKNOWN`, query the deployments API for `readyState` and `errorLink`.

### `vercel connect remove --yes` refuses while a project is attached
- **Context:** Deleting a mis-configured GitHub connector so it could be recreated.
- **Symptom:** `Error: Cannot delete connector github/<name> while it has 1 connected project. Please disconnect any projects using this connector first or use the --disconnect-all flag.` The remove failed, the paired recreate silently bailed (UID still existed), and the connector was verifiably unchanged — a confused round-trip.
- **Workaround:** `vercel connect remove --yes --disconnect-all`.

### `vercel integration add --yes` is not a valid flag (contradicts the marketplace skill's own docs)
- **Context:** Installing the Aurora DSQL marketplace integration per the Vercel plugin's marketplace skill, which explicitly instructs `vercel integration add <name> --yes --no-claim`.
- **Symptom:** `Error: unknown or unexpected option: --yes` (Vercel CLI 55.0.0). Immediate exit code 1.
- **Workaround:** The correct flags are `--non-interactive` plus `-m primaryRegion=<region>`: `vercel integration add aws/aws-dsql -m primaryRegion=fra1 --no-claim --non-interactive`.

### Integration provisions under the CLI's default team scope, not the linked project's team
- **Context:** Project was linked to `dh-infra/time-off-app`, then `vercel integration add aws/aws-dsql` was run from that directory.
- **Symptom:** CLI provisioned a real DSQL cluster under the account's *default* scope (a different team); the auto-connect step then failed with `Error: Failed to connect: Project not found (404)`, leaving an orphaned AWS resource in the wrong team and no env vars wired up.
- **Workaround:** Delete the stray resource (`vercel integration resource remove <name> --yes --scope <wrong-team>`) and re-provision with an explicit `--scope dh-infra`. Always pass `--scope` to `integration add`.

### `vercel integration list` silently hides resources not connected to the current project
- **Context:** Hunting for the stray DSQL resource to clean it up.
- **Symptom:** `vercel integration list --scope <team>` returned `> No resources found.` even though the resource existed — the command is project-scoped by default, and the stray resource wasn't connected to any project.
- **Workaround:** `vercel integration list --all --scope <team> --format json`.

### `vercel integration add` silently installs a third-party agent skill bundle into the repo
- **Context:** Same DSQL install command.
- **Symptom:** After provisioning, the CLI ran `npx skills add https://github.com/aws/agent-toolkit-for-aws --skill aurora-dsql` on its own — cloned the AWS repo, created `.agents/`, `skills-lock.json`, symlinked `.claude/skills/aurora-dsql`, and edited `.gitignore`, all without asking. Repo state was unexpectedly dirty with third-party files.
- **Workaround:** Decide keep-or-revert explicitly; if kept, land it as its own PR to keep the feature PR reviewable.

---

## Team policy & Deployment Protection (enterprise)

### Team deployment policy blocks CLI deploys
- **Context:** First production deploy via CLI (`eve deploy`, which shells out to a CLI-sourced deploy) on the dh-infra team.
- **Symptom:** Deployment was created in state `BLOCKED` (`"source": "cli"`, errorLink to the deployment-policy docs) — the team's policy disallows CLI-sourced deploys.
- **Workaround:** Push to a GitHub repo, `vercel git connect`, and deploy via Git pushes from then on. Assume Git-only deploys on DH teams.

### SSO Deployment Protection blocks browser verification of deployed apps
- **Context:** Opening a freshly deployed production URL in a browser to visually confirm it matched local.
- **Symptom:** Navigation landed on an Okta sign-in page instead of the app — the team's Deployment Protection/SSO gates all URL access, so the app cannot be visually inspected without a team Vercel login.
- **Workaround:** Verify via authenticated `vercel curl` (HTTP 200 + grep expected page content); the CLI auto-generates a deployment protection bypass token.

### AWS marketplace terms acceptance blocks non-interactive install with a mandatory browser step
- **Context:** Installing a marketplace integration (Aurora DSQL) under a team that had never installed an AWS product.
- **Symptom:** Exit 1 with `"reason": "integration_terms_acceptance_required"` — "Accept marketplace terms for 'AWS' in your browser before this install can finish... This command does not wait for acceptance in non-interactive mode." Fully automated provisioning is impossible for a team's first AWS install.
- **Workaround:** A human accepts terms once at `https://vercel.com/<team>/~/integrations/accept-terms/aws?source=cli`, then the same command succeeds on retry.

### Passport-protected preview deployments cannot be verified headlessly — every documented access path fails
- **Context:** Verifying a change on a PR preview deployment where the app is gated by Vercel Passport (`requireUser()` throws without an identity).
- **Symptom:** Chained failures: browser pane redirected to Vercel login; `vercel curl`'s Deployment Protection bypass token got past protection but the page 500'd because bypass tokens carry no Passport identity; `agent-browser` wasn't installed; the `x-vercel-trusted-oidc-idp-token` header path also 500'd; the MCP `get_access_to_vercel_url` tool failed ("Unable to create shareable URL"). Deployment Protection bypass and Passport identity are separate layers, and nothing in the tooling gets past both headlessly.
- **Workaround:** Drive the user's real Chrome (claude-in-chrome), which carries a live Passport session. Plan for this whenever an agent must verify a Passport-gated deployment.

### Finding a PR's preview URL is indirect when the team uses a custom preview domain
- **Context:** Locating a PR's preview deployment to verify a change on it.
- **Symptom:** `gh pr view --json statusCheckRollup` only exposes the Vercel inspector URL, not the preview URL; grepping PR comments for `https://*.vercel.app` finds nothing because the team's previews live on a custom domain (`*.dh-vercel.com`).
- **Workaround:** Take the deployment ID from the status check, query the deployments API (`get_deployment`), and read the `branchAlias`/`alias` field.

### vercel[bot] deployment comment fires a false-positive "new review comment" event
- **Context:** After opening a PR, a CI-monitor event claimed "1 new review comment" requiring action.
- **Symptom:** The "review comment" was just vercel[bot]'s standard deployment-status table (base64 metadata blob + preview links) — no human feedback existed. Verifying required fetching all PR comments, inline comments, and reviews via `gh api`.
- **Workaround:** Treat vercel[bot] comments as deployment status, not review feedback, before investigating.

### Deployment Protection 401s muddy webhook debugging
- **Context:** Diagnosing why GitHub webhook mentions produced nothing — testing whether the webhook route was reachable.
- **Symptom:** Unauthenticated POSTs to the webhook route returned `{"error":{"message":"Protected deployment","code":"401"}}`, making it ambiguous whether Deployment Protection was eating GitHub's webhooks or the route was fine.
- **Workaround:** Use OIDC-authenticated `vercel curl` to test the route independently of protection. (In our case protection was NOT the blocker — missing webhook events were.)

---

## Vercel Connect (GitHub connectors)

### Default GitHub connector only subscribes to `pull_request` webhooks
- **Context:** An agent invoked by `@mention` comments needs `issue_comment` and `pull_request_review_comment` webhook events.
- **Symptom:** Mentions produced no webhook at all — no runtime logs, no reaction, completely silent. The Connect-managed GitHub App had `events: ["pull_request"]` only; the generic `vercel connect create github` provisions a CI-flavored default.
- **Workaround:** No CLI way to edit events on an existing connector — delete and recreate with explicit `--trigger-event issue_comment --trigger-event pull_request_review_comment --trigger-event pull_request`, repeating the browser install flow.

### Connect trigger destination defaults to a branch literally named `production`
- **Context:** Webhooks from the GitHub App are forwarded by Connect to a deployment selected by a trigger destination.
- **Symptom:** The connector's trigger pointed at a git branch named `production` (the CLI default) — a branch that doesn't exist — so forwarded events had no valid destination.
- **Workaround:** `vercel connect attach ... --trigger-branch main`.

### GitHub App slug never matches the connector name (botName mismatch, twice)
- **Context:** eve's `githubChannel({ botName })` must exactly match the GitHub App slug for mentions to be recognized.
- **Symptom:** A connector named `pr-review-agent` produced an app slugged `pr-review-agent-dh`; after delete+recreate the new app got yet another slug (`dh-pr-review-agent`) because the old slug was still reserved. Each mismatch fails silently — mentions just do nothing.
- **Workaround:** Discover the real slug via `gh api orgs/<org>/installations` and set `botName` to that, after every connector recreate.

### Deleting a connector orphans its GitHub App on the org
- **Context:** Connector delete/recreate cycle.
- **Symptom:** The old GitHub App stayed installed org-wide doing nothing after `vercel connect remove`, and kept its slug reserved (forcing the new app onto a different slug).
- **Workaround:** Manually uninstall the orphaned app at `github.com/organizations/<org>/settings/installations` after removing a connector.

---

## eve (agent framework)

### `eve add channel/github` gated on an unreleased eve version
- **Context:** Installing the eve GitHub channel via the guided registry setup (`pnpm exec eve add channel/github`).
- **Symptom:** `This registry item requires eve >=0.31.4, but this project is using eve 0.31.3. Upgrade eve and run the command again.` — but npm's latest published eve is 0.31.3, so the demanded upgrade is impossible. The channel code itself ships in 0.31.3.
- **Workaround:** Perform every guided step manually (`vercel link`, `vercel connect create github`, `vercel connect attach --triggers`, hand-write `agent/channels/github.ts`) — and expect the connector paper cuts above, which the guided flow would have avoided.

### `eve deploy` hangs silently on a blocked deployment
- **Context:** First deploy on a team whose policy blocks CLI deploys.
- **Symptom:** `eve deploy` produced zero output and never returned — blew through a 600s timeout with an empty output file. The BLOCKED state was never surfaced by the command.
- **Workaround:** Kill it; check `vercel ls` / the deployments API for the real state.

### eve sandbox repo checkout fails with git `safe.directory` error — and is swallowed
- **Context:** Each agent turn checks the PR's repo out into the sandbox so reviews can trace callers.
- **Symptom:** Runtime logs show `[eve:github.defaults] GitHub checkout failed — swallowed ... fatal: detected dubious ownership in repository at '/workspace' ... git config --global --add safe.directory /workspace`. eve swallows it, so reviews silently degrade to PR diff + metadata only; the appended hint ("Verify the GitHub App installation has access to this repository.") is misleading.
- **Status:** Unresolved — framework bug in eve 0.31.3 (likely why the registry wants 0.31.4). Upgrade when 0.31.4 ships.

---

## Next.js & project tooling

### create-next-app refuses to scaffold in a directory containing a data file
- **Context:** Scaffolding a Next.js app in an existing project directory that contained only a source data file (`b2b_invoices.csv`).
- **Symptom:** `npx create-next-app@latest .` aborted: "The directory contains files that could conflict: b2b_invoices.csv. Either try using a new directory name, or remove the files listed above." — a plain CSV can't conflict with the template, but the check is name-agnostic.
- **Workaround:** Move the file out to a temp directory, scaffold, move it back.

### `server-only` import breaks any script run with tsx outside Next.js — twice, in two different ways
- **Context:** Running a DB migration script (`dotenv -e .env.local -- tsx scripts/migrate.ts`) that imported `lib/db/client.ts`.
- **Symptom:** First run: `Error: Cannot find module 'server-only'` — Next.js aliases `server-only` internally so it was never a real dependency. After `pnpm add server-only`, second run: `Error: This module cannot be imported from a Client Component module.` — the real npm package unconditionally throws when loaded outside a bundler.
- **Workaround:** Remove `import 'server-only'` from modules that scripts need (rely on a guard in the importing module instead).

### `pnpm lint` fails — v0 scaffold ships a lint script without eslint installed
- **Context:** Running the standard verify loop (typecheck / lint / build).
- **Symptom:** `sh: eslint: command not found` — the v0-generated `package.json` has `"lint": "eslint ."` but eslint is not a dependency.
- **Status:** Unresolved — lint skipped; typecheck and `next build` used as the gate instead.

---

## Database layer (Aurora DSQL via node-postgres)

### node-postgres parses SQL `DATE` into local-timezone JS `Date` objects, silently corrupting string-based date logic
- **Context:** First end-to-end test of the DSQL-backed store: a booking was created, success message showed, but balance and upcoming list didn't update.
- **Symptom:** No error anywhere — `pg` returns `DATE` columns as JS `Date` objects (local timezone), while the app compares dates as plain `'YYYY-MM-DD'` ISO strings, so rows silently failed every comparison. Required direct DB queries to discover the row *had* landed.
- **Workaround:** Override the `pg` type parser for `DATE` to return plain strings, and normalize `TIMESTAMPTZ` in the row mapper.

---

## Vercel docs & brand assets

### Vercel brand-token docs page withholds all literal values
- **Context:** Extracting official Vercel dark-brand colors/fonts from `https://vercel.com/design.dark.md` to bake into standalone SVG/HTML files.
- **Symptom:** The page contains zero hex/rgba/oklch values — it deliberately withholds implementation details, exposing only token *names* (`--vbg-surface-primary`, etc.) and instructing generators to link the published stylesheet rather than read values. Useless when literal values must be embedded in self-contained files.
- **Workaround:** Curl `https://vercel.com/geist/vercel-brand.css` directly, grep the `--vbg-*` declarations, and convert `light-dark(oklch(...))` values to hex by hand.

### Geist-styled pages depend on Google Fonts at runtime — offline risk
- **Context:** Generating self-contained demo pages that must render with zero network (client venue wifi risk).
- **Symptom:** The diagram template loads Geist via a `fonts.googleapis.com` `<link>`; with no network, everything falls back to system-ui. Automated verifiers kept flagging it but left it "for deck consistency" — it recurred across generation runs.
- **Workaround:** Post-process: download the Google Fonts CSS with a Chrome UA (required to be served woff2), fetch each woff2, base64-inline as `data:font/woff2` URIs, then re-verify visually.

### Vercel Agent code review demoted from demo to mention (secondhand)
- **Context:** Referenced in the demo-day battle plan; the originating attempt was in another session.
- **Symptom:** A first Vercel Agent code-review attempt failed or underwhelmed enough to pull it from the live demo lineup.
- **Status:** Downgraded to a spoken mention; retry optional.

---

## Onboarding & environment (from the team onboarding thread, 2026-08-11)

### `vercel git connect` blocked until the Vercel account has a GitHub Login Connection
- **Context:** Onboarding a new team member; the Vercel project was created fine, but connecting the GitHub repo failed.
- **Symptom:** Connecting the repo requires the personal Vercel account to have a GitHub Login Connection — and there is **no CLI flag or API** for it; it's a browser OAuth consent tied to the personal Vercel login, so the agent cannot do it on the user's behalf.
- **Workaround:** The user signs in with GitHub at `https://vercel.com/account/settings/authentication` (same account authenticated in `gh`), then `vercel git connect` is retried and works.

### Warp VPN + Claude Code interaction blocks requests
- **Context:** New-machine onboarding.
- **Symptom:** With Warp VPN connected, Claude Code's requests are blocked/hang.
- **Workaround:** Disconnect Warp while using Claude Code.

### `pnpm install` blocked by org supply-chain policy on pre-existing lockfile entries
- **Context:** Refreshing `pnpm-lock.yaml` after an unrelated code change on a teammate's machine.
- **Symptom:** `pnpm install` was blocked by the org's supply-chain policy (`minimumReleaseAge`), which flagged several *pre-existing* lockfile entries (`@vercel/connect`, `ai`, …) as too-recently-published — unrelated to the change being made. The lockfile could not be refreshed.
- **Workaround:** Don't bypass the policy. Ship without the lockfile refresh, note it in the PR, and re-run `pnpm install` once the flagged entries clear the release-age window (or from an unaffected environment).

### Repos created outside the `dh-infra-poc` org can't deploy — and agents asked instead of defaulting
- **Context:** Multiple onboarding sessions creating demo repos.
- **Symptom:** Deploys only work for repos in the `dh-infra-poc` GitHub org; agents also wasted time asking users which org to use, and one nearly acted on the production `Delivery-Hero` org.
- **Workaround:** Hard rule — only ever create repos in `dh-infra-poc`, never ask, never touch `Delivery-Hero`.

### Demo shipped as a Claude artifact instead of a Vercel deployment
- **Context:** Demo delivery during onboarding.
- **Symptom:** The agent delivered the result as a Claude artifact when the expected deliverable was a deployed Vercel app.
- **Workaround:** Standing instruction — ship demos as Vercel deployments.

---

## Claude Code / agent tooling (encountered along the way)

### Browser pane screenshots/scrolls repeatedly time out when the pane is hidden
- **Symptom:** Screenshot and scroll calls fail with "computer timed out after 30s. The Browser pane is currently hidden." even when the page is healthy — hit in three separate sessions; a stale early screenshot also caused a false "chart not rendering" alarm.
- **Workaround:** Retry; every retry succeeded.

### `preview_start` requires hand-writing `.claude/launch.json` first
- **Symptom:** First call errors with "No .claude/launch.json found" plus a template; the tool can't infer `npm run dev` / port 3000 even for a stock Next.js app.
- **Workaround:** Write the launch.json boilerplate, retry.

### ListAgents can't reach another session in the same directory
- **Symptom:** A parallel session working in the same project never appeared — only the session's own subagents, then "No reachable agents." No way to distinguish "ended" from "unreachable."
- **Status:** Unresolved; coordinate through files or the user instead.

### Workflow resume scripts scattered across per-cwd project slugs
- **Symptom:** Three Workflow runs from one session pointed their resume `scriptPath` at three different `~/.claude/projects/` slugs, keyed to the transient cwd at invocation time. A resume would require hunting the right slug.
- **Status:** Latent — no impact until a resume is needed.

### Auto-mode permission classifier blocked a destructive-looking fix
- **Symptom:** `vercel connect remove` was denied by the auto-mode classifier; the manual hand-off then hit the `--disconnect-all` error, costing a round-trip. ("Why didn't you run this yourself?")
- **Workaround:** Explicit user go-ahead in chat, then the command ran fine.

### Background deploy-waiter went silent for ~40 minutes
- **Symptom:** A background loop waiting for a production deploy never fired its completion notification — the deploy had been Ready within seconds (10s build) but the session idled ~40 min until the user prodded; a stale task indicator lingered afterwards. Related: the harness blocked `sleep 45 && <check>` polling patterns.
- **Workaround:** Check deploy status directly instead of trusting the waiter; use Monitor/until-loop patterns instead of `sleep &&` chains.
