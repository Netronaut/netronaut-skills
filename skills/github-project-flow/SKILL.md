---
name: github-project-flow
description: Use when planning or running project work through GitHub epics, issues, sub-issues and project boards — drafting or editing an epic or story, creating issues, setting issue types or board status, starting or refining the next epic, starting work on a story (branch, "In Progress"), preparing a story with a review team, or writing the PR for a story.
---

# github-project-flow

## Overview

Work is organised as epics with stories as sub-issues on a GitHub project board. The user decides every issue text; Claude drafts, asks, then writes.

A project `CLAUDE.md` overrides this skill: issue language, board name and number, status names, allowed commands. Read it first. Headings below are the German defaults; use the project language's equivalents if it differs.

## Structure

|          | Epic                                                      | Story                                                                |
| -------- | --------------------------------------------------------- | -------------------------------------------------------------------- |
| Type     | Epic                                                      | Task or Feature; Bug for defects                                     |
| Sections | `## Ziel`, `## Entscheidungen`, `## Nicht Teil des Epics` | `## Ziel`, `## Aufgaben`, `**Fertig wenn:**`, optional `## Hinweise` |
| Relation | umbrella                                                  | sub-issue of its epic                                                |

- Short titles, no phase prefix, no labels.
- A story's `## Ziel` names the outcome and why it matters ("…, damit …"), and the beneficiary when it is not the operator. An epic's `## Ziel` says what changes for whom, not what gets built.
- `## Aufgaben`: a checklist of milestones only. One item per milestone, high-level prose, no file paths, flags or commands.
- `## Hinweise`: technical details the reader must know and cannot infer — pitfalls, non-obvious reasons. Bold lead-in per point, absolute minimum.
- `**Fertig wenn:**` is a verifiable criterion, not a restatement of the goal. For safeguards, alerts and access rules it includes the failure case. Several conditions go in a list, each as trigger → result.
- Split a story with more than five `Aufgaben`, or one whose items would each need their own `Fertig wenn`. A phase is an epic or a line in an epic, never a story.
- An open choice ("pick a tool") is not a task. Decide it before the story goes to Todo and record it under the epic's `Entscheidungen`; if it needs research, a time-boxed spike delivers that entry.
- An epic fits on one screen. Concept papers and roadmaps go to `docs/` or an ADR, and the epic links them.
- Bugs in running services do not wait under a roadmap epic; they go to the top of Todo.
- References to another repo are written `repo#n`.
- Dependencies between issues are set as GitHub relationships (blocked by / blocking), never written into the issue body.
- One-off procedures and runbooks go into the PR description, never into the issue. `docs/` holds only what stays valid permanently, written as readable prose, not dense lists.
- When a new finding contradicts an epic's `Entscheidungen` or `Nicht Teil` list, say so and update the epic in the same step as the new issue.
- Name things from the user's perspective ("on-demand", "backup-cleanup"), not after the implementation.
- Charts and tables for an epic, such as impact/effort, go into the epic body, not a comment. Label items with their issue numbers (`#50`), never invented ids.
- An issue is closed only when each condition of `Fertig wenn` is shown in the PR or a comment, or split off into a linked follow-up issue. Unticked `Aufgaben` are explained in the closing comment; tasks never carried out stay unticked. An issue that resolved itself is closed with a comment explaining why.

## Approval

Every write has exactly one level. An action not listed takes the level of the closest row; if none is close, it is Go.

| Level          | Meaning                                                                        |
| -------------- | ------------------------------------------------------------------------------ |
| **Text**       | The user approves the exact text in a preview. Only "Passt" unlocks the write. |
| **Go**         | A yes/no question in the question UI. No text to approve.                      |
| **Free**       | No question. Do it and report it in one line.                                  |
| **On request** | Only when the user asks for it explicitly.                                     |

