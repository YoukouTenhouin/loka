# Issue tracker: GitHub

Issues and specs live in GitHub Issues for `YoukouTenhouin/loka`.
Use the `gh` CLI from this clone; it infers the repo from the remote.

## Operations

- Create: `gh issue create --title "..." --body-file <file>`
- Read: `gh issue view <number> --comments`
- List: `gh issue list --state open --json number,title,body,labels`
- Comment: `gh issue comment <number> --body-file <file>`
- Label: `gh issue edit <number> --add-label "<label>"`
- Remove a label: `gh issue edit <number> --remove-label "<label>"`
- Close: `gh issue close <number>`

Write multiline issue bodies and comments to a temporary file and
pass it with `--body-file`.

When a skill says "publish to the issue tracker", create an issue.
When it says "fetch the relevant ticket", read the issue and comments.

## Pull requests as a triage surface

**PRs as a request surface: no.**

## Wayfinding operations

- Map: one issue labelled `wayfinder:map`, with Notes,
  Decisions-so-far, and Fog sections.
- Child tickets: link them as GitHub sub-issues. If unavailable,
  use a task list in the map and `Part of #<map>` in each child.
- Ticket types: `wayfinder:research`, `wayfinder:prototype`,
  `wayfinder:grilling`, or `wayfinder:task`.
- Blocking: use native GitHub issue dependencies. If unavailable,
  record `Blocked by: #<number>` in the child. A ticket is
  unblocked when all blockers are closed.
- Frontier: choose the first open, unassigned, unblocked child
  in map order.
- Claim: `gh issue edit <number> --add-assignee @me`.
- Resolve: comment with the result, close the ticket, and add
  a brief finding and link to the map's Decisions-so-far.
