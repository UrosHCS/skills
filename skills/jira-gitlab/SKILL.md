---
name: jira-gitlab
description: Zynca's Jira and GitLab setup, via the `acli` and `glab` CLIs. Use when creating a Jira ticket or working with Jira or GitLab.
---

# Jira & GitLab

Use `acli` for Jira and `glab` for GitLab. Both are already logged in; on an auth error, ask the user to log in.

## Jira

Project `PRO`, board `2`. `acli` calls tickets _work items_ (`acli jira workitem ...`).

- **Current sprint:** `acli jira board list-sprints --id 2 --state active --json`, then take `.sprints[0].id`.
- **Custom fields:** `search` returns only a few fields. Read them with `acli jira workitem view <KEY> --fields '*all' --json`.

### Creating a ticket

Besides summary, type, and description, a new ticket gets exactly these fields:

| Field        | ID                  | Value                                   |
| ------------ | ------------------- | --------------------------------------- |
| T-shirt size | `customfield_10282` | `{"id": "10315"}` (`S (hours)`)         |
| Sprint       | `customfield_10020` | current sprint ID, as a number          |
| Assignee     | `assignee`          | `@me`                                   |
| Reporter     | —                   | leave unset; it defaults to the creator |

Custom fields can be set only at creation, and only through `--from-json`. `acli` has no flags for them, and `edit` can't set them later.

```json
{
  "projectKey": "PRO",
  "type": "Task",
  "summary": "…",
  "description": { "type": "doc", "version": 1, "content": [] },
  "assignee": "@me",
  "additionalAttributes": {
    "customfield_10282": { "id": "10315" },
    "customfield_10020": 889
  }
}
```

Run `acli jira workitem create --from-json ticket.json --json`. The description is Atlassian Document Format; `acli jira workitem create --generate-json` prints the full template.

The ticket is done when `view` on the new key shows T-shirt size `S (hours)`, the current sprint, and the user as assignee.

## GitLab

Branch name convention: `PRO-####-short-text`.

Commit convention: `feat|chore|fix: short description - PRO-####`

MRs target `develop`.
