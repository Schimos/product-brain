---
name: task-archive
description: "Use when finished task folders clutter <tasks.dir>/ in a Product Brain hub that keeps task folders, or a task's description no longer says where its ticket stands: the user says 'archive done tasks', 'clean up tasks/', 'sync task status with Jira', 'which tasks are done', 'task-archive', or a ticket was closed or reopened. Reads every task's ticket status from Jira, GitLab or GitHub, records it in each description.md, moves the tasks whose ticket is done to <tasks.dir>/archive/<KEY>/ (and a reopened one back), and rewrites every hub link to a moved folder."
argument-hint: "[KEY ...] (default: every task folder, open and archived)"
allowed-tools: [Bash, Read, Edit, Glob, Grep]
---

# task-archive

## User Input

```text
$ARGUMENTS
```

## What this does

Keeps `<dir>/` to the work that is still open, without losing what is finished. For each task
folder (or only the keys given), it reads the ticket's status in the tracker, then:

| Ticket | Folder now | What happens |
|---|---|---|
| done | `<dir>/<KEY>/` | moved to `<dir>/<archive>/<KEY>/`; description Status `Done` or `Absorbed into <spec>` |
| done | archived | nothing but the Ticket status line |
| not done | `<dir>/<KEY>/` | nothing but the Ticket status line |
| not done (reopened) | archived | moved back to `<dir>/<KEY>/`; description Status `Reopened` |
| not found | either | nothing; listed for the user |

The archive is a place, not a deletion: the files keep their history, `task-facts resolve <KEY>`
and every other task skill still find an archived task, and every link in the hub to a moved
folder is rewritten in the same change.

It never changes a ticket, never deletes a folder, and commits only after the user says so.

`${CLAUDE_SKILL_DIR}` is this skill's base directory (shown when the skill loads). The shared
script is `${CLAUDE_SKILL_DIR}/../../scripts/task-facts.py`; call it `task-facts` below. Tracker
details: [../../references/trackers.md](../../references/trackers.md).

## 1. Preconditions

- **Settings.** `task-facts config` from the hub root. `tracker` must be set; if it is null, stop
  and show the `tasks.tracker` example from trackers.md. Note `archive` (the archive folder's
  name), `specs_dir` and `tracker.done`.
- **A clean start.** `git status --short -- <dir> <specs_dir>`. Uncommitted changes inside a task
  folder may be another session's work: list them and ask before moving that folder. Leave every
  other uncommitted file alone.
- **Connection check** (trackers.md). If it fails, stop with the hint there. Never archive from a
  guess about a ticket's status.

## 2. Read every task and its ticket

1. `task-facts tasks` lists every ticketed folder, open and archived, with its description Status,
   Ticket status line, whether `report.md` exists, `absorbed_into`, and `mentioned_in` (the specs
   whose files name the key).
2. Read the tickets in as few calls as the tracker allows (trackers.md, "Reading statuses"): the
   status name, and for Jira its status category.
3. **Done** is the rule in trackers.md: a status named in `tracker.done` when it is set, else Jira's
   `Done` status category, or a closed GitLab or GitHub issue. A resolution such as "Won't do" is
   still done: the folder is archived and the table says so.

## 3. Where each done task's account is

The constitution's record of a finished task is its `report.md`. A task folded into a feature spec
has none: the spec carries its account. For each done task **without** a `report.md`:

- `absorbed_into` already set: keep it.
- `mentioned_in` names specs: propose the one that delivered it. A mention is a hint, not proof (a
  spec may only cite the task), so the user confirms or corrects it.
- Neither: it archives as `Done`, and the table marks it "no report".

## 4. Show the plan and confirm

One table, every task, so the user confirms the whole run once:

| Key | Ticket | Description Status now | Action | Account |
|---|---|---|---|---|
| PROJ-12 | Done | Ready for plan | archive → `Done` | report.md |
| PROJ-14 | Done | Ready for plan | archive → `Absorbed into specs/033-…` | spec (proposed from mentions) |
| PROJ-15 | In Progress | Ready for plan | status line only | – |

Under it: tickets that were not found, folders with uncommitted changes, and done tasks with no
account. Ask the user to confirm, change the spec of any absorbed row, or skip keys. Move nothing
before they answer.

## 5. Apply

For each confirmed row, one at a time:

- archive: `task-facts archive <KEY> "<ticket status>" [--absorbed-into <spec path>]`
- restore: `task-facts restore <KEY> "<ticket status>"`
- status line only: `task-facts status <KEY> "<ticket status>"`

`archive` and `restore` move the folder, set the description's Status, `Ticket status` and
`Last updated` rows, and rewrite every link to the folder across the hub's text files: relative
links into it, the relative links inside it (it now sits one level deeper or shallower), and plain
mentions of its path. Each prints what it changed. If one stops (exit 2), report it and go on with
the rest.

After `restore`, tell the user to run `/task-description <KEY>` in update mode: `Reopened` is a
marker, and the description's real Status has to be worked out again.

## 6. Check, commit, hand-off

1. **Links.** Grep the hub's text files for `<dir>/<KEY>` paths that no longer exist: there should
   be none. Links outside the hub (ticket bodies, merge requests, chat) still point at the old
   path; list the tickets of the moved tasks and say so once. Don't edit tickets.
2. **Layout.** Run the hub's layout check (`pb check`) if it has one.
3. **Show** the moved folders, the files changed outside them, and the done tasks with no account.
4. **Commit only after the user says so**: stage only the task folders and the files the script
   listed (`git add -A -- <each path>`), one commit for the run, for example
   `docs(tasks): archive 12 done tasks and record ticket status`. Push only when asked. On a
   protected branch, push a branch and open a merge request.

## Do not

- Move a task whose ticket you have not read in this run, or whose status you inferred.
- Delete a task folder, or archive a draft (a draft has no ticket; it stays in `<dir>/<drafts>/`).
- Change a ticket's status, text or links, or comment on it.
- Edit a description beyond the Status, Ticket status and Last updated rows.
- Set `Absorbed into` without the user confirming the spec.