| Action                                           | Level      | When / note                                      |
| ------------------------------------------------ | ---------- | ------------------------------------------------ |
| Create an issue (title, body)                    | Text       |                                                  |
| Edit an issue title or body                      | Text       | Including folding a comment into the body        |
| Post a comment (closing comment, reply)          | Text       |                                                  |
| Close an issue or epic manually                  | Text       | The closing comment is the approval              |
| Board description (focus line)                   | Text       |                                                  |
| PR description longer than one line plus runbook | Text       | Give the reasons first                           |
| PR description, one line plus runbook            | Free       | Covered by the request to open the PR            |
| Type, parent epic, initial status Backlog        | Free       | Part of creating an issue that was just approved |
| Assignee `@me`                                   | Free       | When picking up any issue                        |
| Story and its epic → "In Progress"               | Free       | On "next", or when the user names the story      |
| Tick finished `Aufgaben`                         | Free       |                                                  |
| Backlog → Todo                                   | Free       | Only when every readiness rule holds             |
| Blocked-by / blocking relationships              | Free       | Report the issue numbers                         |
| Create the story branch                          | Free       | Part of starting a story                         |
| Story from Backlog when Todo is empty            | Go         |                                                  |
| Story of a third epic "In Progress"              | Go         |                                                  |
| Third story "In Progress"                        | Go         |                                                  |
| Sub-issue order and board order                  | Go         | Propose the order once as a list                 |
| Move a story to another epic                     | Go         |                                                  |
| Close or fold stale items (weekly review)        | Go         | One list, one question                           |
| Commit, push, open a PR                          | On request |                                                  |
| Delete a branch                                  | On request |                                                  |

Asking for Text:

- Use the AskUserQuestion tool: one question per issue, the full text as `preview`, options "Passt" / "Ändern".
- `preview` is a field of an option, not of the question; a question-level `preview` is dropped silently and the user sees nothing. Put the full text on "Passt" and a short summary on "Ändern", so a preview shows whichever option is focused.
- After "Ändern", revise and ask again.

## Board and issue mechanics

Run `gh` inside the sandbox. Only if a call fails with a TLS or x509 certificate error (seen in the macOS sandbox), repeat that call with `dangerouslyDisableSandbox: true`.

New story: one call creates it with type, epic and dependencies, then the board status is set.

```bash
url=$(gh issue create --title "$title" --body-file "$f" --type Task --parent "$epic" --blocked-by "$blockers")
item=$(gh project item-add N --owner ORG --url "$url" --format json -q .id)
gh project item-edit --project-id "$pid" --id "$item" --field-id "$status_field" --single-select-option-id "$opt"
```

- These flags need a current `gh`; on "unknown flag", upgrade `gh` instead of falling back to GraphQL.
- Issue types exist only in organisations. In a user-owned repo `--type` fails; leave it out and say so.
- Existing issues: `gh issue edit N --type Bug`, `--parent E`, `--add-sub-issue A,B`, `--add-blocked-by A,B`. Check with `gh issue view N --json issueType,parent,subIssues,blockedBy`.
- Ids: `gh project view N --owner ORG --format json -q .id`; status field and options via `gh project field-list N --owner ORG --format json` filtered on `name=="Status"`; item ids via `gh project item-list N --owner ORG --format json --limit 200`, filtered on `.content.number` **and** `.content.repository` (boards span repos, numbers collide).
- `item-list` returns 30 items unless `--limit` is given; always pass it, and raise it when the result has exactly that many items.
- If the board auto-adds new issues, `item-add` fails with "Content already exists"; look the item up instead and set its status.
- Sub-issue order has no flag. Node ids via `gh issue view N --json id -q .id`, then `gh api graphql -f query='mutation($p:ID!,$c:ID!,$a:ID!){reprioritizeSubIssue(input:{issueId:$p,subIssueId:$c,afterId:$a}){issue{number}}}' -f p="$epic_id" -f c="$story_id" -f a="$after_id"`; verify with `gh issue view E --json subIssues`.
- Edit bodies only via `--body-file` from a scratchpad file. For partial edits, a script replaces exact anchors and asserts each one matched. Chain with `&&` so a failed assert stops `gh issue edit` — a newline instead of `&&` once let gh overwrite the body anyway.
- Re-read the changed section after each write (`gh issue view N --json body -q .body | sed -n '/## Aufgaben/,/Fertig/p'`).

## Focus and readiness

- The board description holds one line of focus: what the coming weeks pay into.
- At most two epics are "In Progress"; the epic status marks focus.
- At most two stories are "In Progress" at a time.
- Detailed issues only for the next four to six weeks. Further ideas stay one line in the epic or the roadmap doc until they come close.
- Todo holds only ready stories: template complete, no open choice or `@todo`, dependencies closed, fits in one PR. Anything else stays in Backlog.
- Weekly review, on request: check Todo against these rules, order Backlog by epic, attach orphans, close or fold into one line what has not moved for 30 days.

## Starting an epic

An epic starts when the first of its stories is to be picked up, or when the user names it. Which epic is next is the user's call; ask if the board order does not say.

