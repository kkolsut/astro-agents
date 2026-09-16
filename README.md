# astro-agents

Claude Code subagents for observational astronomy, packaged as a plugin so the
same eleven agents work in every project instead of being copied between them.

The routing table and what each agent does is in
[`plugins/astro/AGENTS.md`](plugins/astro/AGENTS.md).

This repository is both the **plugin** (`plugins/astro/`) and the
**marketplace** that serves it (`.claude-plugin/marketplace.json`). Nothing is
published to Anthropic and nothing leaves your machine or your git host — a
marketplace is just a git repository with a JSON catalog in it.

## Install

Point Claude Code at wherever this repository lives, then install the plugin:

```bash
/plugin marketplace add https://git.example.internal/you/astro-agents.git
/plugin install astro@astro-agents
```

Two things that bite on a self-hosted host (Gitea, Forgejo, self-managed
GitLab):

- **The URL must end in `.git`.** Without the suffix Claude Code treats it as a
  direct link to a hosted `marketplace.json` and never clones the repository.
  GitHub and gitlab.com are the only hosts where the suffix is optional.
- **Prefer SSH for background auto-updates.** Background refresh disables git
  credential helpers for HTTPS, so a private repository over HTTPS updates
  interactively but not in the background. With a key in `ssh-agent`,
  `git@git.example.internal:you/astro-agents.git` works for both.

To pin a version rather than track the default branch, append a ref:

```bash
/plugin marketplace add https://git.example.internal/you/astro-agents.git#v1.0.0
```

### Installing per project, so the project records it

`--scope project` writes the marketplace and the plugin into the project's
`.claude/settings.json`, which is committed:

```bash
cd ~/projects/my-paper
claude plugin marketplace add https://git.example.internal/you/astro-agents.git --scope project
claude plugin install astro@astro-agents --scope project
```

```json
{
  "extraKnownMarketplaces": {
    "astro-agents": {
      "source": { "source": "git", "url": "https://git.example.internal/you/astro-agents.git" }
    }
  },
  "enabledPlugins": { "astro@astro-agents": true }
}
```

That is what keeps provenance intact. The project's own history now says which
agents it used, and pinning the marketplace to a tag makes it say which
version. A collaborator who clones the project and trusts the folder gets the
marketplace registered automatically, and Claude Code tells them the one
command to run to install the plugin itself.

`--scope user` installs it once for every project on the machine instead, at
the cost of the project no longer recording anything.

## Precedence — the thing that will waste your afternoon

Plugin agents are the **lowest** priority source. Order is: managed settings,
`--agents` flag, project `.claude/agents/`, user `~/.claude/agents/`, then
plugins.

So a leftover copy of `lit-scout.md` in a project's `.claude/agents/` silently
wins over this plugin's, and you can edit the plugin all day with no effect.
When adopting the plugin in a project that already had the agents, **delete the
old `.claude/agents/` copies.** Check what actually loaded with `/context`
under Custom Agents, where plugin agents appear scoped as `astro:lit-scout`.

Project-specific agents still belong in `.claude/agents/` — that directory is
scanned recursively and a name that does not collide is simply added to the
set.

## Developing

Edit an agent, then load the working copy without installing it:

```bash
claude --plugin-dir ./plugins/astro
```

`/reload-plugins` picks up changes in an open session. Validate before
committing, and confirm what the plugin actually contributes:

```bash
claude plugin validate .                    # the marketplace
claude plugin validate ./plugins/astro      # the plugin
claude plugin details astro@astro-agents    # agent inventory and token cost
```

`claude plugin details` also prints the always-on context cost. Every agent's
`description` is loaded into every session so the router can match on it — at
eleven agents that is roughly 1.7k tokens before anything runs, which is the
budget to keep in mind when adding a twelfth.

### Versioning

Bump `version` in `plugins/astro/.claude-plugin/plugin.json` when you want
installs to pick the change up, and tag the commit so projects can pin to it.
Agents are prompts, so a change to one can change a result: a project that
pins a tag can say which rules refereed its numbers, and one that tracks the
branch cannot.

## What makes these prompts work

Each file is frontmatter (`name`, `description`, `tools`, `model`) plus a
system prompt. Notes for editing them:

- **The `description` is what routing matches on.** Concrete triggers ("the
  x-axis is time or phase") beat topic labels, and an explicit "NOT for…"
  keeps an agent out of neighbouring work.
- **Write the subfield's real failure modes,** not generic best practice.
  "Overlay the window function" is worth more than "be careful with
  periodograms".
- **Say what the deliverable is,** including the diagnostics, so results come
  back checkable.
- **Restrict tools when the role calls for it.** `results-referee` is read-only
  because a referee that edits is no longer an independent check.
- **Shared conventions belong in the project's `CLAUDE.md`,** not repeated in
  every prompt. The companion `astro-template` repository carries the ones
  these agents assume: explicit units, frames, time systems, and provenance.

Plugin agents cannot use the `hooks`, `mcpServers` or `permissionMode`
frontmatter fields. None of these do; if one ever needs them it has to be
copied into a project's `.claude/agents/` instead.
