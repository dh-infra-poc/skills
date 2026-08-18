---
name: dh-vercel-cli
description: Use the Vercel CLI on citizen developer machines for local development, environment variables, and deployment debugging. Use when running an app locally, pulling env vars, linking a folder to its Vercel project, or checking why a deployment failed. Never use it to deploy.
---

# Vercel CLI for citizen developers

The Vercel CLI is installed for local development and debugging only.
Deployments always come from shipping code through the citizen-dev tools
(push or create_app). The platform rejects every other deployment source.

## Hard rules

- NEVER run `vercel deploy`, `vercel --prod`, `vercel redeploy`, or `vercel rollback`. They will be rejected, and trying them confuses the user.
- NEVER change settings: no `vercel env add`, `vercel domains`, `vercel project`, `vercel dns`, or `vercel teams` mutations. Ask an administrator instead.
- Never show raw tokens or env var values to the user.

## Common tasks

First time in an app folder (folder name usually matches the project):

```sh
vercel link --yes --project <app-name> --scope dh-infra
```

Local development (env pull brings connector credentials like
VERCEL_OIDC_TOKEN, so integrations work locally):

```sh
vercel env pull .env.local
pnpm dev
```

Do not use `vercel dev` for Next.js apps; use the framework dev server
after pulling env vars.

## Verify a shipped change

1. Check the deployment state with the citizen-dev `status` tool.
2. If the build failed, read the build logs:

```sh
vercel ls <app-name> --scope dh-infra
vercel inspect <deployment-url> --logs --scope dh-infra
```

3. Fix the problem locally, then ship again through the citizen-dev tools.
4. Runtime errors after a successful build: `vercel logs <deployment-url>`.

## Idiosyncrasies

- Always pass `--scope dh-infra` when a command reports that a project is not found.
- Describe results to the user in plain language. Do not use terms like deployment source, scope, or CLI in your summary.