1. Close out the previous epic: every sub-issue closed or moved to another epic, its `Entscheidungen` match what was built, then close it with a comment naming what was split off. An epic with open stories stays "In Progress" and counts against the limit of two.
2. Read the new epic in full: body, all comments, its sub-issues and their comments (`gh issue view N --comments`). Comments from the product owner are requirements, not background.
3. Collect what has changed since the epic was written: decisions and follow-ups from the previous epic, comments, the current state of repo and docs.
4. Refine the epic first. Present one summary of the findings and the proposed new epic text. A comment that is folded into the body gets a reply saying so; a comment that contradicts the body is an open choice for the user.
5. Then the stories: draft the missing ones, revise the existing ones, only as far as the next four to six weeks reach. The rest stays one line in the epic.
6. Set sub-issue order and blocked-by relationships, move ready stories to Todo, the others stay in Backlog.
7. The epic goes to "In Progress" with its first story, not before.

A review team can prepare a larger epic the same way as a story; its findings feed steps 4 and 5.

## Starting a story

Always follow the priority. "The next story" is the top Todo item of the board's current order (`gh project item-list N --owner ORG --format json --limit 200`), checked against the epic's sub-issue order. Re-read both every time; the user reorders between turns, so an order from earlier in the session is stale.

On "next", take the top Todo item. The cases that need a go first are in the approval matrix.

1. Close out the previous story first: tick its finished `Aufgaben` items, even when its PR already closed the issue.
2. Read the story's comments; a comment that changes scope goes into the body first.
3. Set the story and its epic to "In Progress" on the board, and make sure the current user is the assignee (`gh issue edit N --add-assignee @me`). This holds for any issue picked up for work, not just stories.
4. Branch `<issue>-<slug>` from an up-to-date `main`. The slug is English, even for a German issue title.
5. Tick finished `Aufgaben` items as milestones land.

## Preparing a story with a review team

On request, or when worth it before a larger story, send a small parallel team of expert agents, each briefed with goal, files and narrow scope, for example:

| Agent         | Looks at                                             |
| ------------- | ---------------------------------------------------- |
| Ops readiness | is the running system ready for the story            |
| Repo review   | code, config and docs the story touches              |
| Plan critique | the story's `Aufgaben` against current upstream docs |

Wait for all, then consolidate one ranked summary:

- **(A) Blockers** before the story → a small preparation PR.
- **(B) In the story** → proposed additions to the story text.
- **(C) Follow-ups** → proposed new issues.
- End with explicit decisions for the user, asked via the question UI.

Verify alarming findings against evidence (git history, existing config, the live value) before escalating. An agent's "urgent, probably broken" has turned out wrong before.

## Commits and PRs

- Conventional commits in the repo's style, imperative, no AI attribution or co-author lines.
- PR description as short as possible: ideally one line plus the runbook steps if any.

## Red flags: STOP

| Thought                                                        | Reality                                                                         |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| "The text is obvious, I'll create the issue and show it after" | Draft in the question UI first. Always.                                         |
| "The question text explains it, no preview needed"             | Asking about GitHub text without a preview on the option is not asking. Add it. |
| "I'll add the commands to `Aufgaben` so it's complete"         | Milestones only. Minimal details go to `## Hinweise`, steps to the PR.          |
| "This runbook is useful, I'll put it in `docs/`"               | One-off → PR description. `docs/` is permanent only.                            |
| "The new issue doesn't really touch the epic"                  | If it contradicts `Entscheidungen` or `Nicht Teil`, update the epic too.        |
| "Agent says it's urgent, I'll alarm the user"                  | Check the evidence first, then report.                                          |
| "I'll explain the change properly in the PR"                   | One line. More needs the user's approval.                                       |
| "Branch is merged, I'll clean it up"                           | Not unasked.                                                                    |
| "The PR closed the last issue, on to the next"                 | Tick its `Aufgaben` first, then start the next story.                           |
| "I know what comes next, we sorted it earlier"                 | Re-read the board and epic order now. Priority is what they say today.          |
| "No Todo left, I'll start the top Backlog item"                | Ask first. Only Todo items are picked without asking.                           |
| "The story says 'pick a tool', I'll pick one"                  | Open choice: ask, record it in `Entscheidungen`.                                |
| "PR merged, the issue is done"                                 | Show each `Fertig wenn` condition or split it off first.                        |
| "I'll note `Abhängig von #…` at the end of the body"           | Set a blocked-by relationship instead.                                          |
| "This epic needs the whole concept in it"                      | One screen. The concept goes to `docs/` or an ADR.                              |
| "The epic body is the spec, I'll start with its first story"   | Read the comments and the previous epic's outcome first. The body may be stale. |
| "The last story is merged, the epic is done"                   | Close the epic explicitly: sub-issues, `Entscheidungen`, closing comment.       |
