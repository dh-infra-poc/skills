---
name: dh-skills-stay-current
description: Keep the Delivery Hero skill pack up to date, and send new learnings back to it. The pack changes many times per day during the POC — an old copy gives wrong commands and repeats a solved paper cut. Use at the start of a DH session, before you set up or deploy a DH project, when a rule in a DH skill looks wrong or is missing, when a command fails in a way no rule covers, or when the user reports a new paper cut.
---

# Keep the DH skill pack current

The pack is `dh-infra-poc/skills`. It changes many times per day during the POC. An old copy gives you wrong commands. It also makes you repeat a paper cut that somebody already solved.

This skill has two jobs. **Pull the updates. Push the new learnings back.**

## 1. Update the pack

Run this at the start of a DH session, and before you set up or deploy a DH project.

```bash
npx skills update -y
```

For the DH pack only:

```bash
npx skills update dh-vercel-setup-idiosyncracies dh-skills-stay-current -y
```

To see what you have installed:

```bash
npx skills list
```

If a skill is not installed yet:

```bash
npx skills add dh-infra-poc/skills --all
```

**Note:** `npx skills add` writes `.agents/`, `.claude/`, and `skills-lock.json` into the current directory. Add them to `.gitignore`, or run the command outside the repo.

## 2. Check for a change you do not have

The updated file is the source of truth. Read the rules again after an update. Do not answer from memory.

To see the last change to the pack:

```bash
gh api repos/dh-infra-poc/skills/commits/main --jq '.commit.message, .commit.committer.date'
```

## 3. Send the new learning back

Do this every time you lose time to a problem that no rule covers. The next person then does not lose the same time.

1. Clone or open the pack.

   ```bash
   gh repo clone dh-infra-poc/skills
   ```

2. Create a branch. Do not push to main.
3. Add the distilled rule to the `SKILL.md` of the correct skill. Keep it to one or two sentences. Give the exact command.
4. Add the full entry to `assets/paper-cuts.md`: **Context**, **Symptom** (with the exact error text), **Workaround**.
5. Open a PR. Say what you lost time to.

**What is worth a PR:** a command that fails, a flag that does not exist, a default that surprises you, a step that needs a human, a silent failure.

**What is not:** a one-time typo, or a problem in the app code only.

## Rules for you

- **Update first, then answer.** A wrong command from an old copy costs the user a round-trip.
- **Do not invent flags.** If the pack has no rule, check `--help` and then write the rule.
- **Run the update yourself.** Do not ask the user to run it.
