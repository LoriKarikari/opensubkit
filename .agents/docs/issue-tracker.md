# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues on `LoriKarikari/opensubkit`. Use the `gh` CLI for all operations. It infers the repo from `git remote -v`.

## Conventions

- **Create**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read**: `gh issue view <number> --comments`.
- **List**: `gh issue list --state open --json number,title,body,labels,comments` with `--label` and `--state` filters.
- **Comment**: `gh issue comment <number> --body "..."`.
- **Labels**: `gh issue edit <number> --add-label "..."` or `--remove-label "..."`.
- **Close**: `gh issue close <number> --comment "..."`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either. Resolve with `gh pr view 42` and fall back to `gh issue view 42`.

## Pull requests as a triage surface

**PRs as a request surface: no.** Set to `yes` to run external PRs through the same triage labels as issues, using the `gh pr` equivalents. When `yes`, triage only PRs whose `authorAssociation` is `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE`.

## Planning maps

A planning map is one issue with child issues as its tickets. These are the map and ticket operations for this repo.

- **Map**: one issue labelled `plan:map`, holding the Destination, Notes, Decisions so far, Not yet specified, and Out of scope sections.
- **Ticket**: a GitHub sub-issue of the map (`gh api` on the sub-issues endpoint). Its type label is one of `plan:research`, `plan:prototype`, `plan:discussion`, `plan:task`. A discussion ticket is resolved in conversation with the human.
- **Blocking**: GitHub's native issue dependencies. Add an edge with `gh api --method POST repos/LoriKarikari/opensubkit/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`. The blocker id is the numeric database id from `gh api repos/LoriKarikari/opensubkit/issues/<n> --jq .id`, not the `#number` or `node_id`. A ticket is unblocked when every blocker is closed.
- **Frontier**: the map's open sub-issues with no open blocker (`issue_dependencies_summary.blocked_by == 0`) and no assignee. First in map order wins.
- **Claim**: `gh issue edit <n> --add-assignee @me`, before any other work on the ticket.
- **Resolve**: `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`, then append a one-line gist and link to the map's Decisions so far.
