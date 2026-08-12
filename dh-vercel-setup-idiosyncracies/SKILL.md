---
name: dh-vercel-setup-idiosyncracies
description: Setup conventions and known paper cuts for Delivery Hero POC work on Vercel — the required toolchain (nvm/Node 24, pnpm), the dh-infra-poc org and dh-infra team rules, the PR-first workflow, eve for agents, connector rules (Jira, BigQuery, Slack), plus workarounds for CLI linking/scoping, marketplace integrations, deployment policy, Deployment Protection/SSO/Passport, and Vercel Connect GitHub connectors. Use when you start a DH project, set up a machine, scaffold, link, provision integrations, deploy, or debug deployments, webhooks, or previews on a DH Vercel team — or when a Vercel CLI command stalls, errors, or behaves unexpectedly.
---

# Delivery Hero × Vercel setup idiosyncracies

Field notes from the DH POC (August 2026). These are real failures hit during setup, each with the working fix. Skim the rules below before running setup commands; consult [assets/paper-cuts.md](assets/paper-cuts.md) for the full log with exact error messages, context, and unresolved items.

## Ground rules (non-negotiable)

- **Use only the `dh-infra-poc` GitHub org.** Do NOT touch the `Delivery-Hero` org. You cannot deploy a repo from a different org.
- **Do not ask which org.** Use `dh-infra-poc`.
- **Create a PR. Do not push to main.** Push to main only if the user asks for it directly.
- **Deploy on Vercel, on the `dh-infra` team.** GitHub repos go in `dh-infra-poc`. Vercel projects go in `dh-infra`.
- **Deliver demos as Vercel deployments.** Do not deliver a Claude artifact.
- **Build all agents with eve** (Vercel's agent framework). Do not build a custom agent stack.

## Toolchain (same setup for all org members)

- **Node 24 or later. Install it with nvm.**

  ```bash
  nvm install 24 && nvm use 24
  ```

- **Use pnpm. Set the minimum release age to 0.** The default policy blocks `pnpm install` on new package versions. See rule 27.

  ```bash
  pnpm config set minimumReleaseAge 0
  ```

## Project workflow

1. **Create the GitHub repo first.** A repo in `dh-infra-poc` is necessary for a `dh-infra` deploy.
2. **Type check after each change.** Make the type check a gate.
3. **Write logs through all of the app.** Logs make the app easy to debug. Read the logs with `vercel logs` or the runtime-logs tool.
4. **Keep the app debuggable behind Passport.** An agent cannot open a Passport-gated page. See rules 11–12.
5. **Use the preview deployment to check the app.** Do not trust the local result only.
6. **Open a PR.**

## Rules that prevent the most lost time

### CLI scoping — nothing defaults to the team you expect
1. **Always pass `--scope dh-infra` (or the relevant team) explicitly** on `vercel link`, `vercel integration add`, and any non-interactive command. `--yes` does NOT pick a default team, and a prior `vercel teams switch` is ignored in non-interactive mode.
2. **`vercel integration add` provisions under the CLI's default scope, not the linked project's team.** Without `--scope` you'll orphan real cloud resources in the wrong team and the auto-connect will 404.
3. **`vercel integration add --yes` is not a flag.** Use `--non-interactive`, with `-m key=value` for required metadata, e.g. `vercel integration add aws/aws-dsql -m primaryRegion=fra1 --no-claim --non-interactive`.
4. **`vercel integration list` hides resources not connected to the current project** — add `--all` when hunting for strays.
5. **`vercel curl` does NOT accept `--scope`** (it forwards unknown flags to curl). Run it from the linked project directory instead.

## Deploys

6. **DH team policy blocks CLI-sourced deploys** — deployments arrive in state `BLOCKED`. Push to GitHub and use `vercel git connect`; deploy via Git from then on.
7. **The CLI shows a BLOCKED deployment as `UNKNOWN`** — query the deployments API (`get_deployment`) for the real `readyState` and `errorLink`.
8. **The very first deploy of a new project becomes production** even without `--prod`. Make deploy #1 a commit you'd stand behind.
9. **A team's first AWS marketplace install requires a human** to accept terms in the browser once (`integration_terms_acceptance_required`); retry the same command after.
10. **`vercel integration add` may silently run `npx skills add`** for the provider's skill bundle, dirtying the repo (`.agents/`, `skills-lock.json`, `.claude/skills/...`). Decide keep-or-revert explicitly; if kept, land as a separate PR.

## Protected deployments (SSO / Deployment Protection / Passport)

11. **You cannot browser-verify deployed apps anonymously** — team SSO gates all deployment URLs. Verify with authenticated `vercel curl` (it auto-generates a protection bypass token).
12. **Passport-gated apps defeat every headless path** — bypass tokens carry no Passport identity, so pages 500 even after protection bypass. The only working verification is a real browser with a live Passport session (e.g. the user's Chrome).
13. **Preview URLs live on a custom domain** (`*.dh-vercel.com`), so don't grep for `*.vercel.app`. Get the deployment ID from the PR's status check and read `branchAlias` from the deployments API.
14. **Unauthenticated 401s on webhook routes are usually NOT the webhook's problem** — test the route with authenticated `vercel curl` before blaming Deployment Protection.
15. **vercel[bot] PR comments look like review feedback to automation** — they're deployment-status tables; don't treat them as actionable review comments.

## Vercel Connect GitHub connectors (for eve agents)

16. **The default connector only subscribes to `pull_request` events.** Mention-driven agents need `--trigger-event issue_comment --trigger-event pull_request_review_comment --trigger-event pull_request` at create time — events can't be edited later; you must delete and recreate.
17. **The default trigger branch is literally `production`** — pass `--trigger-branch main`.
18. **The GitHub App slug will not match your connector name** (and changes on every recreate, since old slugs stay reserved). Read the real slug from `gh api orgs/<org>/installations` and set eve's `botName` to it.
19. **Removing a connector needs `--disconnect-all`** if a project is attached, and **orphans its GitHub App** — uninstall the old app manually in GitHub org settings.

## eve framework

20. **`eve add channel/github` may demand an unpublished eve version** — if the registry gate is impossible to satisfy, do the guided steps manually (link, `connect create`, `connect attach --triggers`, hand-write the channel file) and apply rules 16–18.
21. **`eve deploy` hangs forever on a BLOCKED deployment** with zero output — kill it and check the deployments API (see rules 6–7).
22. **eve 0.31.3 swallows a git `safe.directory` failure in sandbox checkouts**, silently degrading PR reviews to diff-only. Upgrade past 0.31.3 when available.

## Local dev & scaffolding

23. **create-next-app aborts if ANY file is in the directory** (even a data CSV). Move data files out, scaffold, move back.
24. **`import 'server-only'` breaks tsx/script execution** outside the Next.js bundler (module not found, then hard throw). Keep it out of modules that migrations/scripts import.
25. **node-postgres returns SQL `DATE` as local-timezone JS `Date` objects** — string-comparison date logic fails silently. Override the pg type parser for `DATE` to return strings.
26. **v0 scaffolds ship `"lint": "eslint ."` without eslint installed** — gate on typecheck + build, or add eslint.
27. **`pnpm install` can be blocked by the org's supply-chain policy (`minimumReleaseAge`)** flagging *pre-existing* lockfile entries (`@vercel/connect`, `ai`, …) as too-recently-published, even when your change didn't touch them. Don't bypass it — ship without the lockfile refresh and re-run `pnpm install` once the entries clear the window (or from an unaffected environment), noting it in the PR.

## Connectors (Jira, BigQuery, Slack)

28. **Jira: use the Atlassian MCP connector, not raw REST.** Call `https://mcp.atlassian.com/mcp` over the MCP protocol; do not integrate against the Jira REST API directly.
29. **BigQuery: the connector is the okta-google one, not a plain Google connector** — DH access to BigQuery goes through Okta.
30. **Slack: create one Slack connector per agent.** A Slack connector is a Slack app. A Slack app is one bot user with one `@handle`. Two agents on one connector share the same `@handle`, so a user cannot address them separately. This is the same constraint as the GitHub App slug in rule 18.

## Onboarding a new user or machine

Minimal setup, in order:

1. Install the Vercel plugin for coding agents — <https://vercel.com/docs/agent-resources/vercel-plugin> (install choices: claude code / global / symlink).
2. Install the Vercel CLI and log in.
3. Log into GitHub in the terminal (`gh auth login`).

The agent can perform steps 1–3 itself when asked.

**One step no agent or CLI can do:** the user's Vercel account must have a GitHub **Login Connection** before `vercel git connect` will work. It's a browser OAuth consent tied to the personal Vercel login — no CLI flag or API exists. Send the user to <https://vercel.com/account/settings/authentication> to sign in with GitHub (the same account authenticated in `gh`), then retry.

**Environment paper cut:** Warp VPN interacts badly with Claude Code and blocks its requests — disconnect it if requests hang.

## How to answer

- **Write in ASD-STE100 Simplified Technical English.** Use short sentences. Use the active voice. Give one instruction per sentence. Many org members do not read English as a first language.
- **Keep answers easy to scan.** Put the result first. Use short lists. Make the key command or file name easy to find. Do not write long blocks of text.

## Full log

See [assets/paper-cuts.md](assets/paper-cuts.md) — every paper cut with exact error text, what we were doing, the workaround, and which are still unresolved (eve sandbox checkout, orphaned GitHub Apps, Workflow resume slugs). Also includes Claude Code tooling friction encountered along the way (browser-pane timeouts, launch.json boilerplate, background-task notification gaps).
