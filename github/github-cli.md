# GitHub CLI Commands for Projects, Issues, and PRs

## gh project

[Manual](https://cli.github.com/manual/gh_project)

### List Projects

Lists projects for an owner.

```bash
gh project list --owner dburriss
```

Output:
```
7	Dotnet 6 to 10 Upgrade	open	PVT_kwHOAE1ih84BJePQ
4	Road to GH Action v1	open	PVT_kwHOAE1ih84ALis8
3	Overboard v1	open	PVT_kwHOAE1ih84AIpJh
2	@dburriss's backlog	open	PVT_kwHOAE1ih84AIpJY
1	Event Feed MVP	open	PVT_kwHOAE1ih84ABL7p
```

### List Project Items

Lists items (issues, PRs, etc.) in a project.

```bash
gh project item-list 7 --owner dburriss --format json
```

Note: For gh project commands, the project number is the first positional argument, followed by flags like --owner.

Output (abbreviated):
```json
{
  "items": [
    {
      "content": {
        "body": "...",
        "number": 1,
        "repository": "dburriss/wye",
        "title": "Dotnet 6 to 10 Upgrade",
        "type": "Issue",
        "url": "https://github.com/dburriss/wye/issues/1"
      },
      "id": "PVTI_lAHOAE1ih84BJePQzgh52sA",
      "status": "In Progress",
      "title": "Dotnet 6 to 10 Upgrade"
    },
    // ... more items
  ],
  "totalCount": 5
}
```

## gh issue

[Manual](https://cli.github.com/manual/gh_issue)

### Edit Issue

Edits one or more issues within the same repository.

```bash
gh issue edit {<numbers> | <urls>} [flags]
```

Editing issues' projects requires authorization with the project scope. To authorize, run `gh auth refresh -s project`.

The `--add-assignee` and `--remove-assignee` flags both support the following special values:

- `@me`: assign or unassign yourself
- `@copilot`: assign or unassign Copilot (not supported on GitHub Enterprise Server)

### Delete Issue

Deletes an issue.

```bash
gh issue delete <number> --repo <repo>
```

## gh pr

[Manual](https://cli.github.com/manual/gh_pr)

### List PRs

Lists pull requests in a repository.

```bash
gh pr list --repo dburriss/wye --json id,number,closingIssuesReferences,assignees,title,state
```

Output:
```json
[
  {
    "assignees": [
      {
        "id": "MDQ6VXNlcjUwNzE0OTU=",
        "login": "dburriss",
        "name": "Devon Burriss",
        "databaseId": 0
      },
      {
        "id": "BOT_kgDOC9w8XQ",
        "login": "Copilot",
        "name": "",
        "databaseId": 0
      }
    ],
    "closingIssuesReferences": [
      {
        "id": "I_kwDOLyxj787bPkXN",
        "number": 1,
        "repository": {
          "id": "R_kgDOLyxj7w",
          "name": "wye",
          "owner": {
            "id": "MDQ6VXNlcjUwNzE0OTU=",
            "login": "dburriss"
          }
        },
        "url": "https://github.com/dburriss/wye/issues/1"
      }
    ],
    "id": "PR_kwDOLyxj7862NCXY",
    "number": 2,
    "state": "OPEN",
    "title": "Upgrade .NET target framework from 8.0 to 10.0"
  }
]
```

### Close PR

Closes a pull request.

```bash
gh pr close <number> --repo <repo>
```