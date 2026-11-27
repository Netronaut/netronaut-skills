# netronaut-skills

Skills for AI agents used in development at [Netronaut](https://netronaut.de).

| Skill                                                        | What it does                                                                                                                                                                                                                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [`github-project-flow`](skills/github-project-flow/SKILL.md) | Runs project work through GitHub epics, stories as sub-issues and a project board: drafting issues, agreeing on the text, board status, starting an epic or a story, review teams, PRs.                                                                |
| [`code-structure`](skills/code-structure/SKILL.md)           | Decides where a file goes and what it is called in a TypeScript monorepo with Next.js: layers, domain folders, zod schema naming, when an abstraction is allowed, reuse, style. Includes a repeatable audit that lists findings without changing code. |
| [`sops-handoff`](skills/sops-handoff/SKILL.md)               | Keeps sops- and age-encrypted secrets out of the agent's context. The agent prepares the change and hands the user a template to run.                                                                                                                  |

All skills are written in English. The issue headings of `github-project-flow` and the hand-off template of `sops-handoff` default to German; a project's `CLAUDE.md` overrides these defaults, but not the rules about what the agent may do with secrets. The paths and tool names in `code-structure` are defaults that a project's `AGENTS.md` or `CLAUDE.md` overrides.

`code-structure` draws on Robert C. Martin's _Clean Code_ for naming, vertical formatting and comments, and follows John Ousterhout's _A Philosophy of Software Design_ and Sandi Metz's "The Wrong Abstraction" where they disagree with it: no extraction for its own sake, no abstraction before the second implementation.

## Install

### Claude Code

```
/plugin marketplace add Netronaut/netronaut-skills
/plugin install netronaut-skills@netronaut-skills
```

### Other agents

```sh
npx skills add Netronaut/netronaut-skills
```

The skills are written for Claude Code. They name its question tool (`AskUserQuestion`), its sandbox switch, its `!` prompt prefix and `CLAUDE.md`; another agent has to map these to its own equivalents.

### Working copy

To edit the skills while you use them, clone the repository and link each skill into your skills directory:

```sh
git clone https://github.com/Netronaut/netronaut-skills.git
ln -s "$PWD/netronaut-skills/skills/github-project-flow" ~/.claude/skills/github-project-flow
ln -s "$PWD/netronaut-skills/skills/sops-handoff" ~/.claude/skills/sops-handoff
ln -s "$PWD/netronaut-skills/skills/code-structure" ~/.claude/skills/code-structure
```

Do not combine this with the plugin install, or each skill is loaded twice.

## Contributing

Each skill lives in `skills/<name>/SKILL.md`. Commits follow [Conventional Commits](https://www.conventionalcommits.org) with the skill as scope, such as `feat(github-project-flow): add rules for starting an epic`.

## License

Copyright © 2026 Netronaut. Licensed under the [EUPL-1.2](LICENSE).
