## Source: references/skills/github-automation/SKILL.md

---
name: github-automation
description: "Automate GitHub repositories, issues, pull requests, branches, CI/CD, and permissions via Rube MCP (Composio). Manage code workflows, review PRs, search code, and handle deployments programmatically."
risk: unknown
source: community
date_added: "2026-02-27"
---

# GitHub Automation via Rube MCP

Automate GitHub repository management, issue tracking, pull request workflows, branch operations, and CI/CD through Composio's GitHub toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active GitHub connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `github`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `github`
3. If connection is not ACTIVE, follow the returned auth link to complete GitHub OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Issues

**When to use**: User wants to create, list, or manage GitHub issues

**Tool sequence**:
1. `GITHUB_LIST_REPOSITORIES_FOR_THE_AUTHENTICATED_USER` - Find target repo if unknown [Prerequisite]
2. `GITHUB_LIST_REPOSITORY_ISSUES` - List existing issues (includes PRs) [Required]
3. `GITHUB_CREATE_AN_ISSUE` - Create a new issue [Required]
4. `GITHUB_CREATE_AN_ISSUE_COMMENT` - Add comments to an issue [Optional]
5. `GITHUB_SEARCH_ISSUES_AND_PULL_REQUESTS` - Search across repos by keyword [Optional]

**Key parameters**:
- `owner`: Repository owner (username or org), case-insensitive
- `repo`: Repository name without .git extension
- `title`: Issue title (required for creation)
- `body`: Issue description (supports Markdown)
- `labels`: Array of label names
- `assignees`: Array of GitHub usernames
- `state`: 'open', 'closed', or 'all' for filtering

**Pitfalls**:
- `GITHUB_LIST_REPOSITORY_ISSUES` returns both issues AND pull requests; check `pull_request` field to distinguish
- Only users with push access can set assignees, labels, and milestones; they are silently dropped otherwise
- Pagination: `per_page` max 100; iterate pages until empty

### 2. Manage Pull Requests

**When to use**: User wants to create, review, or merge pull requests

**Tool sequence**:
1. `GITHUB_FIND_PULL_REQUESTS` - Search and filter PRs [Required]
2. `GITHUB_GET_A_PULL_REQUEST` - Get detailed PR info including mergeable status [Required]
3. `GITHUB_LIST_PULL_REQUESTS_FILES` - Review changed files [Optional]
4. `GITHUB_CREATE_A_PULL_REQUEST` - Create a new PR [Required]
5. `GITHUB_CREATE_AN_ISSUE_COMMENT` - Post review comments [Optional]
6. `GITHUB_LIST_CHECK_RUNS_FOR_A_REF` - Verify CI status before merge [Optional]
7. `GITHUB_MERGE_A_PULL_REQUEST` - Merge after explicit user approval [Required]

**Key parameters**:
- `head`: Source branch with changes (must exist; for cross-repo: 'username:branch')
- `base`: Target branch to merge into (e.g., 'main')
- `title`: PR title (required unless `issue` number provided)
- `merge_method`: 'merge', 'squash', or 'rebase'
- `state`: 'open', 'closed', or 'all'

**Pitfalls**:
- `GITHUB_CREATE_A_PULL_REQUEST` fails with 422 if base/head are invalid, identical, or already merged
- `GITHUB_MERGE_A_PULL_REQUEST` can be rejected if PR is draft, closed, or branch protection applies
- Always verify mergeable status with `GITHUB_GET_A_PULL_REQUEST` immediately before merging
- Require explicit user confirmation before calling MERGE

### 3. Manage Repositories and Branches

**When to use**: User wants to create repos, manage branches, or update repo settings

**Tool sequence**:
1. `GITHUB_LIST_REPOSITORIES_FOR_THE_AUTHENTICATED_USER` - List user's repos [Required]
2. `GITHUB_GET_A_REPOSITORY` - Get detailed repo info [Optional]
3. `GITHUB_CREATE_A_REPOSITORY_FOR_THE_AUTHENTICATED_USER` - Create personal repo [Required]
4. `GITHUB_CREATE_AN_ORGANIZATION_REPOSITORY` - Create org repo [Alternative]
5. `GITHUB_LIST_BRANCHES` - List branches [Required]
6. `GITHUB_CREATE_A_REFERENCE` - Create new branch from SHA [Required]
7. `GITHUB_UPDATE_A_REPOSITORY` - Update repo settings [Optional]

**Key parameters**:
- `name`: Repository name
- `private`: Boolean for visibility
- `ref`: Full reference path (e.g., 'refs/heads/new-branch')
- `sha`: Commit SHA to point the new reference to
- `default_branch`: Default branch name

**Pitfalls**:
- `GITHUB_CREATE_A_REFERENCE` only creates NEW references; use `GITHUB_UPDATE_A_REFERENCE` for existing ones
- `ref` must start with 'refs/' and contain at least two slashes
- `GITHUB_LIST_BRANCHES` paginates via `page`/`per_page`; iterate until empty page
- `GITHUB_DELETE_A_REPOSITORY` is permanent and irreversible; requires admin privileges

### 4. Search Code and Commits

**When to use**: User wants to find code, files, or commits across repositories

**Tool sequence**:
1. `GITHUB_SEARCH_CODE` - Search file contents and paths [Required]
2. `GITHUB_SEARCH_CODE_ALL_PAGES` - Multi-page code search [Alternative]
3. `GITHUB_SEARCH_COMMITS_BY_AUTHOR` - Search commits by author/date/org [Required]
4. `GITHUB_LIST_COMMITS` - List commits for a specific repo [Alternative]
5. `GITHUB_GET_A_COMMIT` - Get detailed commit info [Optional]
6. `GITHUB_GET_REPOSITORY_CONTENT` - Get file content [Optional]

**Key parameters**:
- `q`: Search query with qualifiers (`language:python`, `repo:owner/repo`, `extension:js`)
- `owner`/`repo`: For repo-specific commit listing
- `author`: Filter by commit author
- `since`/`until`: ISO 8601 date range for commits

**Pitfalls**:
- Code search only indexes files under 384KB on default branch
- Maximum 1000 results returned from code search
- `GITHUB_SEARCH_COMMITS_BY_AUTHOR` requires keywords in addition to qualifiers; qualifier-only queries are not allowed
- `GITHUB_LIST_COMMITS` returns 409 on empty repos

### 5. Manage CI/CD and Deployments

**When to use**: User wants to view workflows, check CI status, or manage deployments

**Tool sequence**:
1. `GITHUB_LIST_REPOSITORY_WORKFLOWS` - List GitHub Actions workflows [Required]
2. `GITHUB_GET_A_WORKFLOW` - Get workflow details by ID or filename [Optional]
3. `GITHUB_CREATE_A_WORKFLOW_DISPATCH_EVENT` - Manually trigger a workflow [Required]
4. `GITHUB_LIST_CHECK_RUNS_FOR_A_REF` - Check CI status for a commit/branch [Required]
5. `GITHUB_LIST_DEPLOYMENTS` - List deployments [Optional]
6. `GITHUB_GET_A_DEPLOYMENT_STATUS` - Get deployment status [Optional]

**Key parameters**:
- `workflow_id`: Numeric ID or filename (e.g., 'ci.yml')
- `ref`: Git reference (branch/tag) for workflow dispatch
- `inputs`: JSON string of workflow inputs matching `on.workflow_dispatch.inputs`
- `environment`: Filter deployments by environment name

**Pitfalls**:
- `GITHUB_CREATE_A_WORKFLOW_DISPATCH_EVENT` requires the workflow to have `workflow_dispatch` trigger configured
- Full path `.github/workflows/main.yml` is auto-stripped to just `main.yml`
- Inputs max 10 key-value pairs; must match workflow's `on.workflow_dispatch.inputs` definitions

### 6. Manage Users and Permissions

**When to use**: User wants to check collaborators, permissions, or branch protection

**Tool sequence**:
1. `GITHUB_LIST_REPOSITORY_COLLABORATORS` - List repo collaborators [Required]
2. `GITHUB_GET_REPOSITORY_PERMISSIONS_FOR_A_USER` - Check specific user's access [Optional]
3. `GITHUB_GET_BRANCH_PROTECTION` - Inspect branch protection rules [Required]
4. `GITHUB_UPDATE_BRANCH_PROTECTION` - Update protection settings [Optional]
5. `GITHUB_ADD_A_REPOSITORY_COLLABORATOR` - Add/update collaborator [Optional]

**Key parameters**:
- `affiliation`: 'outside', 'direct', or 'all' for collaborator filtering
- `permission`: Filter by 'pull', 'triage', 'push', 'maintain', 'admin'
- `branch`: Branch name for protection rules
- `enforce_admins`: Whether protection applies to admins

**Pitfalls**:
- `GITHUB_GET_BRANCH_PROTECTION` returns 404 for unprotected branches; treat as no protection rules
- Determine push ability from `permissions.push` or `role_name`, not display labels
- `GITHUB_LIST_REPOSITORY_COLLABORATORS` paginates; iterate all pages
- `GITHUB_GET_REPOSITORY_PERMISSIONS_FOR_A_USER` may be inconclusive for non-collaborators

## Common Patterns

### ID Resolution
- **Repo name -> owner/repo**: `GITHUB_LIST_REPOSITORIES_FOR_THE_AUTHENTICATED_USER`
- **PR number -> PR details**: `GITHUB_FIND_PULL_REQUESTS` then `GITHUB_GET_A_PULL_REQUEST`
- **Branch name -> SHA**: `GITHUB_GET_A_BRANCH`
- **Workflow name -> ID**: `GITHUB_LIST_REPOSITORY_WORKFLOWS`

### Pagination
All list endpoints use page-based pagination:
- `page`: Page number (starts at 1)
- `per_page`: Results per page (max 100)
- Iterate until response returns fewer results than `per_page`

### Safety
- Always verify PR mergeable status before merge
- Require explicit user confirmation for destructive operations (merge, delete)
- Check CI status with `GITHUB_LIST_CHECK_RUNS_FOR_A_REF` before merging

## Known Pitfalls

- **Issues vs PRs**: `GITHUB_LIST_REPOSITORY_ISSUES` returns both; check `pull_request` field
- **Pagination limits**: `per_page` max 100; always iterate pages until empty
- **Branch creation**: `GITHUB_CREATE_A_REFERENCE` fails with 422 if reference already exists
- **Merge guards**: Merge can fail due to branch protection, failing checks, or draft status
- **Code search limits**: Only files <384KB on default branch; max 1000 results
- **Commit search**: Requires search text keywords alongside qualifiers
- **Destructive actions**: Repo deletion is irreversible; merge cannot be undone
- **Silent permission drops**: Labels, assignees, milestones silently dropped without push access

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List repos | `GITHUB_LIST_REPOSITORIES_FOR_THE_AUTHENTICATED_USER` | `type`, `sort`, `per_page` |
| Get repo | `GITHUB_GET_A_REPOSITORY` | `owner`, `repo` |
| Create issue | `GITHUB_CREATE_AN_ISSUE` | `owner`, `repo`, `title`, `body` |
| List issues | `GITHUB_LIST_REPOSITORY_ISSUES` | `owner`, `repo`, `state` |
| Find PRs | `GITHUB_FIND_PULL_REQUESTS` | `repo`, `state`, `author` |
| Create PR | `GITHUB_CREATE_A_PULL_REQUEST` | `owner`, `repo`, `head`, `base`, `title` |
| Merge PR | `GITHUB_MERGE_A_PULL_REQUEST` | `owner`, `repo`, `pull_number`, `merge_method` |
| List branches | `GITHUB_LIST_BRANCHES` | `owner`, `repo` |
| Create branch | `GITHUB_CREATE_A_REFERENCE` | `owner`, `repo`, `ref`, `sha` |
| Search code | `GITHUB_SEARCH_CODE` | `q` |
| List commits | `GITHUB_LIST_COMMITS` | `owner`, `repo`, `author`, `since` |
| Search commits | `GITHUB_SEARCH_COMMITS_BY_AUTHOR` | `q` |
| List workflows | `GITHUB_LIST_REPOSITORY_WORKFLOWS` | `owner`, `repo` |
| Trigger workflow | `GITHUB_CREATE_A_WORKFLOW_DISPATCH_EVENT` | `owner`, `repo`, `workflow_id`, `ref` |
| Check CI | `GITHUB_LIST_CHECK_RUNS_FOR_A_REF` | `owner`, `repo`, ref |
| List collaborators | `GITHUB_LIST_REPOSITORY_COLLABORATORS` | `owner`, `repo` |
| Branch protection | `GITHUB_GET_BRANCH_PROTECTION` | `owner`, `repo`, `branch` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: asana-automation
description: "Automate Asana tasks via Rube MCP (Composio): tasks, projects, sections, teams, workspaces. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Asana Automation via Rube MCP

Automate Asana operations through Composio's Asana toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Asana connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `asana`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `asana`
3. If connection is not ACTIVE, follow the returned auth link to complete Asana OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Tasks

**When to use**: User wants to create, search, list, or organize tasks

**Tool sequence**:
1. `ASANA_GET_MULTIPLE_WORKSPACES` - Get workspace ID [Prerequisite]
2. `ASANA_SEARCH_TASKS_IN_WORKSPACE` - Search tasks [Optional]
3. `ASANA_GET_TASKS_FROM_A_PROJECT` - List project tasks [Optional]
4. `ASANA_CREATE_A_TASK` - Create a new task [Optional]
5. `ASANA_GET_A_TASK` - Get task details [Optional]
6. `ASANA_CREATE_SUBTASK` - Create a subtask [Optional]
7. `ASANA_GET_TASK_SUBTASKS` - List subtasks [Optional]

**Key parameters**:
- `workspace`: Workspace GID (required for search/creation)
- `projects`: Array of project GIDs to add task to
- `name`: Task name
- `notes`: Task description
- `assignee`: Assignee (user GID or email)
- `due_on`: Due date (YYYY-MM-DD)

**Pitfalls**:
- Workspace GID is required for most operations; get it first
- Task GIDs are returned as strings, not integers
- Search is workspace-scoped, not project-scoped

### 2. Manage Projects and Sections

**When to use**: User wants to create projects, manage sections, or organize tasks

**Tool sequence**:
1. `ASANA_GET_WORKSPACE_PROJECTS` - List workspace projects [Optional]
2. `ASANA_GET_A_PROJECT` - Get project details [Optional]
3. `ASANA_CREATE_A_PROJECT` - Create a new project [Optional]
4. `ASANA_GET_SECTIONS_IN_PROJECT` - List sections [Optional]
5. `ASANA_CREATE_SECTION_IN_PROJECT` - Create a new section [Optional]
6. `ASANA_ADD_TASK_TO_SECTION` - Move task to section [Optional]
7. `ASANA_GET_TASKS_FROM_A_SECTION` - List tasks in section [Optional]

**Key parameters**:
- `project_gid`: Project GID
- `name`: Project or section name
- `workspace`: Workspace GID for creation
- `task`: Task GID for section assignment
- `section`: Section GID

**Pitfalls**:
- Projects belong to workspaces; workspace GID is needed for creation
- Sections are ordered within a project
- DUPLICATE_PROJECT creates a copy with optional task inclusion

### 3. Manage Teams and Users

**When to use**: User wants to list teams, team members, or workspace users

**Tool sequence**:
1. `ASANA_GET_TEAMS_IN_WORKSPACE` - List workspace teams [Optional]
2. `ASANA_GET_USERS_FOR_TEAM` - List team members [Optional]
3. `ASANA_GET_USERS_FOR_WORKSPACE` - List all workspace users [Optional]
4. `ASANA_GET_CURRENT_USER` - Get authenticated user [Optional]
5. `ASANA_GET_MULTIPLE_USERS` - Get multiple user details [Optional]

**Key parameters**:
- `workspace_gid`: Workspace GID
- `team_gid`: Team GID

**Pitfalls**:
- Users are workspace-scoped
- Team membership requires the team GID

### 4. Parallel Operations

**When to use**: User needs to perform bulk operations efficiently

**Tool sequence**:
1. `ASANA_SUBMIT_PARALLEL_REQUESTS` - Execute multiple API calls in parallel [Required]

**Key parameters**:
- `actions`: Array of action objects with method, path, and data

**Pitfalls**:
- Each action must be a valid Asana API call
- Failed individual requests do not roll back successful ones

## Common Patterns

### ID Resolution

**Workspace name -> GID**:
```
1. Call ASANA_GET_MULTIPLE_WORKSPACES
2. Find workspace by name
3. Extract gid field
```

**Project name -> GID**:
```
1. Call ASANA_GET_WORKSPACE_PROJECTS with workspace GID
2. Find project by name
3. Extract gid field
```

### Pagination

- Asana uses cursor-based pagination with `offset` parameter
- Check for `next_page` in response
- Pass `offset` from `next_page.offset` for next request

## Known Pitfalls

**GID Format**:
- All Asana IDs are strings (GIDs), not integers
- GIDs are globally unique identifiers

**Workspace Scoping**:
- Most operations require a workspace context
- Tasks, projects, and users are workspace-scoped

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | ASANA_GET_MULTIPLE_WORKSPACES | (none) |
| Search tasks | ASANA_SEARCH_TASKS_IN_WORKSPACE | workspace, text |
| Create task | ASANA_CREATE_A_TASK | workspace, name, projects |
| Get task | ASANA_GET_A_TASK | task_gid |
| Create subtask | ASANA_CREATE_SUBTASK | parent, name |
| List subtasks | ASANA_GET_TASK_SUBTASKS | task_gid |
| Project tasks | ASANA_GET_TASKS_FROM_A_PROJECT | project_gid |
| List projects | ASANA_GET_WORKSPACE_PROJECTS | workspace |
| Create project | ASANA_CREATE_A_PROJECT | workspace, name |
| Get project | ASANA_GET_A_PROJECT | project_gid |
| Duplicate project | ASANA_DUPLICATE_PROJECT | project_gid |
| List sections | ASANA_GET_SECTIONS_IN_PROJECT | project_gid |
| Create section | ASANA_CREATE_SECTION_IN_PROJECT | project_gid, name |
| Add to section | ASANA_ADD_TASK_TO_SECTION | section, task |
| Section tasks | ASANA_GET_TASKS_FROM_A_SECTION | section_gid |
| List teams | ASANA_GET_TEAMS_IN_WORKSPACE | workspace_gid |
| Team members | ASANA_GET_USERS_FOR_TEAM | team_gid |
| Workspace users | ASANA_GET_USERS_FOR_WORKSPACE | workspace_gid |
| Current user | ASANA_GET_CURRENT_USER | (none) |
| Parallel requests | ASANA_SUBMIT_PARALLEL_REQUESTS | actions |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: basecamp-automation
description: "Automate Basecamp project management, to-dos, messages, people, and to-do list organization via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Basecamp Automation via Rube MCP

Automate Basecamp operations including project management, to-do list creation, task management, message board posting, people management, and to-do group organization through Composio's Basecamp toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Basecamp connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `basecamp`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `basecamp`
3. If connection is not ACTIVE, follow the returned auth link to complete Basecamp OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage To-Do Lists and Tasks

**When to use**: User wants to create to-do lists, add tasks, or organize work within a Basecamp project

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - List projects to find the target bucket_id [Prerequisite]
2. `BASECAMP_GET_BUCKETS_TODOSETS` - Get the to-do set within a project [Prerequisite]
3. `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS` - List existing to-do lists to avoid duplicates [Optional]
4. `BASECAMP_POST_BUCKETS_TODOSETS_TODOLISTS` - Create a new to-do list in a to-do set [Required for list creation]
5. `BASECAMP_GET_BUCKETS_TODOLISTS` - Get details of a specific to-do list [Optional]
6. `BASECAMP_POST_BUCKETS_TODOLISTS_TODOS` - Create a to-do item in a to-do list [Required for task creation]
7. `BASECAMP_CREATE_TODO` - Alternative tool for creating individual to-dos [Alternative]
8. `BASECAMP_GET_BUCKETS_TODOLISTS_TODOS` - List to-dos within a to-do list [Optional]

**Key parameters for creating to-do lists**:
- `bucket_id`: Integer project/bucket ID (from GET_PROJECTS)
- `todoset_id`: Integer to-do set ID (from GET_BUCKETS_TODOSETS)
- `name`: Title of the to-do list (required)
- `description`: HTML-formatted description (supports Rich text)

**Key parameters for creating to-dos**:
- `bucket_id`: Integer project/bucket ID
- `todolist_id`: Integer to-do list ID
- `content`: What the to-do is for (required)
- `description`: HTML details about the to-do
- `assignee_ids`: Array of integer person IDs
- `due_on`: Due date in `YYYY-MM-DD` format
- `starts_on`: Start date in `YYYY-MM-DD` format
- `notify`: Boolean to notify assignees (defaults to false)
- `completion_subscriber_ids`: Person IDs notified upon completion

**Pitfalls**:
- A project (bucket) can contain multiple to-do sets; selecting the wrong `todoset_id` creates lists in the wrong section
- Always check existing to-do lists before creating to avoid near-duplicate names
- Success payloads include user-facing URLs (`app_url`, `app_todos_url`); prefer returning these over raw IDs
- All IDs (`bucket_id`, `todoset_id`, `todolist_id`) are integers, not strings
- Descriptions support HTML formatting only, not Markdown

### 2. Post and Manage Messages

**When to use**: User wants to post messages to a project message board or update existing messages

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - Find the target project and bucket_id [Prerequisite]
2. `BASECAMP_GET_MESSAGE_BOARD` - Get the message board ID for the project [Prerequisite]
3. `BASECAMP_CREATE_MESSAGE` - Create a new message on the board [Required]
4. `BASECAMP_POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` - Alternative message creation tool [Fallback]
5. `BASECAMP_GET_MESSAGE` - Read a specific message by ID [Optional]
6. `BASECAMP_PUT_BUCKETS_MESSAGES` - Update an existing message [Optional]

**Key parameters**:
- `bucket_id`: Integer project/bucket ID
- `message_board_id`: Integer message board ID (from GET_MESSAGE_BOARD)
- `subject`: Message title (required)
- `content`: HTML body of the message
- `status`: Set to `"active"` to publish immediately
- `category_id`: Message type classification (optional)
- `subscriptions`: Array of person IDs to notify; omit to notify all project members

**Pitfalls**:
- `status="draft"` can produce HTTP 400; use `status="active"` as the reliable option
- `bucket_id` and `message_board_id` must belong to the same project; mismatches fail or misroute
- Message content supports HTML tags only; not Markdown
- Updates via `PUT_BUCKETS_MESSAGES` replace the entire body -- include the full corrected content, not just a diff
- Prefer `app_url` from the response for user-facing confirmation links
- Both `CREATE_MESSAGE` and `POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` do the same thing; use CREATE_MESSAGE first and fall back to POST if it fails

### 3. Manage People and Access

**When to use**: User wants to list people, manage project access, or add new users

**Tool sequence**:
1. `BASECAMP_GET_PEOPLE` - List all people visible to the current user [Required]
2. `BASECAMP_GET_PROJECTS` - Find the target project [Prerequisite]
3. `BASECAMP_LIST_PROJECT_PEOPLE` - List people on a specific project [Required]
4. `BASECAMP_GET_PROJECTS_PEOPLE` - Alternative to list project members [Alternative]
5. `BASECAMP_PUT_PROJECTS_PEOPLE_USERS` - Grant or revoke project access [Required for access changes]

**Key parameters for PUT_PROJECTS_PEOPLE_USERS**:
- `project_id`: Integer project ID
- `grant`: Array of integer person IDs to add to the project
- `revoke`: Array of integer person IDs to remove from the project
- `create`: Array of objects with `name`, `email_address`, and optional `company_name`, `title` for new users
- At least one of `grant`, `revoke`, or `create` must be provided

**Pitfalls**:
- Person IDs are integers; always resolve names to IDs via GET_PEOPLE first
- `project_id` for people management is the same as `bucket_id` for other operations
- `LIST_PROJECT_PEOPLE` and `GET_PROJECTS_PEOPLE` are near-identical; use either
- Creating users via `create` also grants them project access in one step

### 4. Organize To-Dos with Groups

**When to use**: User wants to organize to-dos within a list into color-coded groups

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - Find the target project [Prerequisite]
2. `BASECAMP_GET_BUCKETS_TODOLISTS` - Get the to-do list details [Prerequisite]
3. `BASECAMP_GET_TODOLIST_GROUPS` - List existing groups in a to-do list [Optional]
4. `BASECAMP_GET_BUCKETS_TODOLISTS_GROUPS` - Alternative group listing [Alternative]
5. `BASECAMP_POST_BUCKETS_TODOLISTS_GROUPS` - Create a new group in a to-do list [Required]
6. `BASECAMP_CREATE_TODOLIST_GROUP` - Alternative group creation tool [Alternative]

**Key parameters**:
- `bucket_id`: Integer project/bucket ID
- `todolist_id`: Integer to-do list ID
- `name`: Group title (required)
- `color`: Visual color identifier -- one of: `white`, `red`, `orange`, `yellow`, `green`, `blue`, `aqua`, `purple`, `gray`, `pink`, `brown`
- `status`: Filter for listing -- `"archived"` or `"trashed"` (omit for active groups)

**Pitfalls**:
- `POST_BUCKETS_TODOLISTS_GROUPS` and `CREATE_TODOLIST_GROUP` are near-identical; use either
- Color values must be from the fixed palette; arbitrary hex/rgb values are not supported
- Groups are sub-sections within a to-do list, not standalone entities

### 5. Browse and Inspect Projects

**When to use**: User wants to list projects, get project details, or explore project structure

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - List all active projects [Required]
2. `BASECAMP_GET_PROJECT` - Get comprehensive details for a specific project [Optional]
3. `BASECAMP_GET_PROJECTS_BY_PROJECT_ID` - Alternative project detail retrieval [Alternative]

**Key parameters**:
- `status`: Filter by `"archived"` or `"trashed"`; omit for active projects
- `project_id`: Integer project ID for detailed retrieval

**Pitfalls**:
- Projects are sorted by most recently created first
- The response includes a `dock` array with tools (todoset, message_board, etc.) and their IDs
- Use the dock tool IDs to find `todoset_id`, `message_board_id`, etc. for downstream operations

## Common Patterns

### ID Resolution
Basecamp uses a hierarchical ID structure. Always resolve top-down:
- **Project (bucket_id)**: `BASECAMP_GET_PROJECTS` -- find by name, capture the `id`
- **To-do set (todoset_id)**: Found in project dock or via `BASECAMP_GET_BUCKETS_TODOSETS`
- **Message board (message_board_id)**: Found in project dock or via `BASECAMP_GET_MESSAGE_BOARD`
- **To-do list (todolist_id)**: `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS`
- **People (person_id)**: `BASECAMP_GET_PEOPLE` or `BASECAMP_LIST_PROJECT_PEOPLE`
- Note: `bucket_id` and `project_id` refer to the same entity in different contexts

### Pagination
Basecamp uses page-based pagination on list endpoints:
- Response headers or body may indicate more pages available
- `GET_PROJECTS`, `GET_BUCKETS_TODOSETS_TODOLISTS`, and list endpoints return paginated results
- Continue fetching until no more results are returned

### Content Formatting
- All rich text fields use HTML, not Markdown
- Wrap content in `<div>` tags; use `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>`, `<a>` etc.
- Example: `<div><strong>Important:</strong> Complete by Friday</div>`

## Known Pitfalls

### ID Formats
- All Basecamp IDs are integers, not strings or UUIDs
- `bucket_id` = `project_id` (same entity, different parameter names across tools)
- To-do set IDs, to-do list IDs, and message board IDs are found in the project's `dock` array
- Person IDs are integers; resolve names via `GET_PEOPLE` before operations

### Status Field
- `status="draft"` for messages can cause HTTP 400; always use `status="active"`
- Project/to-do list status filters: `"archived"`, `"trashed"`, or omit for active

### Content Format
- HTML only, never Markdown
- Updates replace the entire body, not a partial diff
- Invalid HTML tags may be silently stripped

### Rate Limits
- Basecamp API has rate limits; space out rapid sequential requests
- Large projects with many to-dos should be paginated carefully

### URL Handling
- Prefer `app_url` from API responses for user-facing links
- Do not reconstruct Basecamp URLs manually from IDs

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List projects | `BASECAMP_GET_PROJECTS` | `status` |
| Get project | `BASECAMP_GET_PROJECT` | `project_id` |
| Get project detail | `BASECAMP_GET_PROJECTS_BY_PROJECT_ID` | `project_id` |
| Get to-do set | `BASECAMP_GET_BUCKETS_TODOSETS` | `bucket_id`, `todoset_id` |
| List to-do lists | `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS` | `bucket_id`, `todoset_id` |
| Get to-do list | `BASECAMP_GET_BUCKETS_TODOLISTS` | `bucket_id`, `todolist_id` |
| Create to-do list | `BASECAMP_POST_BUCKETS_TODOSETS_TODOLISTS` | `bucket_id`, `todoset_id`, `name` |
| Create to-do | `BASECAMP_POST_BUCKETS_TODOLISTS_TODOS` | `bucket_id`, `todolist_id`, `content` |
| Create to-do (alt) | `BASECAMP_CREATE_TODO` | `bucket_id`, `todolist_id`, `content` |
| List to-dos | `BASECAMP_GET_BUCKETS_TODOLISTS_TODOS` | `bucket_id`, `todolist_id` |
| List to-do groups | `BASECAMP_GET_TODOLIST_GROUPS` | `bucket_id`, `todolist_id` |
| Create to-do group | `BASECAMP_POST_BUCKETS_TODOLISTS_GROUPS` | `bucket_id`, `todolist_id`, `name`, `color` |
| Create to-do group (alt) | `BASECAMP_CREATE_TODOLIST_GROUP` | `bucket_id`, `todolist_id`, `name` |
| Get message board | `BASECAMP_GET_MESSAGE_BOARD` | `bucket_id`, `message_board_id` |
| Create message | `BASECAMP_CREATE_MESSAGE` | `bucket_id`, `message_board_id`, `subject`, `status` |
| Create message (alt) | `BASECAMP_POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` | `bucket_id`, `message_board_id`, `subject` |
| Get message | `BASECAMP_GET_MESSAGE` | `bucket_id`, `message_id` |
| Update message | `BASECAMP_PUT_BUCKETS_MESSAGES` | `bucket_id`, `message_id` |
| List all people | `BASECAMP_GET_PEOPLE` | (none) |
| List project people | `BASECAMP_LIST_PROJECT_PEOPLE` | `project_id` |
| Manage access | `BASECAMP_PUT_PROJECTS_PEOPLE_USERS` | `project_id`, `grant`, `revoke`, `create` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: bitbucket-automation
description: "Automate Bitbucket repositories, pull requests, branches, issues, and workspace management via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Bitbucket Automation via Rube MCP

Automate Bitbucket operations including repository management, pull request workflows, branch operations, issue tracking, and workspace administration through Composio's Bitbucket toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Bitbucket connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `bitbucket`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `bitbucket`
3. If connection is not ACTIVE, follow the returned auth link to complete Bitbucket OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Pull Requests

**When to use**: User wants to create, review, or inspect pull requests

**Tool sequence**:
1. `BITBUCKET_LIST_WORKSPACES` - Discover accessible workspaces [Prerequisite]
2. `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` - Find the target repository [Prerequisite]
3. `BITBUCKET_LIST_BRANCHES` - Verify source and destination branches exist [Prerequisite]
4. `BITBUCKET_CREATE_PULL_REQUEST` - Create a new PR with title, source branch, and optional reviewers [Required]
5. `BITBUCKET_LIST_PULL_REQUESTS` - List PRs filtered by state (OPEN, MERGED, DECLINED) [Optional]
6. `BITBUCKET_GET_PULL_REQUEST` - Get full details of a specific PR by ID [Optional]
7. `BITBUCKET_GET_PULL_REQUEST_DIFF` - Fetch unified diff for code review [Optional]
8. `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` - Get changed files with lines added/removed [Optional]

**Key parameters**:
- `workspace`: Workspace slug or UUID (required for all operations)
- `repo_slug`: URL-friendly repository name
- `source_branch`: Branch with changes to merge
- `destination_branch`: Target branch (defaults to repo main branch if omitted)
- `reviewers`: List of objects with `uuid` field for reviewer assignment
- `state`: Filter for LIST_PULL_REQUESTS - `OPEN`, `MERGED`, or `DECLINED`
- `max_chars`: Truncation limit for GET_PULL_REQUEST_DIFF to handle large diffs

**Pitfalls**:
- `reviewers` expects an array of objects with `uuid` key, NOT usernames: `[{"uuid": "{...}"}]`
- UUID format must include curly braces: `{123e4567-e89b-12d3-a456-426614174000}`
- `destination_branch` defaults to the repo's main branch if omitted, which may not be `main`
- `pull_request_id` is an integer for GET/DIFF operations but comes back as part of PR listing
- Large diffs can overwhelm context; always set `max_chars` (e.g., 50000) on GET_PULL_REQUEST_DIFF

### 2. Manage Repositories and Workspaces

**When to use**: User wants to list, create, or delete repositories or explore workspaces

**Tool sequence**:
1. `BITBUCKET_LIST_WORKSPACES` - List all accessible workspaces [Required]
2. `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` - List repos with optional BBQL filtering [Required]
3. `BITBUCKET_CREATE_REPOSITORY` - Create a new repo with language, privacy, and project settings [Optional]
4. `BITBUCKET_DELETE_REPOSITORY` - Permanently delete a repository (irreversible) [Optional]
5. `BITBUCKET_LIST_WORKSPACE_MEMBERS` - List members for reviewer assignment or access checks [Optional]

**Key parameters**:
- `workspace`: Workspace slug (find via LIST_WORKSPACES)
- `repo_slug`: URL-friendly name for create/delete
- `q`: BBQL query filter (e.g., `name~"api"`, `project.key="PROJ"`, `is_private=true`)
- `role`: Filter repos by user role: `member`, `contributor`, `admin`, `owner`
- `sort`: Sort field with optional `-` prefix for descending (e.g., `-updated_on`)
- `is_private`: Boolean for repository visibility (defaults to `true`)
- `project_key`: Bitbucket project key; omit to use workspace's oldest project

**Pitfalls**:
- `BITBUCKET_DELETE_REPOSITORY` is **irreversible** and does not affect forks
- BBQL string values MUST be enclosed in double quotes: `name~"my-repo"` not `name~my-repo`
- `repository` is NOT a valid BBQL field; use `name` instead
- Default pagination is 10 results; set `pagelen` explicitly for complete listings
- `CREATE_REPOSITORY` defaults to private; set `is_private: false` for public repos

### 3. Manage Issues

**When to use**: User wants to create, update, list, or comment on repository issues

**Tool sequence**:
1. `BITBUCKET_LIST_ISSUES` - List issues with optional filters for state, priority, kind, assignee [Required]
2. `BITBUCKET_CREATE_ISSUE` - Create a new issue with title, content, priority, and kind [Required]
3. `BITBUCKET_UPDATE_ISSUE` - Modify issue attributes (state, priority, assignee, etc.) [Optional]
4. `BITBUCKET_CREATE_ISSUE_COMMENT` - Add a markdown comment to an existing issue [Optional]
5. `BITBUCKET_DELETE_ISSUE` - Permanently delete an issue [Optional]

**Key parameters**:
- `issue_id`: String identifier for the issue
- `title`, `content`: Required for creation
- `kind`: `bug`, `enhancement`, `proposal`, or `task`
- `priority`: `trivial`, `minor`, `major`, `critical`, or `blocker`
- `state`: `new`, `open`, `resolved`, `on hold`, `invalid`, `duplicate`, `wontfix`, `closed`
- `assignee`: Bitbucket username for CREATE; `assignee_account_id` (UUID) for UPDATE
- `due_on`: ISO 8601 format date string

**Pitfalls**:
- Issue tracker must be enabled on the repository (`has_issues: true`) or API calls will fail
- `CREATE_ISSUE` uses `assignee` (username string), but `UPDATE_ISSUE` uses `assignee_account_id` (UUID) -- they are different fields
- `DELETE_ISSUE` is permanent with no undo
- `state` values include spaces: `"on hold"` not `"on_hold"`
- Filtering by `assignee` in LIST_ISSUES uses account ID, not username; use `"null"` string for unassigned

### 4. Manage Branches

**When to use**: User wants to create branches or explore branch structure

**Tool sequence**:
1. `BITBUCKET_LIST_BRANCHES` - List branches with optional BBQL filter and sorting [Required]
2. `BITBUCKET_CREATE_BRANCH` - Create a new branch from a specific commit hash [Required]

**Key parameters**:
- `name`: Branch name without `refs/heads/` prefix (e.g., `feature/new-login`)
- `target_hash`: Full SHA1 commit hash to branch from (must exist in repo)
- `q`: BBQL filter (e.g., `name~"feature/"`, `name="main"`)
- `sort`: Sort by `name` or `-target.date` (descending commit date)
- `pagelen`: 1-100 results per page (default is 10)

**Pitfalls**:
- `CREATE_BRANCH` requires a full commit hash, NOT a branch name as `target_hash`
- Do NOT include `refs/heads/` prefix in branch names
- Branch names must follow Bitbucket naming conventions (alphanumeric, `/`, `.`, `_`, `-`)
- BBQL string values need double quotes: `name~"feature/"` not `name~feature/`

### 5. Review Pull Requests with Comments

**When to use**: User wants to add review comments to pull requests, including inline code comments

**Tool sequence**:
1. `BITBUCKET_GET_PULL_REQUEST` - Get PR details and verify it exists [Prerequisite]
2. `BITBUCKET_GET_PULL_REQUEST_DIFF` - Review the actual code changes [Prerequisite]
3. `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` - Get list of changed files [Optional]
4. `BITBUCKET_CREATE_PULL_REQUEST_COMMENT` - Post review comments [Required]

**Key parameters**:
- `pull_request_id`: String ID of the PR
- `content_raw`: Markdown-formatted comment text
- `content_markup`: Defaults to `markdown`; also supports `plaintext`
- `inline`: Object with `path`, `from`, `to` for inline code comments
- `parent_comment_id`: Integer ID for threaded replies to existing comments

**Pitfalls**:
- `pull_request_id` is a string in CREATE_PULL_REQUEST_COMMENT but an integer in GET_PULL_REQUEST
- Inline comments require `inline.path` at minimum; `from`/`to` are optional line numbers
- `parent_comment_id` creates a threaded reply; omit for top-level comments
- Line numbers in inline comments reference the diff, not the source file

## Common Patterns

### ID Resolution
Always resolve human-readable names to IDs before operations:
- **Workspace**: `BITBUCKET_LIST_WORKSPACES` to get workspace slugs
- **Repository**: `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` with `q` filter to find repo slugs
- **Branch**: `BITBUCKET_LIST_BRANCHES` to verify branch existence before PR creation
- **Members**: `BITBUCKET_LIST_WORKSPACE_MEMBERS` to get UUIDs for reviewer assignment

### Pagination
Bitbucket uses page-based pagination (not cursor-based):
- Use `page` (starts at 1) and `pagelen` (items per page) parameters
- Default page size is typically 10; set `pagelen` explicitly (max 50 for PRs, 100 for others)
- Check response for `next` URL or total count to determine if more pages exist
- Always iterate through all pages for complete results

### BBQL Filtering
Bitbucket Query Language is available on list endpoints:
- String values MUST use double quotes: `name~"pattern"`
- Operators: `=` (exact), `~` (contains), `!=` (not equal), `>`, `>=`, `<`, `<=`
- Combine with `AND` / `OR`: `name~"api" AND is_private=true`

## Known Pitfalls

### ID Formats
- Workspace: slug string (e.g., `my-workspace`) or UUID in braces (`{uuid}`)
- Reviewer UUIDs must include curly braces: `{123e4567-e89b-12d3-a456-426614174000}`
- Issue IDs are strings; PR IDs are integers in some tools, strings in others
- Commit hashes must be full SHA1 (40 characters)

### Parameter Quirks
- `assignee` vs `assignee_account_id`: CREATE_ISSUE uses username, UPDATE_ISSUE uses UUID
- `state` values for issues include spaces: `"on hold"`, not `"on_hold"`
- `destination_branch` omission defaults to repo main branch, not `main` literally
- BBQL `repository` is not a valid field -- use `name`

### Rate Limits
- Bitbucket Cloud API has rate limits; large batch operations should include delays
- Paginated requests count against rate limits; minimize unnecessary page fetches

### Destructive Operations
- `BITBUCKET_DELETE_REPOSITORY` is irreversible and does not remove forks
- `BITBUCKET_DELETE_ISSUE` is permanent with no recovery option
- Always confirm with the user before executing delete operations

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `BITBUCKET_LIST_WORKSPACES` | `q`, `sort` |
| List repos | `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` | `workspace`, `q`, `role` |
| Create repo | `BITBUCKET_CREATE_REPOSITORY` | `workspace`, `repo_slug`, `is_private` |
| Delete repo | `BITBUCKET_DELETE_REPOSITORY` | `workspace`, `repo_slug` |
| List branches | `BITBUCKET_LIST_BRANCHES` | `workspace`, `repo_slug`, `q` |
| Create branch | `BITBUCKET_CREATE_BRANCH` | `workspace`, `repo_slug`, `name`, `target_hash` |
| List PRs | `BITBUCKET_LIST_PULL_REQUESTS` | `workspace`, `repo_slug`, `state` |
| Create PR | `BITBUCKET_CREATE_PULL_REQUEST` | `workspace`, `repo_slug`, `title`, `source_branch` |
| Get PR details | `BITBUCKET_GET_PULL_REQUEST` | `workspace`, `repo_slug`, `pull_request_id` |
| Get PR diff | `BITBUCKET_GET_PULL_REQUEST_DIFF` | `workspace`, `repo_slug`, `pull_request_id`, `max_chars` |
| Get PR diffstat | `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` | `workspace`, `repo_slug`, `pull_request_id` |
| Comment on PR | `BITBUCKET_CREATE_PULL_REQUEST_COMMENT` | `workspace`, `repo_slug`, `pull_request_id`, `content_raw` |
| List issues | `BITBUCKET_LIST_ISSUES` | `workspace`, `repo_slug`, `state`, `priority` |
| Create issue | `BITBUCKET_CREATE_ISSUE` | `workspace`, `repo_slug`, `title`, `content` |
| Update issue | `BITBUCKET_UPDATE_ISSUE` | `workspace`, `repo_slug`, `issue_id` |
| Comment on issue | `BITBUCKET_CREATE_ISSUE_COMMENT` | `workspace`, `repo_slug`, `issue_id`, `content` |
| Delete issue | `BITBUCKET_DELETE_ISSUE` | `workspace`, `repo_slug`, `issue_id` |
| List members | `BITBUCKET_LIST_WORKSPACE_MEMBERS` | `workspace` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: clickup-automation
description: "Automate ClickUp project management including tasks, spaces, folders, lists, comments, and team operations via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# ClickUp Automation via Rube MCP

Automate ClickUp project management workflows including task creation and updates, workspace hierarchy navigation, comments, and team member management through Composio's ClickUp toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active ClickUp connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `clickup`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `clickup`
3. If connection is not ACTIVE, follow the returned auth link to complete ClickUp OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Tasks

**When to use**: User wants to create tasks, subtasks, update task properties, or list tasks in a ClickUp list.

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - Get workspace/team IDs [Prerequisite]
2. `CLICKUP_GET_SPACES` - List spaces in the workspace [Prerequisite]
3. `CLICKUP_GET_FOLDERS` - List folders in a space [Prerequisite]
4. `CLICKUP_GET_FOLDERLESS_LISTS` - Get lists not inside folders [Optional]
5. `CLICKUP_GET_LIST` - Validate list and check available statuses [Prerequisite]
6. `CLICKUP_CREATE_TASK` - Create a task in the target list [Required]
7. `CLICKUP_CREATE_TASK` (with `parent`) - Create subtask under a parent task [Optional]
8. `CLICKUP_UPDATE_TASK` - Modify task status, assignees, dates, priority [Optional]
9. `CLICKUP_GET_TASK` - Retrieve full task details [Optional]
10. `CLICKUP_GET_TASKS` - List all tasks in a list with filters [Optional]
11. `CLICKUP_DELETE_TASK` - Permanently remove a task [Optional]

**Key parameters for CLICKUP_CREATE_TASK**:
- `list_id`: Target list ID (integer, required)
- `name`: Task name (string, required)
- `description`: Detailed task description
- `status`: Must exactly match (case-sensitive) a status name configured in the target list
- `priority`: 1 (Urgent), 2 (High), 3 (Normal), 4 (Low)
- `assignees`: Array of user IDs (integers)
- `due_date`: Unix timestamp in milliseconds
- `parent`: Parent task ID string for creating subtasks
- `tags`: Array of tag name strings
- `time_estimate`: Estimated time in milliseconds

**Pitfalls**:
- `status` is case-sensitive and must match an existing status in the list; use `CLICKUP_GET_LIST` to check available statuses
- `due_date` and `start_date` are Unix timestamps in **milliseconds**, not seconds
- Subtask `parent` must be a task (not another subtask) in the same list
- `notify_all` triggers watcher notifications; set to false for bulk operations
- Retries can create duplicates; track created task IDs to avoid re-creation
- `custom_item_id` for milestones (ID 1) is subject to workspace plan quotas

### 2. Navigate Workspace Hierarchy

**When to use**: User wants to browse or manage the ClickUp workspace structure (Workspaces > Spaces > Folders > Lists).

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - List all accessible workspaces [Required]
2. `CLICKUP_GET_SPACES` - List spaces within a workspace [Required]
3. `CLICKUP_GET_SPACE` - Get details for a specific space [Optional]
4. `CLICKUP_GET_FOLDERS` - List folders in a space [Required]
5. `CLICKUP_GET_FOLDER` - Get details for a specific folder [Optional]
6. `CLICKUP_CREATE_FOLDER` - Create a new folder in a space [Optional]
7. `CLICKUP_GET_FOLDERLESS_LISTS` - List lists not inside any folder [Required]
8. `CLICKUP_GET_LIST` - Get list details including statuses and custom fields [Optional]

**Key parameters**:
- `team_id`: Workspace ID from GET_AUTHORIZED_TEAMS_WORKSPACES (required for spaces)
- `space_id`: Space ID (required for folders and folderless lists)
- `folder_id`: Folder ID (required for GET_FOLDER)
- `list_id`: List ID (required for GET_LIST)
- `archived`: Boolean filter for archived/active items

**Pitfalls**:
- ClickUp hierarchy is: Workspace (Team) > Space > Folder > List > Task
- Lists can exist directly under Spaces (folderless) or inside Folders
- Must use `CLICKUP_GET_FOLDERLESS_LISTS` to find lists not inside folders; `CLICKUP_GET_FOLDERS` only returns folders
- `team_id` in ClickUp API refers to the Workspace ID, not a user group

### 3. Add Comments to Tasks

**When to use**: User wants to add comments, review existing comments, or manage comment threads on tasks.

**Tool sequence**:
1. `CLICKUP_GET_TASK` - Verify task exists and get task_id [Prerequisite]
2. `CLICKUP_CREATE_TASK_COMMENT` - Add a new comment to the task [Required]
3. `CLICKUP_GET_TASK_COMMENTS` - List existing comments on the task [Optional]
4. `CLICKUP_UPDATE_COMMENT` - Edit comment text, assignee, or resolution status [Optional]

**Key parameters for CLICKUP_CREATE_TASK_COMMENT**:
- `task_id`: Task ID string (required)
- `comment_text`: Comment content with ClickUp formatting support (required)
- `assignee`: User ID to assign the comment to (required)
- `notify_all`: true/false for watcher notifications (required)

**Key parameters for CLICKUP_GET_TASK_COMMENTS**:
- `task_id`: Task ID string (required)
- `start` / `start_id`: Pagination for older comments (max 25 per page)

**Pitfalls**:
- `CLICKUP_CREATE_TASK_COMMENT` requires all four fields: `task_id`, `comment_text`, `assignee`, and `notify_all`
- `assignee` on a comment assigns the comment (not the task) to that user
- Comments are paginated at 25 per page; use `start` (Unix ms) and `start_id` for older pages
- `CLICKUP_UPDATE_COMMENT` requires all four fields: `comment_id`, `comment_text`, `assignee`, `resolved`

### 4. Manage Team Members and Assignments

**When to use**: User wants to view workspace members, check seat utilization, or look up user details.

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - List workspaces and get team_id [Required]
2. `CLICKUP_GET_WORKSPACE_SEATS` - Check seat utilization (members vs guests) [Required]
3. `CLICKUP_GET_TEAMS` - List user groups within the workspace [Optional]
4. `CLICKUP_GET_USER` - Get details for a specific user (Enterprise only) [Optional]
5. `CLICKUP_GET_CUSTOM_ROLES` - List custom permission roles [Optional]

**Key parameters**:
- `team_id`: Workspace ID (required for all team operations)
- `user_id`: Specific user ID for GET_USER
- `group_ids`: Comma-separated group IDs to filter teams

**Pitfalls**:
- `CLICKUP_GET_WORKSPACE_SEATS` returns seat counts, not member details; distinguish members from guests
- `CLICKUP_GET_TEAMS` returns user groups, not workspace members; empty groups does not mean no members
- `CLICKUP_GET_USER` is only available on ClickUp Enterprise Plan
- Must repeat workspace seat queries for each workspace in multi-workspace setups

### 5. Filter and Query Tasks

**When to use**: User wants to find tasks with specific filters (status, assignee, dates, tags, custom fields).

**Tool sequence**:
1. `CLICKUP_GET_TASKS` - Filter tasks in a list with multiple criteria [Required]
2. `CLICKUP_GET_TASK` - Get full details for individual tasks [Optional]

**Key parameters for CLICKUP_GET_TASKS**:
- `list_id`: List ID (integer, required)
- `statuses`: Array of status strings to filter by
- `assignees`: Array of user ID strings
- `tags`: Array of tag name strings
- `due_date_gt` / `due_date_lt`: Unix timestamp in ms for date range
- `include_closed`: Boolean to include closed tasks
- `subtasks`: Boolean to include subtasks
- `order_by`: "id", "created", "updated", or "due_date"
- `page`: Page number starting at 0 (max 100 tasks per page)

**Pitfalls**:
- Only tasks whose home list matches `list_id` are returned; tasks in sublists are not included
- Date filters use Unix timestamps in milliseconds
- Status strings must match exactly; use URL encoding for spaces (e.g., "to%20do")
- Page numbering starts at 0; each page returns up to 100 tasks
- `custom_fields` filter accepts an array of JSON strings, not objects

## Common Patterns

### ID Resolution
Always resolve names to IDs through the hierarchy:
- **Workspace name -> team_id**: `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` and match by name
- **Space name -> space_id**: `CLICKUP_GET_SPACES` with `team_id`
- **Folder name -> folder_id**: `CLICKUP_GET_FOLDERS` with `space_id`
- **List name -> list_id**: Navigate folders or use `CLICKUP_GET_FOLDERLESS_LISTS`
- **Task name -> task_id**: `CLICKUP_GET_TASKS` with `list_id` and match by name

### Pagination
- `CLICKUP_GET_TASKS`: Page-based with `page` starting at 0, max 100 tasks per page
- `CLICKUP_GET_TASK_COMMENTS`: Uses `start` (Unix ms) and `start_id` for cursor-based paging, max 25 per page
- Continue fetching until response returns fewer items than the page size

## Known Pitfalls

### ID Formats
- Workspace/Team IDs are large integers
- Space, folder, and list IDs are integers
- Task IDs are alphanumeric strings (e.g., "9hz", "abc123")
- User IDs are integers
- Comment IDs are integers

### Rate Limits
- ClickUp enforces rate limits; bulk task creation can trigger 429 responses
- Honor `Retry-After` header when present
- Set `notify_all=false` for bulk operations to reduce notification load

### Parameter Quirks
- `team_id` in the API means Workspace ID, not a user group
- `status` on tasks is case-sensitive and list-specific
- Dates are Unix timestamps in **milliseconds** (multiply seconds by 1000)
- `priority` is an integer 1-4 (1=Urgent, 4=Low), not a string
- `CLICKUP_CREATE_TASK_COMMENT` marks `assignee` and `notify_all` as required
- To clear a task description, pass a single space `" "` to `CLICKUP_UPDATE_TASK`

### Hierarchy Rules
- Subtask parent must not itself be a subtask
- Subtask parent must be in the same list
- Lists can be folderless (directly in a Space) or inside a Folder
- Subitem boards are not supported by CLICKUP_CREATE_TASK

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` | (none) |
| List spaces | `CLICKUP_GET_SPACES` | `team_id` |
| Get space details | `CLICKUP_GET_SPACE` | `space_id` |
| List folders | `CLICKUP_GET_FOLDERS` | `space_id` |
| Get folder details | `CLICKUP_GET_FOLDER` | `folder_id` |
| Create folder | `CLICKUP_CREATE_FOLDER` | `space_id`, `name` |
| Folderless lists | `CLICKUP_GET_FOLDERLESS_LISTS` | `space_id` |
| Get list details | `CLICKUP_GET_LIST` | `list_id` |
| Create task | `CLICKUP_CREATE_TASK` | `list_id`, `name`, `status`, `assignees` |
| Update task | `CLICKUP_UPDATE_TASK` | `task_id`, `status`, `priority` |
| Get task | `CLICKUP_GET_TASK` | `task_id`, `include_subtasks` |
| List tasks | `CLICKUP_GET_TASKS` | `list_id`, `statuses`, `page` |
| Delete task | `CLICKUP_DELETE_TASK` | `task_id` |
| Add comment | `CLICKUP_CREATE_TASK_COMMENT` | `task_id`, `comment_text`, `assignee` |
| List comments | `CLICKUP_GET_TASK_COMMENTS` | `task_id`, `start`, `start_id` |
| Update comment | `CLICKUP_UPDATE_COMMENT` | `comment_id`, `comment_text`, `resolved` |
| Workspace seats | `CLICKUP_GET_WORKSPACE_SEATS` | `team_id` |
| List user groups | `CLICKUP_GET_TEAMS` | `team_id` |
| Get user details | `CLICKUP_GET_USER` | `team_id`, `user_id` |
| Custom roles | `CLICKUP_GET_CUSTOM_ROLES` | `team_id` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: github-workflow-automation
description: "Automate GitHub workflows with AI assistance. Includes PR reviews, issue triage, CI/CD integration, and Git operations. Use when automating GitHub workflows, setting up PR review automation, creati..."
risk: unknown
source: community
date_added: "2026-02-27"
---

# 🔧 GitHub Workflow Automation

> Patterns for automating GitHub workflows with AI assistance, inspired by modern agentic CLI and DevOps practices.

## When to Use This Skill

Use this skill when:

- Automating PR reviews with AI
- Setting up issue triage automation
- Creating GitHub Actions workflows
- Integrating AI into CI/CD pipelines
- Automating Git operations (rebases, cherry-picks)

---

## 1. Automated PR Review

### 1.1 PR Review Action

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed
        run: |
          files=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          echo "files<<EOF" >> $GITHUB_OUTPUT
          echo "$files" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Get diff
        id: diff
        run: |
          diff=$(git diff origin/${{ github.base_ref }}...HEAD)
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$diff" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: AI Review
        uses: actions/github-script@v7
        with:
          script: |
            const { Anthropic } = require('@provider-specific-ai-crawler/sdk');
            const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

            const response = await client.messages.create({
              model: "example-model",
              max_tokens: 4096,
              messages: [{
                role: "user",
                content: `Review this PR diff and provide feedback:
                
                Changed files: ${{ steps.changed.outputs.files }}
                
                Diff:
                ${{ steps.diff.outputs.diff }}
                
                Provide:
                1. Summary of changes
                2. Potential issues or bugs
                3. Suggestions for improvement
                4. Security concerns if any
                
                Format as GitHub markdown.`
              }]
            });

            await github.rest.pulls.createReview({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              body: response.content[0].text,
              event: 'COMMENT'
            });
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 1.2 Review Comment Patterns

````markdown
# AI Review Structure

## 📋 Summary

Brief description of what this PR does.

## ✅ What looks good

- Well-structured code
- Good test coverage
- Clear naming conventions

## ⚠️ Potential Issues

1. **Line 42**: Possible null pointer exception
   ```javascript
   // Current
   user.profile.name;
   // Suggested
   user?.profile?.name ?? "Unknown";
   ```
````

2. **Line 78**: Consider error handling
   ```javascript
   // Add try-catch or .catch()
   ```

## 💡 Suggestions

- Consider extracting the validation logic into a separate function
- Add JSDoc comments for public methods

## 🔒 Security Notes

- No sensitive data exposure detected
- API key handling looks correct

````

### 1.3 Focused Reviews

```yaml
# Review only specific file types
- name: Filter code files
  run: |
    files=$(git diff --name-only origin/${{ github.base_ref }}...HEAD | \
            grep -E '\.(ts|tsx|js|jsx|py|go)$' || true)
    echo "code_files=$files" >> $GITHUB_OUTPUT

# Review with context
- name: AI Review with context
  run: |
    # Include relevant context files
    context=""
    for file in ${{ steps.changed.outputs.files }}; do
      if [[ -f "$file" ]]; then
        context+="=== $file ===\n$(cat $file)\n\n"
      fi
    done

    # Send to AI with full file context
````

---

## 2. Issue Triage Automation

### 2.1 Auto-label Issues

```yaml
# .github/workflows/issue-triage.yml
name: Issue Triage

on:
  issues:
    types: [opened]

jobs:
  triage:
    runs-on: ubuntu-latest
    permissions:
      issues: write

    steps:
      - name: Analyze issue
        uses: actions/github-script@v7
        with:
          script: |
            const issue = context.payload.issue;

            // Call AI to analyze
            const analysis = await analyzeIssue(issue.title, issue.body);

            // Apply labels
            const labels = [];

            if (analysis.type === 'bug') {
              labels.push('bug');
              if (analysis.severity === 'high') labels.push('priority: high');
            } else if (analysis.type === 'feature') {
              labels.push('enhancement');
            } else if (analysis.type === 'question') {
              labels.push('question');
            }

            if (analysis.area) {
              labels.push(`area: ${analysis.area}`);
            }

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue.number,
              labels: labels
            });

            // Add initial response
            if (analysis.type === 'bug' && !analysis.hasReproSteps) {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                body: `Thanks for reporting this issue!

To help us investigate, could you please provide:
- Steps to reproduce the issue
- Expected behavior
- Actual behavior
- Environment (OS, version, etc.)

This will help us resolve your issue faster. 🙏`
              });
            }
```

### 2.2 Issue Analysis Prompt

```typescript
const TRIAGE_PROMPT = `
Analyze this GitHub issue and classify it:

Title: {title}
Body: {body}

Return JSON with:
{
  "type": "bug" | "feature" | "question" | "docs" | "other",
  "severity": "low" | "medium" | "high" | "critical",
  "area": "frontend" | "backend" | "api" | "docs" | "ci" | "other",
  "summary": "one-line summary",
  "hasReproSteps": boolean,
  "isFirstContribution": boolean,
  "suggestedLabels": ["label1", "label2"],
  "suggestedAssignees": ["username"] // based on area expertise
}
`;
```

### 2.3 Stale Issue Management

```yaml
# .github/workflows/stale.yml
name: Manage Stale Issues

on:
  schedule:
    - cron: "0 0 * * *" # Daily

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          stale-issue-message: |
            This issue has been automatically marked as stale because it has not had 
            recent activity. It will be closed in 14 days if no further activity occurs.

            If this issue is still relevant:
            - Add a comment with an update
            - Remove the `stale` label

            Thank you for your contributions! 🙏

          stale-pr-message: |
            This PR has been automatically marked as stale. Please update it or it 
            will be closed in 14 days.

          days-before-stale: 60
          days-before-close: 14
          stale-issue-label: "stale"
          stale-pr-label: "stale"
          exempt-issue-labels: "pinned,security,in-progress"
          exempt-pr-labels: "pinned,security"
```

---

## 3. CI/CD Integration

### 3.1 Smart Test Selection

```yaml
# .github/workflows/smart-tests.yml
name: Smart Test Selection

on:
  pull_request:

jobs:
  analyze:
    runs-on: ubuntu-latest
    outputs:
      test_suites: ${{ steps.analyze.outputs.suites }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Analyze changes
        id: analyze
        run: |
          # Get changed files
          changed=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)

          # Determine which test suites to run
          suites="[]"

          if echo "$changed" | grep -q "^src/api/"; then
            suites=$(echo $suites | jq '. + ["api"]')
          fi

          if echo "$changed" | grep -q "^src/frontend/"; then
            suites=$(echo $suites | jq '. + ["frontend"]')
          fi

          if echo "$changed" | grep -q "^src/database/"; then
            suites=$(echo $suites | jq '. + ["database", "api"]')
          fi

          # If nothing specific, run all
          if [ "$suites" = "[]" ]; then
            suites='["all"]'
          fi

          echo "suites=$suites" >> $GITHUB_OUTPUT

  test:
    needs: analyze
    runs-on: ubuntu-latest
    strategy:
      matrix:
        suite: ${{ fromJson(needs.analyze.outputs.test_suites) }}

    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: |
          if [ "${{ matrix.suite }}" = "all" ]; then
            npm test
          else
            npm test -- --suite ${{ matrix.suite }}
          fi
```

### 3.2 Deployment with AI Validation

```yaml
# .github/workflows/deploy.yml
name: Deploy with AI Validation

on:
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get deployment changes
        id: changes
        run: |
          # Get commits since last deployment
          last_deploy=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          if [ -n "$last_deploy" ]; then
            changes=$(git log --oneline $last_deploy..HEAD)
          else
            changes=$(git log --oneline -10)
          fi
          echo "changes<<EOF" >> $GITHUB_OUTPUT
          echo "$changes" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: AI Risk Assessment
        id: assess
        uses: actions/github-script@v7
        with:
          script: |
            // Analyze changes for deployment risk
            const prompt = `
            Analyze these changes for deployment risk:

            ${process.env.CHANGES}

            Return JSON:
            {
              "riskLevel": "low" | "medium" | "high",
              "concerns": ["concern1", "concern2"],
              "recommendations": ["rec1", "rec2"],
              "requiresManualApproval": boolean
            }
            `;

            // Call AI and parse response
            const analysis = await callAI(prompt);

            if (analysis.riskLevel === 'high') {
              core.setFailed('High-risk deployment detected. Manual review required.');
            }

            return analysis;
        env:
          CHANGES: ${{ steps.changes.outputs.changes }}

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        run: |
          echo "Deploying to production..."
          # Deployment commands here
```

### 3.3 Rollback Automation

```yaml
# .github/workflows/rollback.yml
name: Automated Rollback

on:
  workflow_dispatch:
    inputs:
      reason:
        description: "Reason for rollback"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Find last stable version
        id: stable
        run: |
          # Find last successful deployment
          stable=$(git tag -l 'v*' --sort=-version:refname | head -1)
          echo "version=$stable" >> $GITHUB_OUTPUT

      - name: Rollback
        run: |
          git checkout ${{ steps.stable.outputs.version }}
          # Deploy stable version
          npm run deploy

      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🔄 Production rolled back to ${{ steps.stable.outputs.version }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Rollback executed*\n• Version: `${{ steps.stable.outputs.version }}`\n• Reason: ${{ inputs.reason }}\n• Triggered by: ${{ github.actor }}"
                  }
                }
              ]
            }
```

---

## 4. Git Operations

### 4.1 Automated Rebasing

```yaml
# .github/workflows/auto-rebase.yml
name: Auto Rebase

on:
  issue_comment:
    types: [created]

jobs:
  rebase:
    if: github.event.issue.pull_request && contains(github.event.comment.body, '/rebase')
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Rebase PR
        run: |
          # Fetch PR branch
          gh pr checkout ${{ github.event.issue.number }}

          # Rebase onto main
          git fetch origin main
          git rebase origin/main

          # Force push
          git push --force-with-lease
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Comment result
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '✅ Successfully rebased onto main!'
            })
```

### 4.2 Smart Cherry-Pick

```typescript
// AI-assisted cherry-pick that handles conflicts
async function smartCherryPick(commitHash: string, targetBranch: string) {
  // Get commit info
  const commitInfo = await exec(`git show ${commitHash} --stat`);

  // Check for potential conflicts
  const targetDiff = await exec(
    `git diff ${targetBranch}...HEAD -- ${affectedFiles}`
  );

  // AI analysis
  const analysis = await ai.analyze(`
    I need to cherry-pick this commit to ${targetBranch}:
    
    ${commitInfo}
    
    Current state of affected files on ${targetBranch}:
    ${targetDiff}
    
    Will there be conflicts? If so, suggest resolution strategy.
  `);

  if (analysis.willConflict) {
    // Create branch for manual resolution
    await exec(
      `git checkout -b cherry-pick-${commitHash.slice(0, 7)} ${targetBranch}`
    );
    const result = await exec(`git cherry-pick ${commitHash}`, {
      allowFail: true,
    });

    if (result.failed) {
      // AI-assisted conflict resolution
      const conflicts = await getConflicts();
      for (const conflict of conflicts) {
        const resolution = await ai.resolveConflict(conflict);
        await applyResolution(conflict.file, resolution);
      }
    }
  } else {
    await exec(`git checkout ${targetBranch}`);
    await exec(`git cherry-pick ${commitHash}`);
  }
}
```

### 4.3 Branch Cleanup

```yaml
# .github/workflows/branch-cleanup.yml
name: Branch Cleanup

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Find stale branches
        id: stale
        run: |
          # Branches not updated in 30 days
          stale=$(git for-each-ref --sort=-committerdate refs/remotes/origin \
            --format='%(refname:short) %(committerdate:relative)' | \
            grep -E '[3-9][0-9]+ days|[0-9]+ months|[0-9]+ years' | \
            grep -v 'origin/main\|origin/develop' | \
            cut -d' ' -f1 | sed 's|origin/||')

          echo "branches<<EOF" >> $GITHUB_OUTPUT
          echo "$stale" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create cleanup PR
        if: steps.stale.outputs.branches != ''
        uses: actions/github-script@v7
        with:
          script: |
            const branches = `${{ steps.stale.outputs.branches }}`.split('\n').filter(Boolean);

            const body = `## 🧹 Stale Branch Cleanup

The following branches haven't been updated in over 30 days:

${branches.map(b => `- \`${b}\``).join('\n')}

### Actions:
- [ ] Review each branch
- [ ] Delete branches that are no longer needed
- Comment \`/keep branch-name\` to preserve specific branches
`;

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Stale Branch Cleanup',
              body: body,
              labels: ['housekeeping']
            });
```

---

## 5. On-Demand Assistance

### 5.1 @mention Bot

```yaml
# .github/workflows/mention-bot.yml
name: AI Mention Bot

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  respond:
    if: contains(github.event.comment.body, '@ai-helper')
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Extract question
        id: question
        run: |
          # Extract text after @ai-helper
          question=$(echo "${{ github.event.comment.body }}" | sed 's/.*@ai-helper//')
          echo "question=$question" >> $GITHUB_OUTPUT

      - name: Get context
        id: context
        run: |
          if [ "${{ github.event.issue.pull_request }}" != "" ]; then
            # It's a PR - get diff
            gh pr diff ${{ github.event.issue.number }} > context.txt
          else
            # It's an issue - get description
            gh issue view ${{ github.event.issue.number }} --json body -q .body > context.txt
          fi
          echo "context=$(cat context.txt)" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: AI Response
        uses: actions/github-script@v7
        with:
          script: |
            const response = await ai.chat(`
              Context: ${process.env.CONTEXT}
              
              Question: ${process.env.QUESTION}
              
              Provide a helpful, specific answer. Include code examples if relevant.
            `);

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: response
            });
        env:
          CONTEXT: ${{ steps.context.outputs.context }}
          QUESTION: ${{ steps.question.outputs.question }}
```

### 5.2 Command Patterns

```markdown
## Available Commands

| Command              | Description                 |
| :------------------- | :-------------------------- |
| `@ai-helper explain` | Explain the code in this PR |
| `@ai-helper review`  | Request AI code review      |
| `@ai-helper fix`     | Suggest fixes for issues    |
| `@ai-helper test`    | Generate test cases         |
| `@ai-helper docs`    | Generate documentation      |
| `/rebase`            | Rebase PR onto main         |
| `/update`            | Update PR branch from main  |
| `/approve`           | Mark as approved by bot     |
| `/label bug`         | Add 'bug' label             |
| `/assign @user`      | Assign to user              |
```

---

## 6. Repository Configuration

### 6.1 CODEOWNERS

```
# .github/CODEOWNERS

# Global owners
* @org/core-team

# Frontend
/src/frontend/ @org/frontend-team
*.tsx @org/frontend-team
*.css @org/frontend-team

# Backend
/src/api/ @org/backend-team
/src/database/ @org/backend-team

# Infrastructure
/.github/ @org/devops-team
/terraform/ @org/devops-team
Dockerfile @org/devops-team

# Docs
/docs/ @org/docs-team
*.md @org/docs-team

# Security-sensitive
/src/auth/ @org/security-team
/src/crypto/ @org/security-team
```

### 6.2 Branch Protection

```yaml
# Set up via GitHub API
- name: Configure branch protection
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.repos.updateBranchProtection({
        owner: context.repo.owner,
        repo: context.repo.repo,
        branch: 'main',
        required_status_checks: {
          strict: true,
          contexts: ['test', 'lint', 'ai-review']
        },
        enforce_admins: true,
        required_pull_request_reviews: {
          required_approving_review_count: 1,
          require_code_owner_reviews: true,
          dismiss_stale_reviews: true
        },
        restrictions: null,
        required_linear_history: true,
        allow_force_pushes: false,
        allow_deletions: false
      });
```

---

## Best Practices

### Security

- [ ] Store API keys in GitHub Secrets
- [ ] Use minimal permissions in workflows
- [ ] Validate all inputs
- [ ] Don't expose sensitive data in logs

### Performance

- [ ] Cache dependencies
- [ ] Use matrix builds for parallel testing
- [ ] Skip unnecessary jobs with path filters
- [ ] Use self-hosted runners for heavy workloads

### Reliability

- [ ] Add timeouts to jobs
- [ ] Handle rate limits gracefully
- [ ] Implement retry logic
- [ ] Have rollback procedures

---

## Resources

- Agent CLI GitHub Action pattern for repository automation
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub REST API](https://docs.github.com/en/rest)
- [CODEOWNERS Syntax](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

---

## Merged Reference (legacy variant)

---
name: gitlab-automation
description: "Automate GitLab project management, issues, merge requests, pipelines, branches, and user operations via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# GitLab Automation via Rube MCP

Automate GitLab operations including project management, issue tracking, merge request workflows, CI/CD pipeline monitoring, branch management, and user administration through Composio's GitLab toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active GitLab connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `gitlab`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `gitlab`
3. If connection is not ACTIVE, follow the returned auth link to complete GitLab OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Issues

**When to use**: User wants to create, update, list, or search issues in a GitLab project

**Tool sequence**:
1. `GITLAB_GET_PROJECTS` - Find the target project and get its ID [Prerequisite]
2. `GITLAB_LIST_PROJECT_ISSUES` - List and filter issues for a project [Required]
3. `GITLAB_CREATE_PROJECT_ISSUE` - Create a new issue [Required for create]
4. `GITLAB_UPDATE_PROJECT_ISSUE` - Update an existing issue (title, labels, state, assignees) [Required for update]
5. `GITLAB_LIST_PROJECT_USERS` - Find user IDs for assignment [Optional]

**Key parameters**:
- `id`: Project ID (integer) or URL-encoded path (e.g., `"my-group/my-project"`)
- `title`: Issue title (required for creation)
- `description`: Issue body text (max 1,048,576 characters)
- `labels`: Comma-separated label names (e.g., `"bug,critical"`)
- `add_labels` / `remove_labels`: Add or remove labels without replacing all
- `state`: Filter by `"all"`, `"opened"`, or `"closed"`
- `state_event`: `"close"` or `"reopen"` to change issue state
- `assignee_ids`: Array of user IDs; use `[0]` to unassign all
- `issue_iid`: Internal issue ID within the project (required for updates)
- `milestone`: Filter by milestone title
- `search`: Search in title and description
- `scope`: `"created_by_me"`, `"assigned_to_me"`, or `"all"`
- `page` / `per_page`: Pagination (default per_page: 20)

**Pitfalls**:
- `id` accepts either integer project ID or URL-encoded path; wrong IDs yield 4xx errors
- `issue_iid` is the project-internal ID (shown as #42), different from the global issue ID
- Labels in `labels` field replace ALL existing labels; use `add_labels`/`remove_labels` for incremental changes
- Setting `assignee_ids` to empty array does NOT unassign; use `[0]` instead
- `updated_at` field requires administrator or project/group owner rights

### 2. Manage Merge Requests

**When to use**: User wants to list, filter, or review merge requests in a project

**Tool sequence**:
1. `GITLAB_GET_PROJECT` - Get project details and verify access [Prerequisite]
2. `GITLAB_GET_PROJECT_MERGE_REQUESTS` - List and filter merge requests [Required]
3. `GITLAB_GET_REPOSITORY_BRANCHES` - Verify source/target branches [Optional]
4. `GITLAB_LIST_ALL_PROJECT_MEMBERS` - Find reviewers/assignees [Optional]

**Key parameters**:
- `id`: Project ID or URL-encoded path
- `state`: `"opened"`, `"closed"`, `"locked"`, `"merged"`, or `"all"`
- `scope`: `"created_by_me"` (default), `"assigned_to_me"`, or `"all"`
- `source_branch` / `target_branch`: Filter by branch names
- `author_id` / `author_username`: Filter by MR author
- `assignee_id`: Filter by assignee (use `None` for unassigned, `Any` for assigned)
- `reviewer_id` / `reviewer_username`: Filter by reviewer
- `labels`: Comma-separated label filter
- `search`: Search in title and description
- `wip`: `"yes"` for draft MRs, `"no"` for non-draft
- `order_by`: `"created_at"` (default), `"title"`, `"merged_at"`, `"updated_at"`
- `view`: `"simple"` for minimal fields
- `iids[]`: Filter by specific MR internal IDs

**Pitfalls**:
- Default `scope` is `"created_by_me"` which limits results; use `"all"` for complete listings
- `author_id` and `author_username` are mutually exclusive
- `reviewer_id` and `reviewer_username` are mutually exclusive
- `approved` filter requires the `mr_approved_filter` feature flag (disabled by default)
- Large MR histories can be noisy; use filters and moderate `per_page` values

### 3. Manage Projects and Repositories

**When to use**: User wants to list projects, create new projects, or manage branches

**Tool sequence**:
1. `GITLAB_GET_PROJECTS` - List all accessible projects with filters [Required]
2. `GITLAB_GET_PROJECT` - Get detailed info for a specific project [Optional]
3. `GITLAB_LIST_USER_PROJECTS` - List projects owned by a specific user [Optional]
4. `GITLAB_CREATE_PROJECT` - Create a new project [Required for create]
5. `GITLAB_GET_REPOSITORY_BRANCHES` - List branches in a project [Required for branch ops]
6. `GITLAB_CREATE_REPOSITORY_BRANCH` - Create a new branch [Optional]
7. `GITLAB_GET_REPOSITORY_BRANCH` - Get details of a specific branch [Optional]
8. `GITLAB_LIST_REPOSITORY_COMMITS` - View commit history [Optional]
9. `GITLAB_GET_PROJECT_LANGUAGES` - Get language breakdown [Optional]

**Key parameters**:
- `name` / `path`: Project name and URL-friendly path (both required for creation)
- `visibility`: `"private"`, `"internal"`, or `"public"`
- `namespace_id`: Group or user ID for project placement
- `search`: Case-insensitive substring search for projects
- `membership`: `true` to limit to projects user is a member of
- `owned`: `true` to limit to user-owned projects
- `project_id`: Project ID for branch operations
- `branch_name`: Name for new branch
- `ref`: Source branch or commit SHA for new branch creation
- `order_by`: `"id"`, `"name"`, `"path"`, `"created_at"`, `"updated_at"`, `"star_count"`, `"last_activity_at"`

**Pitfalls**:
- `GITLAB_GET_PROJECTS` pagination is required for complete coverage; stopping at first page misses projects
- Some responses place items under `data.details`; parse the actual returned list structure
- Most follow-up calls depend on correct `project_id`; verify with `GITLAB_GET_PROJECT` first
- Invalid `branch_name`/`ref`/`sha` causes client errors; verify branch existence via `GITLAB_GET_REPOSITORY_BRANCHES` first
- Both `name` and `path` are required for `GITLAB_CREATE_PROJECT`

### 4. Monitor CI/CD Pipelines

**When to use**: User wants to check pipeline status, list jobs, or monitor CI/CD runs

**Tool sequence**:
1. `GITLAB_GET_PROJECT` - Verify project access [Prerequisite]
2. `GITLAB_LIST_PROJECT_PIPELINES` - List pipelines with filters [Required]
3. `GITLAB_GET_SINGLE_PIPELINE` - Get detailed info for a specific pipeline [Optional]
4. `GITLAB_LIST_PIPELINE_JOBS` - List jobs within a pipeline [Optional]

**Key parameters**:
- `id`: Project ID or URL-encoded path
- `status`: Filter by `"created"`, `"waiting_for_resource"`, `"preparing"`, `"pending"`, `"running"`, `"success"`, `"failed"`, `"canceled"`, `"skipped"`, `"manual"`, `"scheduled"`
- `scope`: `"running"`, `"pending"`, `"finished"`, `"branches"`, `"tags"`
- `ref`: Branch or tag name
- `sha`: Specific commit SHA
- `source`: Pipeline source (use `"parent_pipeline"` for child pipelines)
- `order_by`: `"id"` (default), `"status"`, `"ref"`, `"updated_at"`, `"user_id"`
- `created_after` / `created_before`: ISO 8601 date filters
- `pipeline_id`: Specific pipeline ID for job listing
- `include_retried`: `true` to include retried jobs (default `false`)

**Pitfalls**:
- Large pipeline histories can be noisy; use `status`, `ref`, and date filters to narrow results
- Use moderate `per_page` values to keep output manageable
- Pipeline job `scope` accepts single status string or array of statuses
- `yaml_errors: true` returns only pipelines with invalid configurations

### 5. Manage Users and Members

**When to use**: User wants to find users, list project members, or check user status

**Tool sequence**:
1. `GITLAB_GET_USERS` - Search and list GitLab users [Required]
2. `GITLAB_GET_USER` - Get details for a specific user by ID [Optional]
3. `GITLAB_GET_USERS_ID_STATUS` - Get user status message and availability [Optional]
4. `GITLAB_LIST_ALL_PROJECT_MEMBERS` - List all project members (direct + inherited) [Required for member listing]
5. `GITLAB_LIST_PROJECT_USERS` - List project users with search filter [Optional]

**Key parameters**:
- `search`: Search by name, username, or public email
- `username`: Get specific user by username
- `active` / `blocked`: Filter by user state
- `id`: Project ID for member listing
- `query`: Filter members by name, email, or username
- `state`: Filter members by `"awaiting"` or `"active"` (Premium/Ultimate)
- `user_ids`: Filter by specific user IDs

**Pitfalls**:
- Many user filters (admins, auditors, extern_uid, two_factor) are admin-only
- `GITLAB_LIST_ALL_PROJECT_MEMBERS` includes direct, inherited, and invited members
- User search is case-insensitive but may not match partial email domains
- Premium/Ultimate features (state filter, seat info) are not available on free plans

## Common Patterns

### ID Resolution
GitLab uses two identifier formats for projects:
- **Numeric ID**: Integer project ID (e.g., `123`)
- **URL-encoded path**: Namespace/project format (e.g., `"my-group%2Fmy-project"` or `"my-group/my-project"`)
- **Issue IID vs ID**: `issue_iid` is the project-internal number (#42); the global `id` is different
- **User ID**: Numeric; resolve via `GITLAB_GET_USERS` with `search` or `username`

### Pagination
GitLab uses offset-based pagination:
- Set `page` (starting at 1) and `per_page` (1-100, default 20)
- Continue incrementing `page` until response returns fewer items than `per_page` or is empty
- Total count may be available in response headers (`X-Total`, `X-Total-Pages`)
- Always paginate to completion for accurate results

### URL-Encoded Paths
When using project paths as identifiers:
- Forward slashes must be URL-encoded: `my-group/my-project` becomes `my-group%2Fmy-project`
- Some tools accept unencoded paths; check schema for each tool
- Prefer numeric IDs when available for reliability

## Known Pitfalls

### ID Formats
- Project `id` field accepts both integer and string (URL-encoded path)
- Issue `issue_iid` is project-scoped; do not confuse with global issue ID
- Pipeline IDs are project-scoped integers
- User IDs are global integers across the GitLab instance

### Rate Limits
- GitLab has per-user rate limits (typically 300-2000 requests/minute depending on plan)
- Large pipeline/issue histories should use date and status filters to reduce result sets
- Paginate responsibly with moderate `per_page` values

### Parameter Quirks
- `labels` field replaces ALL labels; use `add_labels`/`remove_labels` for incremental changes
- `assignee_ids: [0]` unassigns all; empty array does nothing
- `scope` defaults vary: `"created_by_me"` for MRs, `"all"` for issues
- `author_id` and `author_username` are mutually exclusive in MR filters
- Date parameters use ISO 8601 format: `"2024-01-15T10:30:00Z"`

### Plan Restrictions
- Some features require Premium/Ultimate: `epic_id`, `weight`, `iteration_id`, `approved_by_ids`, member `state` filter
- Admin-only features: user management filters, `updated_at` override, custom attributes
- The `mr_approved_filter` feature flag is disabled by default

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List projects | `GITLAB_GET_PROJECTS` | `search`, `membership`, `visibility` |
| Get project details | `GITLAB_GET_PROJECT` | `id` |
| User's projects | `GITLAB_LIST_USER_PROJECTS` | `id`, `search`, `owned` |
| Create project | `GITLAB_CREATE_PROJECT` | `name`, `path`, `visibility` |
| List issues | `GITLAB_LIST_PROJECT_ISSUES` | `id`, `state`, `labels`, `search` |
| Create issue | `GITLAB_CREATE_PROJECT_ISSUE` | `id`, `title`, `description`, `labels` |
| Update issue | `GITLAB_UPDATE_PROJECT_ISSUE` | `id`, `issue_iid`, `state_event` |
| List merge requests | `GITLAB_GET_PROJECT_MERGE_REQUESTS` | `id`, `state`, `scope`, `labels` |
| List branches | `GITLAB_GET_REPOSITORY_BRANCHES` | `project_id`, `search` |
| Get branch | `GITLAB_GET_REPOSITORY_BRANCH` | `project_id`, `branch_name` |
| Create branch | `GITLAB_CREATE_REPOSITORY_BRANCH` | `project_id`, `branch_name`, `ref` |
| List commits | `GITLAB_LIST_REPOSITORY_COMMITS` | project ID, branch ref |
| Project languages | `GITLAB_GET_PROJECT_LANGUAGES` | project ID |
| List pipelines | `GITLAB_LIST_PROJECT_PIPELINES` | `id`, `status`, `ref` |
| Get pipeline | `GITLAB_GET_SINGLE_PIPELINE` | `project_id`, `pipeline_id` |
| List pipeline jobs | `GITLAB_LIST_PIPELINE_JOBS` | `id`, `pipeline_id`, `scope` |
| Search users | `GITLAB_GET_USERS` | `search`, `username`, `active` |
| Get user | `GITLAB_GET_USER` | user ID |
| User status | `GITLAB_GET_USERS_ID_STATUS` | user ID |
| List project members | `GITLAB_LIST_ALL_PROJECT_MEMBERS` | `id`, `query`, `state` |
| List project users | `GITLAB_LIST_PROJECT_USERS` | `id`, `search` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: jira-automation
description: "Automate Jira tasks via Rube MCP (Composio): issues, projects, sprints, boards, comments, users. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Jira Automation via Rube MCP

Automate Jira operations through Composio's Jira toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Jira connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `jira`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `jira`
3. If connection is not ACTIVE, follow the returned auth link to complete Jira OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Search and Filter Issues

**When to use**: User wants to find issues using JQL or browse project issues

**Tool sequence**:
1. `JIRA_SEARCH_FOR_ISSUES_USING_JQL_POST` - Search with JQL query [Required]
2. `JIRA_GET_ISSUE` - Get full details of a specific issue [Optional]

**Key parameters**:
- `jql`: JQL query string (e.g., `project = PROJ AND status = "In Progress"`)
- `maxResults`: Max results per page (default 50, max 100)
- `startAt`: Pagination offset
- `fields`: Array of field names to return
- `issueIdOrKey`: Issue key like 'PROJ-123' for GET_ISSUE

**Pitfalls**:
- JQL field names are case-sensitive and must match Jira configuration
- Custom fields use IDs like `customfield_10001`, not display names
- Results are paginated; check `total` vs `startAt + maxResults` to continue

### 2. Create and Edit Issues

**When to use**: User wants to create new issues or update existing ones

**Tool sequence**:
1. `JIRA_GET_ALL_PROJECTS` - List projects to find project key [Prerequisite]
2. `JIRA_GET_FIELDS` - Get available fields and their IDs [Prerequisite]
3. `JIRA_CREATE_ISSUE` - Create a new issue [Required]
4. `JIRA_EDIT_ISSUE` - Update fields on an existing issue [Optional]
5. `JIRA_ASSIGN_ISSUE` - Assign issue to a user [Optional]

**Key parameters**:
- `project`: Project key (e.g., 'PROJ')
- `issuetype`: Issue type name (e.g., 'Bug', 'Story', 'Task')
- `summary`: Issue title
- `description`: Issue description (Atlassian Document Format or plain text)
- `issueIdOrKey`: Issue key for edits

**Pitfalls**:
- Issue types and required fields vary by project; use GET_FIELDS to check
- Custom fields require exact field IDs, not display names
- Description may need Atlassian Document Format (ADF) for rich content

### 3. Manage Sprints and Boards

**When to use**: User wants to work with agile boards, sprints, and backlogs

**Tool sequence**:
1. `JIRA_LIST_BOARDS` - List all boards [Prerequisite]
2. `JIRA_LIST_SPRINTS` - List sprints for a board [Required]
3. `JIRA_MOVE_ISSUE_TO_SPRINT` - Move issue to a sprint [Optional]
4. `JIRA_CREATE_SPRINT` - Create a new sprint [Optional]

**Key parameters**:
- `boardId`: Board ID from LIST_BOARDS
- `sprintId`: Sprint ID for move operations
- `name`: Sprint name for creation
- `startDate`/`endDate`: Sprint dates in ISO format

**Pitfalls**:
- Boards and sprints are specific to Jira Software (not Jira Core)
- Only one sprint can be active at a time per board

### 4. Manage Comments

**When to use**: User wants to add or view comments on issues

**Tool sequence**:
1. `JIRA_LIST_ISSUE_COMMENTS` - List existing comments [Optional]
2. `JIRA_ADD_COMMENT` - Add a comment to an issue [Required]

**Key parameters**:
- `issueIdOrKey`: Issue key like 'PROJ-123'
- `body`: Comment body (supports ADF for rich text)

**Pitfalls**:
- Comments support ADF (Atlassian Document Format) for formatting
- Mentions use account IDs, not usernames

### 5. Manage Projects and Users

**When to use**: User wants to list projects, find users, or manage project roles

**Tool sequence**:
1. `JIRA_GET_ALL_PROJECTS` - List all projects [Optional]
2. `JIRA_GET_PROJECT` - Get project details [Optional]
3. `JIRA_FIND_USERS` / `JIRA_GET_ALL_USERS` - Search for users [Optional]
4. `JIRA_GET_PROJECT_ROLES` - List project roles [Optional]
5. `JIRA_ADD_USERS_TO_PROJECT_ROLE` - Add user to role [Optional]

**Key parameters**:
- `projectIdOrKey`: Project key
- `query`: Search text for FIND_USERS
- `roleId`: Role ID for role operations

**Pitfalls**:
- User operations use account IDs (not email or display name)
- Project roles differ from global permissions

## Common Patterns

### JQL Syntax

**Common operators**:
- `project = "PROJ"` - Filter by project
- `status = "In Progress"` - Filter by status
- `assignee = currentUser()` - Current user's issues
- `created >= -7d` - Created in last 7 days
- `labels = "bug"` - Filter by label
- `priority = High` - Filter by priority
- `ORDER BY created DESC` - Sort results

**Combinators**:
- `AND` - Both conditions
- `OR` - Either condition
- `NOT` - Negate condition

### Pagination

- Use `startAt` and `maxResults` parameters
- Check `total` in response to determine remaining pages
- Continue until `startAt + maxResults >= total`

## Known Pitfalls

**Field Names**:
- Custom fields use IDs like `customfield_10001`
- Use JIRA_GET_FIELDS to discover field IDs and names
- Field names in JQL may differ from API field names

**Authentication**:
- Jira Cloud uses account IDs, not usernames
- Site URL must be configured correctly in the connection

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| Search issues (JQL) | JIRA_SEARCH_FOR_ISSUES_USING_JQL_POST | jql, maxResults |
| Get issue | JIRA_GET_ISSUE | issueIdOrKey |
| Create issue | JIRA_CREATE_ISSUE | project, issuetype, summary |
| Edit issue | JIRA_EDIT_ISSUE | issueIdOrKey, fields |
| Assign issue | JIRA_ASSIGN_ISSUE | issueIdOrKey, accountId |
| Add comment | JIRA_ADD_COMMENT | issueIdOrKey, body |
| List comments | JIRA_LIST_ISSUE_COMMENTS | issueIdOrKey |
| List projects | JIRA_GET_ALL_PROJECTS | (none) |
| Get project | JIRA_GET_PROJECT | projectIdOrKey |
| List boards | JIRA_LIST_BOARDS | (none) |
| List sprints | JIRA_LIST_SPRINTS | boardId |
| Move to sprint | JIRA_MOVE_ISSUE_TO_SPRINT | sprintId, issues |
| Create sprint | JIRA_CREATE_SPRINT | name, boardId |
| Find users | JIRA_FIND_USERS | query |
| Get fields | JIRA_GET_FIELDS | (none) |
| List filters | JIRA_LIST_FILTERS | (none) |
| Project roles | JIRA_GET_PROJECT_ROLES | projectIdOrKey |
| Project versions | JIRA_GET_PROJECT_VERSIONS | projectIdOrKey |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: linear-automation
description: "Automate Linear tasks via Rube MCP (Composio): issues, projects, cycles, teams, labels. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Linear Automation via Rube MCP

Automate Linear operations through Composio's Linear toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Linear connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `linear`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `linear`
3. If connection is not ACTIVE, follow the returned auth link to complete Linear OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Issues

**When to use**: User wants to create, search, update, or list Linear issues

**Tool sequence**:
1. `LINEAR_GET_ALL_LINEAR_TEAMS` - Get team IDs [Prerequisite]
2. `LINEAR_LIST_LINEAR_STATES` - Get workflow states for a team [Prerequisite]
3. `LINEAR_CREATE_LINEAR_ISSUE` - Create a new issue [Optional]
4. `LINEAR_SEARCH_ISSUES` / `LINEAR_LIST_LINEAR_ISSUES` - Find issues [Optional]
5. `LINEAR_GET_LINEAR_ISSUE` - Get issue details [Optional]
6. `LINEAR_UPDATE_ISSUE` - Update issue properties [Optional]

**Key parameters**:
- `team_id`: Team ID (required for creation)
- `title`: Issue title
- `description`: Issue description (Markdown supported)
- `state_id`: Workflow state ID
- `assignee_id`: Assignee user ID
- `priority`: 0 (none), 1 (urgent), 2 (high), 3 (medium), 4 (low)
- `label_ids`: Array of label IDs

**Pitfalls**:
- Team ID is required when creating issues; use GET_ALL_LINEAR_TEAMS first
- State IDs are team-specific; use LIST_LINEAR_STATES with the correct team
- Priority uses integer values 0-4, not string names

### 2. Manage Projects

**When to use**: User wants to create or update Linear projects

**Tool sequence**:
1. `LINEAR_LIST_LINEAR_PROJECTS` - List existing projects [Optional]
2. `LINEAR_CREATE_LINEAR_PROJECT` - Create a new project [Optional]
3. `LINEAR_UPDATE_LINEAR_PROJECT` - Update project details [Optional]

**Key parameters**:
- `name`: Project name
- `description`: Project description
- `team_ids`: Array of team IDs associated with the project
- `state`: Project state (e.g., 'planned', 'started', 'completed')

**Pitfalls**:
- Projects span teams; they can be associated with multiple teams

### 3. Manage Cycles

**When to use**: User wants to work with Linear cycles (sprints)

**Tool sequence**:
1. `LINEAR_GET_ALL_LINEAR_TEAMS` - Get team ID [Prerequisite]
2. `LINEAR_GET_CYCLES_BY_TEAM_ID` / `LINEAR_LIST_LINEAR_CYCLES` - List cycles [Required]

**Key parameters**:
- `team_id`: Team ID for cycle operations
- `number`: Cycle number

**Pitfalls**:
- Cycles are team-specific; always scope by team_id

### 4. Manage Labels and Comments

**When to use**: User wants to create labels or comment on issues

**Tool sequence**:
1. `LINEAR_CREATE_LINEAR_LABEL` - Create a new label [Optional]
2. `LINEAR_CREATE_LINEAR_COMMENT` - Comment on an issue [Optional]
3. `LINEAR_UPDATE_LINEAR_COMMENT` - Edit a comment [Optional]

**Key parameters**:
- `name`: Label name
- `color`: Label color (hex)
- `issue_id`: Issue ID for comments
- `body`: Comment body (Markdown)

**Pitfalls**:
- Labels can be team-scoped or workspace-scoped
- Comment body supports Markdown formatting

### 5. Custom GraphQL Queries

**When to use**: User needs advanced queries not covered by standard tools

**Tool sequence**:
1. `LINEAR_RUN_QUERY_OR_MUTATION` - Execute custom GraphQL [Required]

**Key parameters**:
- `query`: GraphQL query or mutation string
- `variables`: Variables for the query

**Pitfalls**:
- Requires knowledge of Linear's GraphQL schema
- Rate limits apply to GraphQL queries

## Common Patterns

### ID Resolution

**Team name -> Team ID**:
```
1. Call LINEAR_GET_ALL_LINEAR_TEAMS
2. Find team by name in response
3. Extract id field
```

**State name -> State ID**:
```
1. Call LINEAR_LIST_LINEAR_STATES with team_id
2. Find state by name
3. Extract id field
```

### Pagination

- Linear tools return paginated results
- Check for pagination cursors in responses
- Pass cursor to next request for additional pages

## Known Pitfalls

**Team Scoping**:
- Issues, states, and cycles are team-specific
- Always resolve team_id before creating issues

**Priority Values**:
- 0 = No priority, 1 = Urgent, 2 = High, 3 = Medium, 4 = Low
- Use integer values, not string names

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List teams | LINEAR_GET_ALL_LINEAR_TEAMS | (none) |
| Create issue | LINEAR_CREATE_LINEAR_ISSUE | team_id, title, description |
| Search issues | LINEAR_SEARCH_ISSUES | query |
| List issues | LINEAR_LIST_LINEAR_ISSUES | team_id, filters |
| Get issue | LINEAR_GET_LINEAR_ISSUE | issue_id |
| Update issue | LINEAR_UPDATE_ISSUE | issue_id, fields |
| List states | LINEAR_LIST_LINEAR_STATES | team_id |
| List projects | LINEAR_LIST_LINEAR_PROJECTS | (none) |
| Create project | LINEAR_CREATE_LINEAR_PROJECT | name, team_ids |
| Update project | LINEAR_UPDATE_LINEAR_PROJECT | project_id, fields |
| List cycles | LINEAR_LIST_LINEAR_CYCLES | team_id |
| Get cycles | LINEAR_GET_CYCLES_BY_TEAM_ID | team_id |
| Create label | LINEAR_CREATE_LINEAR_LABEL | name, color |
| Create comment | LINEAR_CREATE_LINEAR_COMMENT | issue_id, body |
| Update comment | LINEAR_UPDATE_LINEAR_COMMENT | comment_id, body |
| List users | LINEAR_LIST_LINEAR_USERS | (none) |
| Current user | LINEAR_GET_CURRENT_USER | (none) |
| Run GraphQL | LINEAR_RUN_QUERY_OR_MUTATION | query, variables |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: monday-automation
description: "Automate Monday.com work management including boards, items, columns, groups, subitems, and updates via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Monday.com Automation via Rube MCP

Automate Monday.com work management workflows including board creation, item management, column value updates, group organization, subitems, and update/comment threads through Composio's Monday toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Monday.com connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `monday`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `monday`
3. If connection is not ACTIVE, follow the returned auth link to complete Monday.com OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Boards

**When to use**: User wants to create a new board, list existing boards, or set up workspace structure.

**Tool sequence**:
1. `MONDAY_GET_WORKSPACES` - List available workspaces and resolve workspace ID [Prerequisite]
2. `MONDAY_LIST_BOARDS` - List existing boards to check for duplicates [Optional]
3. `MONDAY_CREATE_BOARD` - Create a new board with name, kind, and workspace [Required]
4. `MONDAY_CREATE_COLUMN` - Add columns to the new board [Optional]
5. `MONDAY_CREATE_GROUP` - Add groups to organize items [Optional]
6. `MONDAY_BOARDS` - Retrieve detailed board metadata [Optional]

**Key parameters**:
- `board_name`: Name for the new board (required)
- `board_kind`: "public", "private", or "share" (required)
- `workspace_id`: Numeric workspace ID; omit for default workspace
- `folder_id`: Folder ID; must be within `workspace_id` if both provided
- `template_id`: ID of accessible template to clone

**Pitfalls**:
- `board_kind` is required and must be one of: "public", "private", "share"
- If both `workspace_id` and `folder_id` are provided, the folder must exist within that workspace
- `template_id` must reference a template the authenticated user can access
- Board IDs are large integers; always use the exact value from API responses

### 2. Create and Manage Items

**When to use**: User wants to add tasks/items to a board, list existing items, or move items between groups.

**Tool sequence**:
1. `MONDAY_LIST_BOARDS` - Resolve board name to board ID [Prerequisite]
2. `MONDAY_LIST_GROUPS` - List groups on the board to get group_id [Prerequisite]
3. `MONDAY_LIST_COLUMNS` - Get column IDs and types for setting values [Prerequisite]
4. `MONDAY_CREATE_ITEM` - Create a new item with name and column values [Required]
5. `MONDAY_LIST_BOARD_ITEMS` - List all items on the board [Optional]
6. `MONDAY_MOVE_ITEM_TO_GROUP` - Move an item to a different group [Optional]
7. `MONDAY_ITEMS_PAGE` - Paginated item retrieval with filtering [Optional]

**Key parameters**:
- `board_id`: Board ID (required, integer)
- `item_name`: Item name, max 256 characters (required)
- `group_id`: Group ID string to place the item in (optional)
- `column_values`: JSON object or string mapping column IDs to values

**Pitfalls**:
- `column_values` must use column IDs (not titles); get them from `MONDAY_LIST_COLUMNS`
- Column value formats vary by type: status uses `{"index": 0}` or `{"label": "Done"}`, date uses `{"date": "YYYY-MM-DD"}`, people uses `{"personsAndTeams": [{"id": 123, "kind": "person"}]}`
- `item_name` has a 256-character maximum
- Subitem boards are NOT supported by `MONDAY_CREATE_ITEM`; use GraphQL via `MONDAY_CREATE_OBJECT`

### 3. Update Item Column Values

**When to use**: User wants to change status, date, text, or other column values on existing items.

**Tool sequence**:
1. `MONDAY_LIST_COLUMNS` or `MONDAY_COLUMNS` - Get column IDs and types [Prerequisite]
2. `MONDAY_LIST_BOARD_ITEMS` or `MONDAY_ITEMS_PAGE` - Find the target item ID [Prerequisite]
3. `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` - Update text, status, or dropdown with a string value [Required]
4. `MONDAY_UPDATE_ITEM` - Update complex column types (timeline, people, date) with JSON [Required]

**Key parameters for MONDAY_CHANGE_SIMPLE_COLUMN_VALUE**:
- `board_id`: Board ID (integer, required)
- `item_id`: Item ID (integer, required)
- `column_id`: Column ID string (required)
- `value`: Simple string value (e.g., "Done", "Working on it")
- `create_labels_if_missing`: true to auto-create status/dropdown labels (default true)

**Key parameters for MONDAY_UPDATE_ITEM**:
- `board_id`: Board ID (integer, required)
- `item_id`: Item ID (integer, required)
- `column_id`: Column ID string (required)
- `value`: JSON object matching the column type schema
- `create_labels_if_missing`: false by default; set true for status/dropdown

**Pitfalls**:
- Use `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` for simple text/status/dropdown updates (string value)
- Use `MONDAY_UPDATE_ITEM` for complex types like timeline, people, date (JSON value)
- Column IDs are lowercase strings with underscores (e.g., "status_1", "date_2", "text"); get them from `MONDAY_LIST_COLUMNS`
- Status values can be set by label name ("Done") or index number ("1")
- `create_labels_if_missing` defaults differ: true for CHANGE_SIMPLE, false for UPDATE_ITEM

### 4. Work with Groups and Board Structure

**When to use**: User wants to organize items into groups, add columns, or inspect board structure.

**Tool sequence**:
1. `MONDAY_LIST_BOARDS` - Resolve board ID [Prerequisite]
2. `MONDAY_LIST_GROUPS` - List all groups on a board [Required]
3. `MONDAY_CREATE_GROUP` - Create a new group [Optional]
4. `MONDAY_LIST_COLUMNS` or `MONDAY_COLUMNS` - Inspect column structure [Required]
5. `MONDAY_CREATE_COLUMN` - Add a new column to the board [Optional]
6. `MONDAY_MOVE_ITEM_TO_GROUP` - Reorganize items across groups [Optional]

**Key parameters**:
- `board_id`: Board ID (required for all group/column operations)
- `group_name`: Name for new group (CREATE_GROUP)
- `column_type`: Must be a valid GraphQL enum token in snake_case (e.g., "status", "text", "long_text", "numbers", "date", "dropdown", "people")
- `title`: Column display title
- `defaults`: JSON string for status/dropdown labels, e.g., `'{"labels": ["To Do", "In Progress", "Done"]}'`

**Pitfalls**:
- `column_type` must be exact snake_case values; "person" is NOT valid, use "people"
- Group IDs are strings (e.g., "topics", "new_group_12345"), not integers
- `MONDAY_COLUMNS` accepts an array of `board_ids` and returns column metadata including settings
- `MONDAY_LIST_COLUMNS` is simpler and takes a single `board_id`

### 5. Manage Subitems and Updates

**When to use**: User wants to view subitems of a task or add comments/updates to items.

**Tool sequence**:
1. `MONDAY_LIST_BOARD_ITEMS` - Find parent item IDs [Prerequisite]
2. `MONDAY_LIST_SUBITEMS_BY_PARENT` - Retrieve subitems with column values [Required]
3. `MONDAY_CREATE_UPDATE` - Add a comment/update to an item [Optional]
4. `MONDAY_CREATE_OBJECT` - Create subitems via GraphQL mutation [Optional]

**Key parameters for MONDAY_LIST_SUBITEMS_BY_PARENT**:
- `parent_item_ids`: Array of parent item IDs (integer array, required)
- `include_column_values`: true to include column data (default true)
- `include_parent_fields`: true to include parent item info (default true)

**Key parameters for MONDAY_CREATE_OBJECT** (GraphQL):
- `query`: Full GraphQL mutation string
- `variables`: Optional variables object

**Pitfalls**:
- Subitems can only be queried through their parent items
- To create subitems, use `MONDAY_CREATE_OBJECT` with a `create_subitem` GraphQL mutation
- `MONDAY_CREATE_UPDATE` is for adding comments/updates to items (Monday's "updates" feature), not for modifying item values
- `MONDAY_CREATE_OBJECT` is a raw GraphQL endpoint; ensure correct mutation syntax

## Common Patterns

### ID Resolution
Always resolve display names to IDs before operations:
- **Board name -> board_id**: `MONDAY_LIST_BOARDS` and match by name
- **Group name -> group_id**: `MONDAY_LIST_GROUPS` with `board_id`
- **Column title -> column_id**: `MONDAY_LIST_COLUMNS` with `board_id`
- **Workspace name -> workspace_id**: `MONDAY_GET_WORKSPACES` and match by name
- **Item name -> item_id**: `MONDAY_LIST_BOARD_ITEMS` or `MONDAY_ITEMS_PAGE`

### Pagination
Monday.com uses cursor-based pagination for items:
- `MONDAY_ITEMS_PAGE` returns a `cursor` in the response for the next page
- Pass the `cursor` to the next call; `board_id` and `query_params` are ignored when cursor is provided
- Cursors are cached for 60 minutes
- Maximum `limit` is 500 per page
- `MONDAY_LIST_BOARDS` and `MONDAY_GET_WORKSPACES` use page-based pagination with `page` and `limit`

### Column Value Formatting
Different column types require different value formats:
- **Status**: `{"index": 0}` or `{"label": "Done"}` or simple string "Done"
- **Date**: `{"date": "YYYY-MM-DD"}`
- **People**: `{"personsAndTeams": [{"id": 123, "kind": "person"}]}`
- **Text/Numbers**: Plain string or number
- **Timeline**: `{"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"}`

## Known Pitfalls

### ID Formats
- Board IDs and item IDs are large integers (e.g., 1234567890)
- Group IDs are strings (e.g., "topics", "new_group_12345")
- Column IDs are short strings (e.g., "status_1", "date4", "text")
- Workspace IDs are integers

### Rate Limits
- Monday.com GraphQL API has complexity-based rate limits
- Large boards with many columns increase query complexity
- Use `limit` parameter to reduce items per request if hitting limits

### Parameter Quirks
- `column_type` for CREATE_COLUMN must be exact snake_case enum values; "people" not "person"
- `column_values` in CREATE_ITEM accepts both JSON string and object formats
- `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` auto-creates missing labels by default; `MONDAY_UPDATE_ITEM` does not
- `MONDAY_CREATE_OBJECT` is a raw GraphQL interface; use it for operations without dedicated tools (e.g., create_subitem, delete_item, archive_board)

### Response Structure
- Board items are returned as arrays with `id`, `name`, and `state` fields
- Column values include both raw `value` (JSON) and rendered `text` (display string)
- Subitems are nested under parent items and cannot be queried independently

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `MONDAY_GET_WORKSPACES` | `kind`, `state`, `limit` |
| Create workspace | `MONDAY_CREATE_WORKSPACE` | `name`, `kind` |
| List boards | `MONDAY_LIST_BOARDS` | `limit`, `page`, `state` |
| Create board | `MONDAY_CREATE_BOARD` | `board_name`, `board_kind`, `workspace_id` |
| Get board metadata | `MONDAY_BOARDS` | `board_ids`, `board_kind` |
| List groups | `MONDAY_LIST_GROUPS` | `board_id` |
| Create group | `MONDAY_CREATE_GROUP` | `board_id`, `group_name` |
| List columns | `MONDAY_LIST_COLUMNS` | `board_id` |
| Get column metadata | `MONDAY_COLUMNS` | `board_ids`, `column_types` |
| Create column | `MONDAY_CREATE_COLUMN` | `board_id`, `column_type`, `title` |
| Create item | `MONDAY_CREATE_ITEM` | `board_id`, `item_name`, `column_values` |
| List board items | `MONDAY_LIST_BOARD_ITEMS` | `board_id` |
| Paginated items | `MONDAY_ITEMS_PAGE` | `board_id`, `limit`, `query_params` |
| Update column (simple) | `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` | `board_id`, `item_id`, `column_id`, `value` |
| Update column (complex) | `MONDAY_UPDATE_ITEM` | `board_id`, `item_id`, `column_id`, `value` |
| Move item to group | `MONDAY_MOVE_ITEM_TO_GROUP` | `item_id`, `group_id` |
| List subitems | `MONDAY_LIST_SUBITEMS_BY_PARENT` | `parent_item_ids` |
| Add comment/update | `MONDAY_CREATE_UPDATE` | `item_id`, `body` |
| Raw GraphQL mutation | `MONDAY_CREATE_OBJECT` | `query`, `variables` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: trello-automation
description: "Automate Trello boards, cards, and workflows via Rube MCP (Composio). Create cards, manage lists, assign members, and search across boards programmatically."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Trello Automation via Rube MCP

Automate Trello board management, card creation, and team workflows through Composio's Rube MCP integration.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Trello connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `trello`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `trello`
3. If connection is not ACTIVE, follow the returned auth link to complete Trello auth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create a Card on a Board

**When to use**: User wants to add a new card/task to a Trello board

**Tool sequence**:
1. `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` - List boards to find target board ID [Prerequisite]
2. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get lists on board to find target list ID [Prerequisite]
3. `TRELLO_ADD_CARDS` - Create the card on the resolved list [Required]
4. `TRELLO_ADD_CARDS_CHECKLISTS_BY_ID_CARD` - Add a checklist to the card [Optional]
5. `TRELLO_ADD_CARDS_CHECKLIST_CHECK_ITEM_BY_ID_CARD_BY_ID_CHECKLIST` - Add items to the checklist [Optional]

**Key parameters**:
- `idList`: 24-char hex ID (NOT list name)
- `name`: Card title
- `desc`: Card description (supports Markdown)
- `pos`: Position ('top'/'bottom')
- `due`: Due date (ISO 8601 format)

**Pitfalls**:
- Store returned id (idCard) immediately; downstream checklist operations fail without it
- Checklist payload may be nested (data.data); extract idChecklist from inner object
- One API call per checklist item; large checklists can trigger rate limits

### 2. Manage Boards and Lists

**When to use**: User wants to view, browse, or restructure board layout

**Tool sequence**:
1. `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` - List all boards for the user [Required]
2. `TRELLO_GET_BOARDS_BY_ID_BOARD` - Get detailed board info [Required]
3. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get lists (columns) on the board [Optional]
4. `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD` - Get board members [Optional]
5. `TRELLO_GET_BOARDS_LABELS_BY_ID_BOARD` - Get labels on the board [Optional]

**Key parameters**:
- `idMember`: Use 'me' for authenticated user
- `filter`: 'open', 'starred', or 'all'
- `idBoard`: 24-char hex or 8-char shortLink (NOT board name)

**Pitfalls**:
- Some runs return boards under response.data.details[]—don't assume flat top-level array
- Lists may be nested under results[0].response.data.details—parse defensively
- ISO 8601 timestamps with trailing 'Z' must be parsed as timezone-aware

### 3. Move Cards Between Lists

**When to use**: User wants to change a card's status by moving it to another list

**Tool sequence**:
1. `TRELLO_GET_SEARCH` - Find the card by name or keyword [Prerequisite]
2. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get destination list ID [Prerequisite]
3. `TRELLO_UPDATE_CARDS_BY_ID_CARD` - Update card's idList to move it [Required]

**Key parameters**:
- `idCard`: Card ID from search
- `idList`: Destination list ID
- `pos`: Optional ordering within new list

**Pitfalls**:
- Search returns partial matches; verify card name before updating
- Moving doesn't update position within new list; set pos if ordering matters

### 4. Assign Members to Cards

**When to use**: User wants to assign team members to cards

**Tool sequence**:
1. `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD` - Get member IDs from the board [Prerequisite]
2. `TRELLO_ADD_CARDS_ID_MEMBERS_BY_ID_CARD` - Add a member to the card [Required]

**Key parameters**:
- `idCard`: Target card ID
- `value`: Member ID to assign

**Pitfalls**:
- UPDATE_CARDS_ID_MEMBERS replaces entire member list; use ADD_CARDS_ID_MEMBERS to append
- Member must have board permissions

### 5. Search and Filter Cards

**When to use**: User wants to find specific cards across boards

**Tool sequence**:
1. `TRELLO_GET_SEARCH` - Search by query string [Required]

**Key parameters**:
- `query`: Search string (supports board:, list:, label:, is:open/archived operators)
- `modelTypes`: Set to 'cards'
- `partial`: Set to 'true' for prefix matching

**Pitfalls**:
- Search indexing has delay; newly created cards may not appear for several minutes
- For exact name matching, use TRELLO_GET_BOARDS_CARDS_BY_ID_BOARD and filter locally
- Query uses word tokenization; common words may be ignored as stop words

### 6. Add Comments and Attachments

**When to use**: User wants to add context to an existing card

**Tool sequence**:
1. `TRELLO_ADD_CARDS_ACTIONS_COMMENTS_BY_ID_CARD` - Post a comment on the card [Required]
2. `TRELLO_ADD_CARDS_ATTACHMENTS_BY_ID_CARD` - Attach a file or URL [Optional]

**Key parameters**:
- `text`: Comment text (1-16384 chars, supports Markdown and @mentions)
- `url` OR `file`: Attachment source (not both)
- `name`: Attachment display name
- `mimeType`: File MIME type

**Pitfalls**:
- Comments don't support file attachments; use the attachment tool separately
- Attachment deletion is irreversible

## Common Patterns

### ID Resolution
Always resolve display names to IDs before operations:
- **Board name → Board ID**: `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` with idMember='me'
- **List name → List ID**: `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` with resolved board ID
- **Card name → Card ID**: `TRELLO_GET_SEARCH` with query string
- **Member name → Member ID**: `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD`

### Pagination
Most list endpoints return all items. For boards with 1000+ cards, use `limit` and `before` parameters on card listing endpoints.

### Rate Limits
300 requests per 10 seconds per token. Use `TRELLO_GET_BATCH` for bulk read operations to stay within limits.

## Known Pitfalls

- **ID Requirements**: Nearly every tool requires IDs, not display names. Always resolve names to IDs first.
- **Board ID Format**: Board IDs must be 24-char hex or 8-char shortLink. URL slugs like 'my-board' are NOT valid.
- **Search Delays**: Search indexing has delays; newly created/updated cards may not appear immediately.
- **Nested Responses**: Response data is often nested (data.data or data.details[]); parse defensively.
- **Rate Limiting**: 300 req/10s per token. Batch reads with TRELLO_GET_BATCH.

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List user's boards | TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER | idMember='me', filter='open' |
| Get board details | TRELLO_GET_BOARDS_BY_ID_BOARD | idBoard (24-char hex) |
| List board lists | TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD | idBoard |
| Create card | TRELLO_ADD_CARDS | idList, name, desc, pos, due |
| Update card | TRELLO_UPDATE_CARDS_BY_ID_CARD | idCard, idList (to move) |
| Search cards | TRELLO_GET_SEARCH | query, modelTypes='cards' |
| Add checklist | TRELLO_ADD_CARDS_CHECKLISTS_BY_ID_CARD | idCard, name |
| Add comment | TRELLO_ADD_CARDS_ACTIONS_COMMENTS_BY_ID_CARD | idCard, text |
| Assign member | TRELLO_ADD_CARDS_ID_MEMBERS_BY_ID_CARD | idCard, value (member ID) |
| Attach file/URL | TRELLO_ADD_CARDS_ATTACHMENTS_BY_ID_CARD | idCard, url OR file |
| Get board members | TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD | idBoard |
| Batch read | TRELLO_GET_BATCH | urls (comma-separated paths) |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

---

## Merged Reference (legacy variant)

---
name: wrike-automation
description: "Automate Wrike project management via Rube MCP (Composio): create tasks/folders, manage projects, assign work, and track progress. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Wrike Automation via Rube MCP

Automate Wrike project management operations through Composio's Wrike toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Wrike connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `wrike`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `wrike`
3. If connection is not ACTIVE, follow the returned auth link to complete Wrike OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Tasks

**When to use**: User wants to create, assign, or update tasks in Wrike

**Tool sequence**:
1. `WRIKE_GET_FOLDERS` - Find the target folder/project [Prerequisite]
2. `WRIKE_GET_ALL_CUSTOM_FIELDS` - Get custom field IDs if needed [Optional]
3. `WRIKE_CREATE_TASK` - Create a new task [Required]
4. `WRIKE_MODIFY_TASK` - Update task properties [Optional]

**Key parameters**:
- `folderId`: Parent folder ID where the task will be created
- `title`: Task title
- `description`: Task description (supports HTML)
- `responsibles`: Array of user IDs to assign
- `status`: 'Active', 'Completed', 'Deferred', 'Cancelled'
- `importance`: 'High', 'Normal', 'Low'
- `customFields`: Array of {id, value} objects
- `dates`: Object with type, start, due, duration

**Pitfalls**:
- folderId is required; tasks must belong to a folder
- responsibles requires Wrike user IDs, not emails or names
- Custom field IDs must be obtained from GET_ALL_CUSTOM_FIELDS
- priorityBefore and priorityAfter are mutually exclusive
- Status field may not be available on Team plan
- dates.start and dates.due use 'YYYY-MM-DD' format

### 2. Manage Folders and Projects

**When to use**: User wants to create, modify, or organize folders and projects

**Tool sequence**:
1. `WRIKE_GET_FOLDERS` - List existing folders [Required]
2. `WRIKE_CREATE_FOLDER` - Create a new folder/project [Optional]
3. `WRIKE_MODIFY_FOLDER` - Update folder properties [Optional]
4. `WRIKE_LIST_SUBFOLDERS_BY_FOLDER_ID` - List subfolders [Optional]
5. `WRIKE_DELETE_FOLDER` - Delete a folder permanently [Optional]

**Key parameters**:
- `folderId`: Parent folder ID for creation; target folder ID for modification
- `title`: Folder name
- `description`: Folder description
- `customItemTypeId`: Set to create as a project instead of a folder
- `shareds`: Array of user IDs or emails to share with
- `project`: Filter for projects (true) or folders (false) in GET_FOLDERS

**Pitfalls**:
- DELETE_FOLDER is permanent and removes ALL contents (tasks, subfolders, documents)
- Cannot modify rootFolderId or recycleBinId as parents
- Folder creation auto-shares with the creator
- customItemTypeId converts a folder into a project
- GET_FOLDERS with descendants=true returns folder tree (may be large)

### 3. Retrieve and Track Tasks

**When to use**: User wants to find tasks, check status, or monitor progress

**Tool sequence**:
1. `WRIKE_FETCH_ALL_TASKS` - List tasks with optional filters [Required]
2. `WRIKE_GET_TASK_BY_ID` - Get detailed info for a specific task [Optional]

**Key parameters**:
- `status`: Filter by task status ('Active', 'Completed', etc.)
- `dueDate`: Filter by due date range (start/end/equal)
- `fields`: Additional response fields to include
- `page_size`: Results per page (1-100)
- `taskId`: Specific task ID for detailed retrieval
- `resolve_user_names`: Auto-resolve user IDs to names (default true)

**Pitfalls**:
- FETCH_ALL_TASKS paginates at max 100 items per page
- dueDate filter supports 'equal', 'start', and 'end' fields
- Date format: 'yyyy-MM-dd' or 'yyyy-MM-ddTHH:mm:ss'
- GET_TASK_BY_ID returns read-only detailed information
- customFields are returned by default for single task queries

### 4. Launch Task Blueprints

**When to use**: User wants to create tasks from predefined templates

**Tool sequence**:
1. `WRIKE_LIST_TASK_BLUEPRINTS` - List available blueprints [Prerequisite]
2. `WRIKE_LIST_SPACE_TASK_BLUEPRINTS` - List blueprints in a specific space [Alternative]
3. `WRIKE_LAUNCH_TASK_BLUEPRINT_ASYNC` - Launch a blueprint [Required]

**Key parameters**:
- `task_blueprint_id`: ID of the blueprint to launch
- `title`: Title for the root task
- `parent_id`: Parent folder/project ID (OR super_task_id)
- `super_task_id`: Parent task ID (OR parent_id)
- `reschedule_date`: Target date for task rescheduling
- `reschedule_mode`: 'RescheduleStartDate' or 'RescheduleFinishDate'
- `entry_limit`: Max tasks to copy (1-250)

**Pitfalls**:
- Either parent_id or super_task_id is required, not both
- Blueprint launch is asynchronous; tasks may take time to appear
- reschedule_date requires reschedule_mode to be set
- entry_limit caps at 250 tasks/folders per blueprint launch
- copy_descriptions defaults to false; set true to include task descriptions

### 5. Manage Workspace and Members

**When to use**: User wants to manage spaces, members, or invitations

**Tool sequence**:
1. `WRIKE_GET_SPACE` - Get space details [Optional]
2. `WRIKE_GET_CONTACTS` - List workspace contacts/members [Optional]
3. `WRIKE_CREATE_INVITATION` - Invite a user to the workspace [Optional]
4. `WRIKE_DELETE_SPACE` - Delete a space permanently [Optional]

**Key parameters**:
- `spaceId`: Space identifier
- `email`: Email for invitation
- `role`: User role ('Admin', 'Regular User', 'External User')
- `firstName`/`lastName`: Invitee name

**Pitfalls**:
- DELETE_SPACE is irreversible and removes all space contents
- userTypeId and role/external are mutually exclusive in invitations
- Custom email subjects/messages require a paid Wrike plan
- GET_CONTACTS returns workspace-level contacts, not task-specific assignments

## Common Patterns

### Folder ID Resolution

```
1. Call WRIKE_GET_FOLDERS (optionally with project=true for projects only)
2. Navigate folder tree to find target
3. Extract folder id (e.g., 'IEAGKVLFK4IHGQOI')
4. Use as folderId in task/folder creation
```

### Custom Field Setup

```
1. Call WRIKE_GET_ALL_CUSTOM_FIELDS to get definitions
2. Find field by name, extract id and type
3. Format value according to type (text, dropdown, number, date)
4. Include as {id: 'FIELD_ID', value: 'VALUE'} in customFields array
```

### Task Assignment

```
1. Call WRIKE_GET_CONTACTS to find user IDs
2. Use user IDs in responsibles array when creating tasks
3. Or use addResponsibles/removeResponsibles when modifying tasks
```

### Pagination

- FETCH_ALL_TASKS: Use page_size (max 100) and check for more results
- GET_FOLDERS: Use nextPageToken when descendants=false and pageSize is set
- LIST_TASK_BLUEPRINTS: Use next_page_token and page_size (default 100)

## Known Pitfalls

**ID Formats**:
- Wrike IDs are opaque alphanumeric strings (e.g., 'IEAGTXR7I4IHGABC')
- Task IDs, folder IDs, space IDs, and user IDs all use this format
- Custom field IDs follow the same pattern
- Never guess IDs; always resolve from list/search operations

**Permissions**:
- Operations depend on user role and sharing settings
- Shared folders/tasks are visible only to shared users
- Admin operations require appropriate role
- Some features (custom statuses, billing types) are plan-dependent

**Deletion Safety**:
- DELETE_FOLDER removes ALL contents permanently
- DELETE_SPACE removes the entire space and contents
- Consider using MODIFY_FOLDER to move to recycle bin instead
- Restore from recycle bin is possible via MODIFY_FOLDER with restore=true

**Date Handling**:
- Dates use 'yyyy-MM-dd' format
- DateTime uses 'yyyy-MM-ddTHH:mm:ssZ' or with timezone offset
- Task dates include type ('Planned', 'Actual'), start, due, duration
- Duration is in minutes

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| Create task | WRIKE_CREATE_TASK | folderId, title, responsibles, status |
| Modify task | WRIKE_MODIFY_TASK | taskId, title, status, addResponsibles |
| Get task by ID | WRIKE_GET_TASK_BY_ID | taskId |
| Fetch all tasks | WRIKE_FETCH_ALL_TASKS | status, dueDate, page_size |
| Get folders | WRIKE_GET_FOLDERS | project, descendants |
| Create folder | WRIKE_CREATE_FOLDER | folderId, title |
| Modify folder | WRIKE_MODIFY_FOLDER | folderId, title, addShareds |
| Delete folder | WRIKE_DELETE_FOLDER | folderId |
| List subfolders | WRIKE_LIST_SUBFOLDERS_BY_FOLDER_ID | folderId |
| Get custom fields | WRIKE_GET_ALL_CUSTOM_FIELDS | (none) |
| List blueprints | WRIKE_LIST_TASK_BLUEPRINTS | limit, page_size |
| Launch blueprint | WRIKE_LAUNCH_TASK_BLUEPRINT_ASYNC | task_blueprint_id, title, parent_id |
| Get space | WRIKE_GET_SPACE | spaceId |
| Delete space | WRIKE_DELETE_SPACE | spaceId |
| Get contacts | WRIKE_GET_CONTACTS | (none) |
| Invite user | WRIKE_CREATE_INVITATION | email, role |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/asana-automation/SKILL.md

---
name: asana-automation
description: "Automate Asana tasks via Rube MCP (Composio): tasks, projects, sections, teams, workspaces. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Asana Automation via Rube MCP

Automate Asana operations through Composio's Asana toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Asana connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `asana`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `asana`
3. If connection is not ACTIVE, follow the returned auth link to complete Asana OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Tasks

**When to use**: User wants to create, search, list, or organize tasks

**Tool sequence**:
1. `ASANA_GET_MULTIPLE_WORKSPACES` - Get workspace ID [Prerequisite]
2. `ASANA_SEARCH_TASKS_IN_WORKSPACE` - Search tasks [Optional]
3. `ASANA_GET_TASKS_FROM_A_PROJECT` - List project tasks [Optional]
4. `ASANA_CREATE_A_TASK` - Create a new task [Optional]
5. `ASANA_GET_A_TASK` - Get task details [Optional]
6. `ASANA_CREATE_SUBTASK` - Create a subtask [Optional]
7. `ASANA_GET_TASK_SUBTASKS` - List subtasks [Optional]

**Key parameters**:
- `workspace`: Workspace GID (required for search/creation)
- `projects`: Array of project GIDs to add task to
- `name`: Task name
- `notes`: Task description
- `assignee`: Assignee (user GID or email)
- `due_on`: Due date (YYYY-MM-DD)

**Pitfalls**:
- Workspace GID is required for most operations; get it first
- Task GIDs are returned as strings, not integers
- Search is workspace-scoped, not project-scoped

### 2. Manage Projects and Sections

**When to use**: User wants to create projects, manage sections, or organize tasks

**Tool sequence**:
1. `ASANA_GET_WORKSPACE_PROJECTS` - List workspace projects [Optional]
2. `ASANA_GET_A_PROJECT` - Get project details [Optional]
3. `ASANA_CREATE_A_PROJECT` - Create a new project [Optional]
4. `ASANA_GET_SECTIONS_IN_PROJECT` - List sections [Optional]
5. `ASANA_CREATE_SECTION_IN_PROJECT` - Create a new section [Optional]
6. `ASANA_ADD_TASK_TO_SECTION` - Move task to section [Optional]
7. `ASANA_GET_TASKS_FROM_A_SECTION` - List tasks in section [Optional]

**Key parameters**:
- `project_gid`: Project GID
- `name`: Project or section name
- `workspace`: Workspace GID for creation
- `task`: Task GID for section assignment
- `section`: Section GID

**Pitfalls**:
- Projects belong to workspaces; workspace GID is needed for creation
- Sections are ordered within a project
- DUPLICATE_PROJECT creates a copy with optional task inclusion

### 3. Manage Teams and Users

**When to use**: User wants to list teams, team members, or workspace users

**Tool sequence**:
1. `ASANA_GET_TEAMS_IN_WORKSPACE` - List workspace teams [Optional]
2. `ASANA_GET_USERS_FOR_TEAM` - List team members [Optional]
3. `ASANA_GET_USERS_FOR_WORKSPACE` - List all workspace users [Optional]
4. `ASANA_GET_CURRENT_USER` - Get authenticated user [Optional]
5. `ASANA_GET_MULTIPLE_USERS` - Get multiple user details [Optional]

**Key parameters**:
- `workspace_gid`: Workspace GID
- `team_gid`: Team GID

**Pitfalls**:
- Users are workspace-scoped
- Team membership requires the team GID

### 4. Parallel Operations

**When to use**: User needs to perform bulk operations efficiently

**Tool sequence**:
1. `ASANA_SUBMIT_PARALLEL_REQUESTS` - Execute multiple API calls in parallel [Required]

**Key parameters**:
- `actions`: Array of action objects with method, path, and data

**Pitfalls**:
- Each action must be a valid Asana API call
- Failed individual requests do not roll back successful ones

## Common Patterns

### ID Resolution

**Workspace name -> GID**:
```
1. Call ASANA_GET_MULTIPLE_WORKSPACES
2. Find workspace by name
3. Extract gid field
```

**Project name -> GID**:
```
1. Call ASANA_GET_WORKSPACE_PROJECTS with workspace GID
2. Find project by name
3. Extract gid field
```

### Pagination

- Asana uses cursor-based pagination with `offset` parameter
- Check for `next_page` in response
- Pass `offset` from `next_page.offset` for next request

## Known Pitfalls

**GID Format**:
- All Asana IDs are strings (GIDs), not integers
- GIDs are globally unique identifiers

**Workspace Scoping**:
- Most operations require a workspace context
- Tasks, projects, and users are workspace-scoped

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | ASANA_GET_MULTIPLE_WORKSPACES | (none) |
| Search tasks | ASANA_SEARCH_TASKS_IN_WORKSPACE | workspace, text |
| Create task | ASANA_CREATE_A_TASK | workspace, name, projects |
| Get task | ASANA_GET_A_TASK | task_gid |
| Create subtask | ASANA_CREATE_SUBTASK | parent, name |
| List subtasks | ASANA_GET_TASK_SUBTASKS | task_gid |
| Project tasks | ASANA_GET_TASKS_FROM_A_PROJECT | project_gid |
| List projects | ASANA_GET_WORKSPACE_PROJECTS | workspace |
| Create project | ASANA_CREATE_A_PROJECT | workspace, name |
| Get project | ASANA_GET_A_PROJECT | project_gid |
| Duplicate project | ASANA_DUPLICATE_PROJECT | project_gid |
| List sections | ASANA_GET_SECTIONS_IN_PROJECT | project_gid |
| Create section | ASANA_CREATE_SECTION_IN_PROJECT | project_gid, name |
| Add to section | ASANA_ADD_TASK_TO_SECTION | section, task |
| Section tasks | ASANA_GET_TASKS_FROM_A_SECTION | section_gid |
| List teams | ASANA_GET_TEAMS_IN_WORKSPACE | workspace_gid |
| Team members | ASANA_GET_USERS_FOR_TEAM | team_gid |
| Workspace users | ASANA_GET_USERS_FOR_WORKSPACE | workspace_gid |
| Current user | ASANA_GET_CURRENT_USER | (none) |
| Parallel requests | ASANA_SUBMIT_PARALLEL_REQUESTS | actions |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/basecamp-automation/SKILL.md

---
name: basecamp-automation
description: "Automate Basecamp project management, to-dos, messages, people, and to-do list organization via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Basecamp Automation via Rube MCP

Automate Basecamp operations including project management, to-do list creation, task management, message board posting, people management, and to-do group organization through Composio's Basecamp toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Basecamp connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `basecamp`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `basecamp`
3. If connection is not ACTIVE, follow the returned auth link to complete Basecamp OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage To-Do Lists and Tasks

**When to use**: User wants to create to-do lists, add tasks, or organize work within a Basecamp project

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - List projects to find the target bucket_id [Prerequisite]
2. `BASECAMP_GET_BUCKETS_TODOSETS` - Get the to-do set within a project [Prerequisite]
3. `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS` - List existing to-do lists to avoid duplicates [Optional]
4. `BASECAMP_POST_BUCKETS_TODOSETS_TODOLISTS` - Create a new to-do list in a to-do set [Required for list creation]
5. `BASECAMP_GET_BUCKETS_TODOLISTS` - Get details of a specific to-do list [Optional]
6. `BASECAMP_POST_BUCKETS_TODOLISTS_TODOS` - Create a to-do item in a to-do list [Required for task creation]
7. `BASECAMP_CREATE_TODO` - Alternative tool for creating individual to-dos [Alternative]
8. `BASECAMP_GET_BUCKETS_TODOLISTS_TODOS` - List to-dos within a to-do list [Optional]

**Key parameters for creating to-do lists**:
- `bucket_id`: Integer project/bucket ID (from GET_PROJECTS)
- `todoset_id`: Integer to-do set ID (from GET_BUCKETS_TODOSETS)
- `name`: Title of the to-do list (required)
- `description`: HTML-formatted description (supports Rich text)

**Key parameters for creating to-dos**:
- `bucket_id`: Integer project/bucket ID
- `todolist_id`: Integer to-do list ID
- `content`: What the to-do is for (required)
- `description`: HTML details about the to-do
- `assignee_ids`: Array of integer person IDs
- `due_on`: Due date in `YYYY-MM-DD` format
- `starts_on`: Start date in `YYYY-MM-DD` format
- `notify`: Boolean to notify assignees (defaults to false)
- `completion_subscriber_ids`: Person IDs notified upon completion

**Pitfalls**:
- A project (bucket) can contain multiple to-do sets; selecting the wrong `todoset_id` creates lists in the wrong section
- Always check existing to-do lists before creating to avoid near-duplicate names
- Success payloads include user-facing URLs (`app_url`, `app_todos_url`); prefer returning these over raw IDs
- All IDs (`bucket_id`, `todoset_id`, `todolist_id`) are integers, not strings
- Descriptions support HTML formatting only, not Markdown

### 2. Post and Manage Messages

**When to use**: User wants to post messages to a project message board or update existing messages

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - Find the target project and bucket_id [Prerequisite]
2. `BASECAMP_GET_MESSAGE_BOARD` - Get the message board ID for the project [Prerequisite]
3. `BASECAMP_CREATE_MESSAGE` - Create a new message on the board [Required]
4. `BASECAMP_POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` - Alternative message creation tool [Fallback]
5. `BASECAMP_GET_MESSAGE` - Read a specific message by ID [Optional]
6. `BASECAMP_PUT_BUCKETS_MESSAGES` - Update an existing message [Optional]

**Key parameters**:
- `bucket_id`: Integer project/bucket ID
- `message_board_id`: Integer message board ID (from GET_MESSAGE_BOARD)
- `subject`: Message title (required)
- `content`: HTML body of the message
- `status`: Set to `"active"` to publish immediately
- `category_id`: Message type classification (optional)
- `subscriptions`: Array of person IDs to notify; omit to notify all project members

**Pitfalls**:
- `status="draft"` can produce HTTP 400; use `status="active"` as the reliable option
- `bucket_id` and `message_board_id` must belong to the same project; mismatches fail or misroute
- Message content supports HTML tags only; not Markdown
- Updates via `PUT_BUCKETS_MESSAGES` replace the entire body -- include the full corrected content, not just a diff
- Prefer `app_url` from the response for user-facing confirmation links
- Both `CREATE_MESSAGE` and `POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` do the same thing; use CREATE_MESSAGE first and fall back to POST if it fails

### 3. Manage People and Access

**When to use**: User wants to list people, manage project access, or add new users

**Tool sequence**:
1. `BASECAMP_GET_PEOPLE` - List all people visible to the current user [Required]
2. `BASECAMP_GET_PROJECTS` - Find the target project [Prerequisite]
3. `BASECAMP_LIST_PROJECT_PEOPLE` - List people on a specific project [Required]
4. `BASECAMP_GET_PROJECTS_PEOPLE` - Alternative to list project members [Alternative]
5. `BASECAMP_PUT_PROJECTS_PEOPLE_USERS` - Grant or revoke project access [Required for access changes]

**Key parameters for PUT_PROJECTS_PEOPLE_USERS**:
- `project_id`: Integer project ID
- `grant`: Array of integer person IDs to add to the project
- `revoke`: Array of integer person IDs to remove from the project
- `create`: Array of objects with `name`, `email_address`, and optional `company_name`, `title` for new users
- At least one of `grant`, `revoke`, or `create` must be provided

**Pitfalls**:
- Person IDs are integers; always resolve names to IDs via GET_PEOPLE first
- `project_id` for people management is the same as `bucket_id` for other operations
- `LIST_PROJECT_PEOPLE` and `GET_PROJECTS_PEOPLE` are near-identical; use either
- Creating users via `create` also grants them project access in one step

### 4. Organize To-Dos with Groups

**When to use**: User wants to organize to-dos within a list into color-coded groups

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - Find the target project [Prerequisite]
2. `BASECAMP_GET_BUCKETS_TODOLISTS` - Get the to-do list details [Prerequisite]
3. `BASECAMP_GET_TODOLIST_GROUPS` - List existing groups in a to-do list [Optional]
4. `BASECAMP_GET_BUCKETS_TODOLISTS_GROUPS` - Alternative group listing [Alternative]
5. `BASECAMP_POST_BUCKETS_TODOLISTS_GROUPS` - Create a new group in a to-do list [Required]
6. `BASECAMP_CREATE_TODOLIST_GROUP` - Alternative group creation tool [Alternative]

**Key parameters**:
- `bucket_id`: Integer project/bucket ID
- `todolist_id`: Integer to-do list ID
- `name`: Group title (required)
- `color`: Visual color identifier -- one of: `white`, `red`, `orange`, `yellow`, `green`, `blue`, `aqua`, `purple`, `gray`, `pink`, `brown`
- `status`: Filter for listing -- `"archived"` or `"trashed"` (omit for active groups)

**Pitfalls**:
- `POST_BUCKETS_TODOLISTS_GROUPS` and `CREATE_TODOLIST_GROUP` are near-identical; use either
- Color values must be from the fixed palette; arbitrary hex/rgb values are not supported
- Groups are sub-sections within a to-do list, not standalone entities

### 5. Browse and Inspect Projects

**When to use**: User wants to list projects, get project details, or explore project structure

**Tool sequence**:
1. `BASECAMP_GET_PROJECTS` - List all active projects [Required]
2. `BASECAMP_GET_PROJECT` - Get comprehensive details for a specific project [Optional]
3. `BASECAMP_GET_PROJECTS_BY_PROJECT_ID` - Alternative project detail retrieval [Alternative]

**Key parameters**:
- `status`: Filter by `"archived"` or `"trashed"`; omit for active projects
- `project_id`: Integer project ID for detailed retrieval

**Pitfalls**:
- Projects are sorted by most recently created first
- The response includes a `dock` array with tools (todoset, message_board, etc.) and their IDs
- Use the dock tool IDs to find `todoset_id`, `message_board_id`, etc. for downstream operations

## Common Patterns

### ID Resolution
Basecamp uses a hierarchical ID structure. Always resolve top-down:
- **Project (bucket_id)**: `BASECAMP_GET_PROJECTS` -- find by name, capture the `id`
- **To-do set (todoset_id)**: Found in project dock or via `BASECAMP_GET_BUCKETS_TODOSETS`
- **Message board (message_board_id)**: Found in project dock or via `BASECAMP_GET_MESSAGE_BOARD`
- **To-do list (todolist_id)**: `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS`
- **People (person_id)**: `BASECAMP_GET_PEOPLE` or `BASECAMP_LIST_PROJECT_PEOPLE`
- Note: `bucket_id` and `project_id` refer to the same entity in different contexts

### Pagination
Basecamp uses page-based pagination on list endpoints:
- Response headers or body may indicate more pages available
- `GET_PROJECTS`, `GET_BUCKETS_TODOSETS_TODOLISTS`, and list endpoints return paginated results
- Continue fetching until no more results are returned

### Content Formatting
- All rich text fields use HTML, not Markdown
- Wrap content in `<div>` tags; use `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>`, `<a>` etc.
- Example: `<div><strong>Important:</strong> Complete by Friday</div>`

## Known Pitfalls

### ID Formats
- All Basecamp IDs are integers, not strings or UUIDs
- `bucket_id` = `project_id` (same entity, different parameter names across tools)
- To-do set IDs, to-do list IDs, and message board IDs are found in the project's `dock` array
- Person IDs are integers; resolve names via `GET_PEOPLE` before operations

### Status Field
- `status="draft"` for messages can cause HTTP 400; always use `status="active"`
- Project/to-do list status filters: `"archived"`, `"trashed"`, or omit for active

### Content Format
- HTML only, never Markdown
- Updates replace the entire body, not a partial diff
- Invalid HTML tags may be silently stripped

### Rate Limits
- Basecamp API has rate limits; space out rapid sequential requests
- Large projects with many to-dos should be paginated carefully

### URL Handling
- Prefer `app_url` from API responses for user-facing links
- Do not reconstruct Basecamp URLs manually from IDs

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List projects | `BASECAMP_GET_PROJECTS` | `status` |
| Get project | `BASECAMP_GET_PROJECT` | `project_id` |
| Get project detail | `BASECAMP_GET_PROJECTS_BY_PROJECT_ID` | `project_id` |
| Get to-do set | `BASECAMP_GET_BUCKETS_TODOSETS` | `bucket_id`, `todoset_id` |
| List to-do lists | `BASECAMP_GET_BUCKETS_TODOSETS_TODOLISTS` | `bucket_id`, `todoset_id` |
| Get to-do list | `BASECAMP_GET_BUCKETS_TODOLISTS` | `bucket_id`, `todolist_id` |
| Create to-do list | `BASECAMP_POST_BUCKETS_TODOSETS_TODOLISTS` | `bucket_id`, `todoset_id`, `name` |
| Create to-do | `BASECAMP_POST_BUCKETS_TODOLISTS_TODOS` | `bucket_id`, `todolist_id`, `content` |
| Create to-do (alt) | `BASECAMP_CREATE_TODO` | `bucket_id`, `todolist_id`, `content` |
| List to-dos | `BASECAMP_GET_BUCKETS_TODOLISTS_TODOS` | `bucket_id`, `todolist_id` |
| List to-do groups | `BASECAMP_GET_TODOLIST_GROUPS` | `bucket_id`, `todolist_id` |
| Create to-do group | `BASECAMP_POST_BUCKETS_TODOLISTS_GROUPS` | `bucket_id`, `todolist_id`, `name`, `color` |
| Create to-do group (alt) | `BASECAMP_CREATE_TODOLIST_GROUP` | `bucket_id`, `todolist_id`, `name` |
| Get message board | `BASECAMP_GET_MESSAGE_BOARD` | `bucket_id`, `message_board_id` |
| Create message | `BASECAMP_CREATE_MESSAGE` | `bucket_id`, `message_board_id`, `subject`, `status` |
| Create message (alt) | `BASECAMP_POST_BUCKETS_MESSAGE_BOARDS_MESSAGES` | `bucket_id`, `message_board_id`, `subject` |
| Get message | `BASECAMP_GET_MESSAGE` | `bucket_id`, `message_id` |
| Update message | `BASECAMP_PUT_BUCKETS_MESSAGES` | `bucket_id`, `message_id` |
| List all people | `BASECAMP_GET_PEOPLE` | (none) |
| List project people | `BASECAMP_LIST_PROJECT_PEOPLE` | `project_id` |
| Manage access | `BASECAMP_PUT_PROJECTS_PEOPLE_USERS` | `project_id`, `grant`, `revoke`, `create` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/bitbucket-automation/SKILL.md

---
name: bitbucket-automation
description: "Automate Bitbucket repositories, pull requests, branches, issues, and workspace management via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Bitbucket Automation via Rube MCP

Automate Bitbucket operations including repository management, pull request workflows, branch operations, issue tracking, and workspace administration through Composio's Bitbucket toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Bitbucket connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `bitbucket`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `bitbucket`
3. If connection is not ACTIVE, follow the returned auth link to complete Bitbucket OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Pull Requests

**When to use**: User wants to create, review, or inspect pull requests

**Tool sequence**:
1. `BITBUCKET_LIST_WORKSPACES` - Discover accessible workspaces [Prerequisite]
2. `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` - Find the target repository [Prerequisite]
3. `BITBUCKET_LIST_BRANCHES` - Verify source and destination branches exist [Prerequisite]
4. `BITBUCKET_CREATE_PULL_REQUEST` - Create a new PR with title, source branch, and optional reviewers [Required]
5. `BITBUCKET_LIST_PULL_REQUESTS` - List PRs filtered by state (OPEN, MERGED, DECLINED) [Optional]
6. `BITBUCKET_GET_PULL_REQUEST` - Get full details of a specific PR by ID [Optional]
7. `BITBUCKET_GET_PULL_REQUEST_DIFF` - Fetch unified diff for code review [Optional]
8. `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` - Get changed files with lines added/removed [Optional]

**Key parameters**:
- `workspace`: Workspace slug or UUID (required for all operations)
- `repo_slug`: URL-friendly repository name
- `source_branch`: Branch with changes to merge
- `destination_branch`: Target branch (defaults to repo main branch if omitted)
- `reviewers`: List of objects with `uuid` field for reviewer assignment
- `state`: Filter for LIST_PULL_REQUESTS - `OPEN`, `MERGED`, or `DECLINED`
- `max_chars`: Truncation limit for GET_PULL_REQUEST_DIFF to handle large diffs

**Pitfalls**:
- `reviewers` expects an array of objects with `uuid` key, NOT usernames: `[{"uuid": "{...}"}]`
- UUID format must include curly braces: `{123e4567-e89b-12d3-a456-426614174000}`
- `destination_branch` defaults to the repo's main branch if omitted, which may not be `main`
- `pull_request_id` is an integer for GET/DIFF operations but comes back as part of PR listing
- Large diffs can overwhelm context; always set `max_chars` (e.g., 50000) on GET_PULL_REQUEST_DIFF

### 2. Manage Repositories and Workspaces

**When to use**: User wants to list, create, or delete repositories or explore workspaces

**Tool sequence**:
1. `BITBUCKET_LIST_WORKSPACES` - List all accessible workspaces [Required]
2. `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` - List repos with optional BBQL filtering [Required]
3. `BITBUCKET_CREATE_REPOSITORY` - Create a new repo with language, privacy, and project settings [Optional]
4. `BITBUCKET_DELETE_REPOSITORY` - Permanently delete a repository (irreversible) [Optional]
5. `BITBUCKET_LIST_WORKSPACE_MEMBERS` - List members for reviewer assignment or access checks [Optional]

**Key parameters**:
- `workspace`: Workspace slug (find via LIST_WORKSPACES)
- `repo_slug`: URL-friendly name for create/delete
- `q`: BBQL query filter (e.g., `name~"api"`, `project.key="PROJ"`, `is_private=true`)
- `role`: Filter repos by user role: `member`, `contributor`, `admin`, `owner`
- `sort`: Sort field with optional `-` prefix for descending (e.g., `-updated_on`)
- `is_private`: Boolean for repository visibility (defaults to `true`)
- `project_key`: Bitbucket project key; omit to use workspace's oldest project

**Pitfalls**:
- `BITBUCKET_DELETE_REPOSITORY` is **irreversible** and does not affect forks
- BBQL string values MUST be enclosed in double quotes: `name~"my-repo"` not `name~my-repo`
- `repository` is NOT a valid BBQL field; use `name` instead
- Default pagination is 10 results; set `pagelen` explicitly for complete listings
- `CREATE_REPOSITORY` defaults to private; set `is_private: false` for public repos

### 3. Manage Issues

**When to use**: User wants to create, update, list, or comment on repository issues

**Tool sequence**:
1. `BITBUCKET_LIST_ISSUES` - List issues with optional filters for state, priority, kind, assignee [Required]
2. `BITBUCKET_CREATE_ISSUE` - Create a new issue with title, content, priority, and kind [Required]
3. `BITBUCKET_UPDATE_ISSUE` - Modify issue attributes (state, priority, assignee, etc.) [Optional]
4. `BITBUCKET_CREATE_ISSUE_COMMENT` - Add a markdown comment to an existing issue [Optional]
5. `BITBUCKET_DELETE_ISSUE` - Permanently delete an issue [Optional]

**Key parameters**:
- `issue_id`: String identifier for the issue
- `title`, `content`: Required for creation
- `kind`: `bug`, `enhancement`, `proposal`, or `task`
- `priority`: `trivial`, `minor`, `major`, `critical`, or `blocker`
- `state`: `new`, `open`, `resolved`, `on hold`, `invalid`, `duplicate`, `wontfix`, `closed`
- `assignee`: Bitbucket username for CREATE; `assignee_account_id` (UUID) for UPDATE
- `due_on`: ISO 8601 format date string

**Pitfalls**:
- Issue tracker must be enabled on the repository (`has_issues: true`) or API calls will fail
- `CREATE_ISSUE` uses `assignee` (username string), but `UPDATE_ISSUE` uses `assignee_account_id` (UUID) -- they are different fields
- `DELETE_ISSUE` is permanent with no undo
- `state` values include spaces: `"on hold"` not `"on_hold"`
- Filtering by `assignee` in LIST_ISSUES uses account ID, not username; use `"null"` string for unassigned

### 4. Manage Branches

**When to use**: User wants to create branches or explore branch structure

**Tool sequence**:
1. `BITBUCKET_LIST_BRANCHES` - List branches with optional BBQL filter and sorting [Required]
2. `BITBUCKET_CREATE_BRANCH` - Create a new branch from a specific commit hash [Required]

**Key parameters**:
- `name`: Branch name without `refs/heads/` prefix (e.g., `feature/new-login`)
- `target_hash`: Full SHA1 commit hash to branch from (must exist in repo)
- `q`: BBQL filter (e.g., `name~"feature/"`, `name="main"`)
- `sort`: Sort by `name` or `-target.date` (descending commit date)
- `pagelen`: 1-100 results per page (default is 10)

**Pitfalls**:
- `CREATE_BRANCH` requires a full commit hash, NOT a branch name as `target_hash`
- Do NOT include `refs/heads/` prefix in branch names
- Branch names must follow Bitbucket naming conventions (alphanumeric, `/`, `.`, `_`, `-`)
- BBQL string values need double quotes: `name~"feature/"` not `name~feature/`

### 5. Review Pull Requests with Comments

**When to use**: User wants to add review comments to pull requests, including inline code comments

**Tool sequence**:
1. `BITBUCKET_GET_PULL_REQUEST` - Get PR details and verify it exists [Prerequisite]
2. `BITBUCKET_GET_PULL_REQUEST_DIFF` - Review the actual code changes [Prerequisite]
3. `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` - Get list of changed files [Optional]
4. `BITBUCKET_CREATE_PULL_REQUEST_COMMENT` - Post review comments [Required]

**Key parameters**:
- `pull_request_id`: String ID of the PR
- `content_raw`: Markdown-formatted comment text
- `content_markup`: Defaults to `markdown`; also supports `plaintext`
- `inline`: Object with `path`, `from`, `to` for inline code comments
- `parent_comment_id`: Integer ID for threaded replies to existing comments

**Pitfalls**:
- `pull_request_id` is a string in CREATE_PULL_REQUEST_COMMENT but an integer in GET_PULL_REQUEST
- Inline comments require `inline.path` at minimum; `from`/`to` are optional line numbers
- `parent_comment_id` creates a threaded reply; omit for top-level comments
- Line numbers in inline comments reference the diff, not the source file

## Common Patterns

### ID Resolution
Always resolve human-readable names to IDs before operations:
- **Workspace**: `BITBUCKET_LIST_WORKSPACES` to get workspace slugs
- **Repository**: `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` with `q` filter to find repo slugs
- **Branch**: `BITBUCKET_LIST_BRANCHES` to verify branch existence before PR creation
- **Members**: `BITBUCKET_LIST_WORKSPACE_MEMBERS` to get UUIDs for reviewer assignment

### Pagination
Bitbucket uses page-based pagination (not cursor-based):
- Use `page` (starts at 1) and `pagelen` (items per page) parameters
- Default page size is typically 10; set `pagelen` explicitly (max 50 for PRs, 100 for others)
- Check response for `next` URL or total count to determine if more pages exist
- Always iterate through all pages for complete results

### BBQL Filtering
Bitbucket Query Language is available on list endpoints:
- String values MUST use double quotes: `name~"pattern"`
- Operators: `=` (exact), `~` (contains), `!=` (not equal), `>`, `>=`, `<`, `<=`
- Combine with `AND` / `OR`: `name~"api" AND is_private=true`

## Known Pitfalls

### ID Formats
- Workspace: slug string (e.g., `my-workspace`) or UUID in braces (`{uuid}`)
- Reviewer UUIDs must include curly braces: `{123e4567-e89b-12d3-a456-426614174000}`
- Issue IDs are strings; PR IDs are integers in some tools, strings in others
- Commit hashes must be full SHA1 (40 characters)

### Parameter Quirks
- `assignee` vs `assignee_account_id`: CREATE_ISSUE uses username, UPDATE_ISSUE uses UUID
- `state` values for issues include spaces: `"on hold"`, not `"on_hold"`
- `destination_branch` omission defaults to repo main branch, not `main` literally
- BBQL `repository` is not a valid field -- use `name`

### Rate Limits
- Bitbucket Cloud API has rate limits; large batch operations should include delays
- Paginated requests count against rate limits; minimize unnecessary page fetches

### Destructive Operations
- `BITBUCKET_DELETE_REPOSITORY` is irreversible and does not remove forks
- `BITBUCKET_DELETE_ISSUE` is permanent with no recovery option
- Always confirm with the user before executing delete operations

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `BITBUCKET_LIST_WORKSPACES` | `q`, `sort` |
| List repos | `BITBUCKET_LIST_REPOSITORIES_IN_WORKSPACE` | `workspace`, `q`, `role` |
| Create repo | `BITBUCKET_CREATE_REPOSITORY` | `workspace`, `repo_slug`, `is_private` |
| Delete repo | `BITBUCKET_DELETE_REPOSITORY` | `workspace`, `repo_slug` |
| List branches | `BITBUCKET_LIST_BRANCHES` | `workspace`, `repo_slug`, `q` |
| Create branch | `BITBUCKET_CREATE_BRANCH` | `workspace`, `repo_slug`, `name`, `target_hash` |
| List PRs | `BITBUCKET_LIST_PULL_REQUESTS` | `workspace`, `repo_slug`, `state` |
| Create PR | `BITBUCKET_CREATE_PULL_REQUEST` | `workspace`, `repo_slug`, `title`, `source_branch` |
| Get PR details | `BITBUCKET_GET_PULL_REQUEST` | `workspace`, `repo_slug`, `pull_request_id` |
| Get PR diff | `BITBUCKET_GET_PULL_REQUEST_DIFF` | `workspace`, `repo_slug`, `pull_request_id`, `max_chars` |
| Get PR diffstat | `BITBUCKET_GET_PULL_REQUEST_DIFFSTAT` | `workspace`, `repo_slug`, `pull_request_id` |
| Comment on PR | `BITBUCKET_CREATE_PULL_REQUEST_COMMENT` | `workspace`, `repo_slug`, `pull_request_id`, `content_raw` |
| List issues | `BITBUCKET_LIST_ISSUES` | `workspace`, `repo_slug`, `state`, `priority` |
| Create issue | `BITBUCKET_CREATE_ISSUE` | `workspace`, `repo_slug`, `title`, `content` |
| Update issue | `BITBUCKET_UPDATE_ISSUE` | `workspace`, `repo_slug`, `issue_id` |
| Comment on issue | `BITBUCKET_CREATE_ISSUE_COMMENT` | `workspace`, `repo_slug`, `issue_id`, `content` |
| Delete issue | `BITBUCKET_DELETE_ISSUE` | `workspace`, `repo_slug`, `issue_id` |
| List members | `BITBUCKET_LIST_WORKSPACE_MEMBERS` | `workspace` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/clickup-automation/SKILL.md

---
name: clickup-automation
description: "Automate ClickUp project management including tasks, spaces, folders, lists, comments, and team operations via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# ClickUp Automation via Rube MCP

Automate ClickUp project management workflows including task creation and updates, workspace hierarchy navigation, comments, and team member management through Composio's ClickUp toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active ClickUp connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `clickup`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `clickup`
3. If connection is not ACTIVE, follow the returned auth link to complete ClickUp OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Tasks

**When to use**: User wants to create tasks, subtasks, update task properties, or list tasks in a ClickUp list.

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - Get workspace/team IDs [Prerequisite]
2. `CLICKUP_GET_SPACES` - List spaces in the workspace [Prerequisite]
3. `CLICKUP_GET_FOLDERS` - List folders in a space [Prerequisite]
4. `CLICKUP_GET_FOLDERLESS_LISTS` - Get lists not inside folders [Optional]
5. `CLICKUP_GET_LIST` - Validate list and check available statuses [Prerequisite]
6. `CLICKUP_CREATE_TASK` - Create a task in the target list [Required]
7. `CLICKUP_CREATE_TASK` (with `parent`) - Create subtask under a parent task [Optional]
8. `CLICKUP_UPDATE_TASK` - Modify task status, assignees, dates, priority [Optional]
9. `CLICKUP_GET_TASK` - Retrieve full task details [Optional]
10. `CLICKUP_GET_TASKS` - List all tasks in a list with filters [Optional]
11. `CLICKUP_DELETE_TASK` - Permanently remove a task [Optional]

**Key parameters for CLICKUP_CREATE_TASK**:
- `list_id`: Target list ID (integer, required)
- `name`: Task name (string, required)
- `description`: Detailed task description
- `status`: Must exactly match (case-sensitive) a status name configured in the target list
- `priority`: 1 (Urgent), 2 (High), 3 (Normal), 4 (Low)
- `assignees`: Array of user IDs (integers)
- `due_date`: Unix timestamp in milliseconds
- `parent`: Parent task ID string for creating subtasks
- `tags`: Array of tag name strings
- `time_estimate`: Estimated time in milliseconds

**Pitfalls**:
- `status` is case-sensitive and must match an existing status in the list; use `CLICKUP_GET_LIST` to check available statuses
- `due_date` and `start_date` are Unix timestamps in **milliseconds**, not seconds
- Subtask `parent` must be a task (not another subtask) in the same list
- `notify_all` triggers watcher notifications; set to false for bulk operations
- Retries can create duplicates; track created task IDs to avoid re-creation
- `custom_item_id` for milestones (ID 1) is subject to workspace plan quotas

### 2. Navigate Workspace Hierarchy

**When to use**: User wants to browse or manage the ClickUp workspace structure (Workspaces > Spaces > Folders > Lists).

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - List all accessible workspaces [Required]
2. `CLICKUP_GET_SPACES` - List spaces within a workspace [Required]
3. `CLICKUP_GET_SPACE` - Get details for a specific space [Optional]
4. `CLICKUP_GET_FOLDERS` - List folders in a space [Required]
5. `CLICKUP_GET_FOLDER` - Get details for a specific folder [Optional]
6. `CLICKUP_CREATE_FOLDER` - Create a new folder in a space [Optional]
7. `CLICKUP_GET_FOLDERLESS_LISTS` - List lists not inside any folder [Required]
8. `CLICKUP_GET_LIST` - Get list details including statuses and custom fields [Optional]

**Key parameters**:
- `team_id`: Workspace ID from GET_AUTHORIZED_TEAMS_WORKSPACES (required for spaces)
- `space_id`: Space ID (required for folders and folderless lists)
- `folder_id`: Folder ID (required for GET_FOLDER)
- `list_id`: List ID (required for GET_LIST)
- `archived`: Boolean filter for archived/active items

**Pitfalls**:
- ClickUp hierarchy is: Workspace (Team) > Space > Folder > List > Task
- Lists can exist directly under Spaces (folderless) or inside Folders
- Must use `CLICKUP_GET_FOLDERLESS_LISTS` to find lists not inside folders; `CLICKUP_GET_FOLDERS` only returns folders
- `team_id` in ClickUp API refers to the Workspace ID, not a user group

### 3. Add Comments to Tasks

**When to use**: User wants to add comments, review existing comments, or manage comment threads on tasks.

**Tool sequence**:
1. `CLICKUP_GET_TASK` - Verify task exists and get task_id [Prerequisite]
2. `CLICKUP_CREATE_TASK_COMMENT` - Add a new comment to the task [Required]
3. `CLICKUP_GET_TASK_COMMENTS` - List existing comments on the task [Optional]
4. `CLICKUP_UPDATE_COMMENT` - Edit comment text, assignee, or resolution status [Optional]

**Key parameters for CLICKUP_CREATE_TASK_COMMENT**:
- `task_id`: Task ID string (required)
- `comment_text`: Comment content with ClickUp formatting support (required)
- `assignee`: User ID to assign the comment to (required)
- `notify_all`: true/false for watcher notifications (required)

**Key parameters for CLICKUP_GET_TASK_COMMENTS**:
- `task_id`: Task ID string (required)
- `start` / `start_id`: Pagination for older comments (max 25 per page)

**Pitfalls**:
- `CLICKUP_CREATE_TASK_COMMENT` requires all four fields: `task_id`, `comment_text`, `assignee`, and `notify_all`
- `assignee` on a comment assigns the comment (not the task) to that user
- Comments are paginated at 25 per page; use `start` (Unix ms) and `start_id` for older pages
- `CLICKUP_UPDATE_COMMENT` requires all four fields: `comment_id`, `comment_text`, `assignee`, `resolved`

### 4. Manage Team Members and Assignments

**When to use**: User wants to view workspace members, check seat utilization, or look up user details.

**Tool sequence**:
1. `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` - List workspaces and get team_id [Required]
2. `CLICKUP_GET_WORKSPACE_SEATS` - Check seat utilization (members vs guests) [Required]
3. `CLICKUP_GET_TEAMS` - List user groups within the workspace [Optional]
4. `CLICKUP_GET_USER` - Get details for a specific user (Enterprise only) [Optional]
5. `CLICKUP_GET_CUSTOM_ROLES` - List custom permission roles [Optional]

**Key parameters**:
- `team_id`: Workspace ID (required for all team operations)
- `user_id`: Specific user ID for GET_USER
- `group_ids`: Comma-separated group IDs to filter teams

**Pitfalls**:
- `CLICKUP_GET_WORKSPACE_SEATS` returns seat counts, not member details; distinguish members from guests
- `CLICKUP_GET_TEAMS` returns user groups, not workspace members; empty groups does not mean no members
- `CLICKUP_GET_USER` is only available on ClickUp Enterprise Plan
- Must repeat workspace seat queries for each workspace in multi-workspace setups

### 5. Filter and Query Tasks

**When to use**: User wants to find tasks with specific filters (status, assignee, dates, tags, custom fields).

**Tool sequence**:
1. `CLICKUP_GET_TASKS` - Filter tasks in a list with multiple criteria [Required]
2. `CLICKUP_GET_TASK` - Get full details for individual tasks [Optional]

**Key parameters for CLICKUP_GET_TASKS**:
- `list_id`: List ID (integer, required)
- `statuses`: Array of status strings to filter by
- `assignees`: Array of user ID strings
- `tags`: Array of tag name strings
- `due_date_gt` / `due_date_lt`: Unix timestamp in ms for date range
- `include_closed`: Boolean to include closed tasks
- `subtasks`: Boolean to include subtasks
- `order_by`: "id", "created", "updated", or "due_date"
- `page`: Page number starting at 0 (max 100 tasks per page)

**Pitfalls**:
- Only tasks whose home list matches `list_id` are returned; tasks in sublists are not included
- Date filters use Unix timestamps in milliseconds
- Status strings must match exactly; use URL encoding for spaces (e.g., "to%20do")
- Page numbering starts at 0; each page returns up to 100 tasks
- `custom_fields` filter accepts an array of JSON strings, not objects

## Common Patterns

### ID Resolution
Always resolve names to IDs through the hierarchy:
- **Workspace name -> team_id**: `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` and match by name
- **Space name -> space_id**: `CLICKUP_GET_SPACES` with `team_id`
- **Folder name -> folder_id**: `CLICKUP_GET_FOLDERS` with `space_id`
- **List name -> list_id**: Navigate folders or use `CLICKUP_GET_FOLDERLESS_LISTS`
- **Task name -> task_id**: `CLICKUP_GET_TASKS` with `list_id` and match by name

### Pagination
- `CLICKUP_GET_TASKS`: Page-based with `page` starting at 0, max 100 tasks per page
- `CLICKUP_GET_TASK_COMMENTS`: Uses `start` (Unix ms) and `start_id` for cursor-based paging, max 25 per page
- Continue fetching until response returns fewer items than the page size

## Known Pitfalls

### ID Formats
- Workspace/Team IDs are large integers
- Space, folder, and list IDs are integers
- Task IDs are alphanumeric strings (e.g., "9hz", "abc123")
- User IDs are integers
- Comment IDs are integers

### Rate Limits
- ClickUp enforces rate limits; bulk task creation can trigger 429 responses
- Honor `Retry-After` header when present
- Set `notify_all=false` for bulk operations to reduce notification load

### Parameter Quirks
- `team_id` in the API means Workspace ID, not a user group
- `status` on tasks is case-sensitive and list-specific
- Dates are Unix timestamps in **milliseconds** (multiply seconds by 1000)
- `priority` is an integer 1-4 (1=Urgent, 4=Low), not a string
- `CLICKUP_CREATE_TASK_COMMENT` marks `assignee` and `notify_all` as required
- To clear a task description, pass a single space `" "` to `CLICKUP_UPDATE_TASK`

### Hierarchy Rules
- Subtask parent must not itself be a subtask
- Subtask parent must be in the same list
- Lists can be folderless (directly in a Space) or inside a Folder
- Subitem boards are not supported by CLICKUP_CREATE_TASK

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `CLICKUP_GET_AUTHORIZED_TEAMS_WORKSPACES` | (none) |
| List spaces | `CLICKUP_GET_SPACES` | `team_id` |
| Get space details | `CLICKUP_GET_SPACE` | `space_id` |
| List folders | `CLICKUP_GET_FOLDERS` | `space_id` |
| Get folder details | `CLICKUP_GET_FOLDER` | `folder_id` |
| Create folder | `CLICKUP_CREATE_FOLDER` | `space_id`, `name` |
| Folderless lists | `CLICKUP_GET_FOLDERLESS_LISTS` | `space_id` |
| Get list details | `CLICKUP_GET_LIST` | `list_id` |
| Create task | `CLICKUP_CREATE_TASK` | `list_id`, `name`, `status`, `assignees` |
| Update task | `CLICKUP_UPDATE_TASK` | `task_id`, `status`, `priority` |
| Get task | `CLICKUP_GET_TASK` | `task_id`, `include_subtasks` |
| List tasks | `CLICKUP_GET_TASKS` | `list_id`, `statuses`, `page` |
| Delete task | `CLICKUP_DELETE_TASK` | `task_id` |
| Add comment | `CLICKUP_CREATE_TASK_COMMENT` | `task_id`, `comment_text`, `assignee` |
| List comments | `CLICKUP_GET_TASK_COMMENTS` | `task_id`, `start`, `start_id` |
| Update comment | `CLICKUP_UPDATE_COMMENT` | `comment_id`, `comment_text`, `resolved` |
| Workspace seats | `CLICKUP_GET_WORKSPACE_SEATS` | `team_id` |
| List user groups | `CLICKUP_GET_TEAMS` | `team_id` |
| Get user details | `CLICKUP_GET_USER` | `team_id`, `user_id` |
| Custom roles | `CLICKUP_GET_CUSTOM_ROLES` | `team_id` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/github-workflow-automation/SKILL.md

---
name: github-workflow-automation
description: "Automate GitHub workflows with AI assistance. Includes PR reviews, issue triage, CI/CD integration, and Git operations. Use when automating GitHub workflows, setting up PR review automation, creati..."
risk: unknown
source: community
date_added: "2026-02-27"
---

# 🔧 GitHub Workflow Automation

> Patterns for automating GitHub workflows with AI assistance, inspired by modern agentic CLI and DevOps practices.

## When to Use This Skill

Use this skill when:

- Automating PR reviews with AI
- Setting up issue triage automation
- Creating GitHub Actions workflows
- Integrating AI into CI/CD pipelines
- Automating Git operations (rebases, cherry-picks)

---

## 1. Automated PR Review

### 1.1 PR Review Action

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed
        run: |
          files=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          echo "files<<EOF" >> $GITHUB_OUTPUT
          echo "$files" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Get diff
        id: diff
        run: |
          diff=$(git diff origin/${{ github.base_ref }}...HEAD)
          echo "diff<<EOF" >> $GITHUB_OUTPUT
          echo "$diff" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: AI Review
        uses: actions/github-script@v7
        with:
          script: |
            const { Anthropic } = require('@provider-specific-ai-crawler/sdk');
            const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

            const response = await client.messages.create({
              model: "example-model",
              max_tokens: 4096,
              messages: [{
                role: "user",
                content: `Review this PR diff and provide feedback:
                
                Changed files: ${{ steps.changed.outputs.files }}
                
                Diff:
                ${{ steps.diff.outputs.diff }}
                
                Provide:
                1. Summary of changes
                2. Potential issues or bugs
                3. Suggestions for improvement
                4. Security concerns if any
                
                Format as GitHub markdown.`
              }]
            });

            await github.rest.pulls.createReview({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              body: response.content[0].text,
              event: 'COMMENT'
            });
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### 1.2 Review Comment Patterns

````markdown
# AI Review Structure

## 📋 Summary

Brief description of what this PR does.

## ✅ What looks good

- Well-structured code
- Good test coverage
- Clear naming conventions

## ⚠️ Potential Issues

1. **Line 42**: Possible null pointer exception
   ```javascript
   // Current
   user.profile.name;
   // Suggested
   user?.profile?.name ?? "Unknown";
   ```
````

2. **Line 78**: Consider error handling
   ```javascript
   // Add try-catch or .catch()
   ```

## 💡 Suggestions

- Consider extracting the validation logic into a separate function
- Add JSDoc comments for public methods

## 🔒 Security Notes

- No sensitive data exposure detected
- API key handling looks correct

````

### 1.3 Focused Reviews

```yaml
# Review only specific file types
- name: Filter code files
  run: |
    files=$(git diff --name-only origin/${{ github.base_ref }}...HEAD | \
            grep -E '\.(ts|tsx|js|jsx|py|go)$' || true)
    echo "code_files=$files" >> $GITHUB_OUTPUT

# Review with context
- name: AI Review with context
  run: |
    # Include relevant context files
    context=""
    for file in ${{ steps.changed.outputs.files }}; do
      if [[ -f "$file" ]]; then
        context+="=== $file ===\n$(cat $file)\n\n"
      fi
    done

    # Send to AI with full file context
````

---

## 2. Issue Triage Automation

### 2.1 Auto-label Issues

```yaml
# .github/workflows/issue-triage.yml
name: Issue Triage

on:
  issues:
    types: [opened]

jobs:
  triage:
    runs-on: ubuntu-latest
    permissions:
      issues: write

    steps:
      - name: Analyze issue
        uses: actions/github-script@v7
        with:
          script: |
            const issue = context.payload.issue;

            // Call AI to analyze
            const analysis = await analyzeIssue(issue.title, issue.body);

            // Apply labels
            const labels = [];

            if (analysis.type === 'bug') {
              labels.push('bug');
              if (analysis.severity === 'high') labels.push('priority: high');
            } else if (analysis.type === 'feature') {
              labels.push('enhancement');
            } else if (analysis.type === 'question') {
              labels.push('question');
            }

            if (analysis.area) {
              labels.push(`area: ${analysis.area}`);
            }

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: issue.number,
              labels: labels
            });

            // Add initial response
            if (analysis.type === 'bug' && !analysis.hasReproSteps) {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                body: `Thanks for reporting this issue!

To help us investigate, could you please provide:
- Steps to reproduce the issue
- Expected behavior
- Actual behavior
- Environment (OS, version, etc.)

This will help us resolve your issue faster. 🙏`
              });
            }
```

### 2.2 Issue Analysis Prompt

```typescript
const TRIAGE_PROMPT = `
Analyze this GitHub issue and classify it:

Title: {title}
Body: {body}

Return JSON with:
{
  "type": "bug" | "feature" | "question" | "docs" | "other",
  "severity": "low" | "medium" | "high" | "critical",
  "area": "frontend" | "backend" | "api" | "docs" | "ci" | "other",
  "summary": "one-line summary",
  "hasReproSteps": boolean,
  "isFirstContribution": boolean,
  "suggestedLabels": ["label1", "label2"],
  "suggestedAssignees": ["username"] // based on area expertise
}
`;
```

### 2.3 Stale Issue Management

```yaml
# .github/workflows/stale.yml
name: Manage Stale Issues

on:
  schedule:
    - cron: "0 0 * * *" # Daily

jobs:
  stale:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/stale@v9
        with:
          stale-issue-message: |
            This issue has been automatically marked as stale because it has not had 
            recent activity. It will be closed in 14 days if no further activity occurs.

            If this issue is still relevant:
            - Add a comment with an update
            - Remove the `stale` label

            Thank you for your contributions! 🙏

          stale-pr-message: |
            This PR has been automatically marked as stale. Please update it or it 
            will be closed in 14 days.

          days-before-stale: 60
          days-before-close: 14
          stale-issue-label: "stale"
          stale-pr-label: "stale"
          exempt-issue-labels: "pinned,security,in-progress"
          exempt-pr-labels: "pinned,security"
```

---

## 3. CI/CD Integration

### 3.1 Smart Test Selection

```yaml
# .github/workflows/smart-tests.yml
name: Smart Test Selection

on:
  pull_request:

jobs:
  analyze:
    runs-on: ubuntu-latest
    outputs:
      test_suites: ${{ steps.analyze.outputs.suites }}

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Analyze changes
        id: analyze
        run: |
          # Get changed files
          changed=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)

          # Determine which test suites to run
          suites="[]"

          if echo "$changed" | grep -q "^src/api/"; then
            suites=$(echo $suites | jq '. + ["api"]')
          fi

          if echo "$changed" | grep -q "^src/frontend/"; then
            suites=$(echo $suites | jq '. + ["frontend"]')
          fi

          if echo "$changed" | grep -q "^src/database/"; then
            suites=$(echo $suites | jq '. + ["database", "api"]')
          fi

          # If nothing specific, run all
          if [ "$suites" = "[]" ]; then
            suites='["all"]'
          fi

          echo "suites=$suites" >> $GITHUB_OUTPUT

  test:
    needs: analyze
    runs-on: ubuntu-latest
    strategy:
      matrix:
        suite: ${{ fromJson(needs.analyze.outputs.test_suites) }}

    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: |
          if [ "${{ matrix.suite }}" = "all" ]; then
            npm test
          else
            npm test -- --suite ${{ matrix.suite }}
          fi
```

### 3.2 Deployment with AI Validation

```yaml
# .github/workflows/deploy.yml
name: Deploy with AI Validation

on:
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Get deployment changes
        id: changes
        run: |
          # Get commits since last deployment
          last_deploy=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          if [ -n "$last_deploy" ]; then
            changes=$(git log --oneline $last_deploy..HEAD)
          else
            changes=$(git log --oneline -10)
          fi
          echo "changes<<EOF" >> $GITHUB_OUTPUT
          echo "$changes" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: AI Risk Assessment
        id: assess
        uses: actions/github-script@v7
        with:
          script: |
            // Analyze changes for deployment risk
            const prompt = `
            Analyze these changes for deployment risk:

            ${process.env.CHANGES}

            Return JSON:
            {
              "riskLevel": "low" | "medium" | "high",
              "concerns": ["concern1", "concern2"],
              "recommendations": ["rec1", "rec2"],
              "requiresManualApproval": boolean
            }
            `;

            // Call AI and parse response
            const analysis = await callAI(prompt);

            if (analysis.riskLevel === 'high') {
              core.setFailed('High-risk deployment detected. Manual review required.');
            }

            return analysis;
        env:
          CHANGES: ${{ steps.changes.outputs.changes }}

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy
        run: |
          echo "Deploying to production..."
          # Deployment commands here
```

### 3.3 Rollback Automation

```yaml
# .github/workflows/rollback.yml
name: Automated Rollback

on:
  workflow_dispatch:
    inputs:
      reason:
        description: "Reason for rollback"
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Find last stable version
        id: stable
        run: |
          # Find last successful deployment
          stable=$(git tag -l 'v*' --sort=-version:refname | head -1)
          echo "version=$stable" >> $GITHUB_OUTPUT

      - name: Rollback
        run: |
          git checkout ${{ steps.stable.outputs.version }}
          # Deploy stable version
          npm run deploy

      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🔄 Production rolled back to ${{ steps.stable.outputs.version }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Rollback executed*\n• Version: `${{ steps.stable.outputs.version }}`\n• Reason: ${{ inputs.reason }}\n• Triggered by: ${{ github.actor }}"
                  }
                }
              ]
            }
```

---

## 4. Git Operations

### 4.1 Automated Rebasing

```yaml
# .github/workflows/auto-rebase.yml
name: Auto Rebase

on:
  issue_comment:
    types: [created]

jobs:
  rebase:
    if: github.event.issue.pull_request && contains(github.event.comment.body, '/rebase')
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Setup Git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Rebase PR
        run: |
          # Fetch PR branch
          gh pr checkout ${{ github.event.issue.number }}

          # Rebase onto main
          git fetch origin main
          git rebase origin/main

          # Force push
          git push --force-with-lease
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Comment result
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '✅ Successfully rebased onto main!'
            })
```

### 4.2 Smart Cherry-Pick

```typescript
// AI-assisted cherry-pick that handles conflicts
async function smartCherryPick(commitHash: string, targetBranch: string) {
  // Get commit info
  const commitInfo = await exec(`git show ${commitHash} --stat`);

  // Check for potential conflicts
  const targetDiff = await exec(
    `git diff ${targetBranch}...HEAD -- ${affectedFiles}`
  );

  // AI analysis
  const analysis = await ai.analyze(`
    I need to cherry-pick this commit to ${targetBranch}:
    
    ${commitInfo}
    
    Current state of affected files on ${targetBranch}:
    ${targetDiff}
    
    Will there be conflicts? If so, suggest resolution strategy.
  `);

  if (analysis.willConflict) {
    // Create branch for manual resolution
    await exec(
      `git checkout -b cherry-pick-${commitHash.slice(0, 7)} ${targetBranch}`
    );
    const result = await exec(`git cherry-pick ${commitHash}`, {
      allowFail: true,
    });

    if (result.failed) {
      // AI-assisted conflict resolution
      const conflicts = await getConflicts();
      for (const conflict of conflicts) {
        const resolution = await ai.resolveConflict(conflict);
        await applyResolution(conflict.file, resolution);
      }
    }
  } else {
    await exec(`git checkout ${targetBranch}`);
    await exec(`git cherry-pick ${commitHash}`);
  }
}
```

### 4.3 Branch Cleanup

```yaml
# .github/workflows/branch-cleanup.yml
name: Branch Cleanup

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Find stale branches
        id: stale
        run: |
          # Branches not updated in 30 days
          stale=$(git for-each-ref --sort=-committerdate refs/remotes/origin \
            --format='%(refname:short) %(committerdate:relative)' | \
            grep -E '[3-9][0-9]+ days|[0-9]+ months|[0-9]+ years' | \
            grep -v 'origin/main\|origin/develop' | \
            cut -d' ' -f1 | sed 's|origin/||')

          echo "branches<<EOF" >> $GITHUB_OUTPUT
          echo "$stale" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Create cleanup PR
        if: steps.stale.outputs.branches != ''
        uses: actions/github-script@v7
        with:
          script: |
            const branches = `${{ steps.stale.outputs.branches }}`.split('\n').filter(Boolean);

            const body = `## 🧹 Stale Branch Cleanup

The following branches haven't been updated in over 30 days:

${branches.map(b => `- \`${b}\``).join('\n')}

### Actions:
- [ ] Review each branch
- [ ] Delete branches that are no longer needed
- Comment \`/keep branch-name\` to preserve specific branches
`;

            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: 'Stale Branch Cleanup',
              body: body,
              labels: ['housekeeping']
            });
```

---

## 5. On-Demand Assistance

### 5.1 @mention Bot

```yaml
# .github/workflows/mention-bot.yml
name: AI Mention Bot

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  respond:
    if: contains(github.event.comment.body, '@ai-helper')
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Extract question
        id: question
        run: |
          # Extract text after @ai-helper
          question=$(echo "${{ github.event.comment.body }}" | sed 's/.*@ai-helper//')
          echo "question=$question" >> $GITHUB_OUTPUT

      - name: Get context
        id: context
        run: |
          if [ "${{ github.event.issue.pull_request }}" != "" ]; then
            # It's a PR - get diff
            gh pr diff ${{ github.event.issue.number }} > context.txt
          else
            # It's an issue - get description
            gh issue view ${{ github.event.issue.number }} --json body -q .body > context.txt
          fi
          echo "context=$(cat context.txt)" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: AI Response
        uses: actions/github-script@v7
        with:
          script: |
            const response = await ai.chat(`
              Context: ${process.env.CONTEXT}
              
              Question: ${process.env.QUESTION}
              
              Provide a helpful, specific answer. Include code examples if relevant.
            `);

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: response
            });
        env:
          CONTEXT: ${{ steps.context.outputs.context }}
          QUESTION: ${{ steps.question.outputs.question }}
```

### 5.2 Command Patterns

```markdown
## Available Commands

| Command              | Description                 |
| :------------------- | :-------------------------- |
| `@ai-helper explain` | Explain the code in this PR |
| `@ai-helper review`  | Request AI code review      |
| `@ai-helper fix`     | Suggest fixes for issues    |
| `@ai-helper test`    | Generate test cases         |
| `@ai-helper docs`    | Generate documentation      |
| `/rebase`            | Rebase PR onto main         |
| `/update`            | Update PR branch from main  |
| `/approve`           | Mark as approved by bot     |
| `/label bug`         | Add 'bug' label             |
| `/assign @user`      | Assign to user              |
```

---

## 6. Repository Configuration

### 6.1 CODEOWNERS

```
# .github/CODEOWNERS

# Global owners
* @org/core-team

# Frontend
/src/frontend/ @org/frontend-team
*.tsx @org/frontend-team
*.css @org/frontend-team

# Backend
/src/api/ @org/backend-team
/src/database/ @org/backend-team

# Infrastructure
/.github/ @org/devops-team
/terraform/ @org/devops-team
Dockerfile @org/devops-team

# Docs
/docs/ @org/docs-team
*.md @org/docs-team

# Security-sensitive
/src/auth/ @org/security-team
/src/crypto/ @org/security-team
```

### 6.2 Branch Protection

```yaml
# Set up via GitHub API
- name: Configure branch protection
  uses: actions/github-script@v7
  with:
    script: |
      await github.rest.repos.updateBranchProtection({
        owner: context.repo.owner,
        repo: context.repo.repo,
        branch: 'main',
        required_status_checks: {
          strict: true,
          contexts: ['test', 'lint', 'ai-review']
        },
        enforce_admins: true,
        required_pull_request_reviews: {
          required_approving_review_count: 1,
          require_code_owner_reviews: true,
          dismiss_stale_reviews: true
        },
        restrictions: null,
        required_linear_history: true,
        allow_force_pushes: false,
        allow_deletions: false
      });
```

---

## Best Practices

### Security

- [ ] Store API keys in GitHub Secrets
- [ ] Use minimal permissions in workflows
- [ ] Validate all inputs
- [ ] Don't expose sensitive data in logs

### Performance

- [ ] Cache dependencies
- [ ] Use matrix builds for parallel testing
- [ ] Skip unnecessary jobs with path filters
- [ ] Use self-hosted runners for heavy workloads

### Reliability

- [ ] Add timeouts to jobs
- [ ] Handle rate limits gracefully
- [ ] Implement retry logic
- [ ] Have rollback procedures

---

## Resources

- Agent CLI GitHub Action pattern for repository automation
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub REST API](https://docs.github.com/en/rest)
- [CODEOWNERS Syntax](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)

## Source: references/skills/github-automation/references/legacy/gitlab-automation/SKILL.md

---
name: gitlab-automation
description: "Automate GitLab project management, issues, merge requests, pipelines, branches, and user operations via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# GitLab Automation via Rube MCP

Automate GitLab operations including project management, issue tracking, merge request workflows, CI/CD pipeline monitoring, branch management, and user administration through Composio's GitLab toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active GitLab connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `gitlab`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `gitlab`
3. If connection is not ACTIVE, follow the returned auth link to complete GitLab OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Issues

**When to use**: User wants to create, update, list, or search issues in a GitLab project

**Tool sequence**:
1. `GITLAB_GET_PROJECTS` - Find the target project and get its ID [Prerequisite]
2. `GITLAB_LIST_PROJECT_ISSUES` - List and filter issues for a project [Required]
3. `GITLAB_CREATE_PROJECT_ISSUE` - Create a new issue [Required for create]
4. `GITLAB_UPDATE_PROJECT_ISSUE` - Update an existing issue (title, labels, state, assignees) [Required for update]
5. `GITLAB_LIST_PROJECT_USERS` - Find user IDs for assignment [Optional]

**Key parameters**:
- `id`: Project ID (integer) or URL-encoded path (e.g., `"my-group/my-project"`)
- `title`: Issue title (required for creation)
- `description`: Issue body text (max 1,048,576 characters)
- `labels`: Comma-separated label names (e.g., `"bug,critical"`)
- `add_labels` / `remove_labels`: Add or remove labels without replacing all
- `state`: Filter by `"all"`, `"opened"`, or `"closed"`
- `state_event`: `"close"` or `"reopen"` to change issue state
- `assignee_ids`: Array of user IDs; use `[0]` to unassign all
- `issue_iid`: Internal issue ID within the project (required for updates)
- `milestone`: Filter by milestone title
- `search`: Search in title and description
- `scope`: `"created_by_me"`, `"assigned_to_me"`, or `"all"`
- `page` / `per_page`: Pagination (default per_page: 20)

**Pitfalls**:
- `id` accepts either integer project ID or URL-encoded path; wrong IDs yield 4xx errors
- `issue_iid` is the project-internal ID (shown as #42), different from the global issue ID
- Labels in `labels` field replace ALL existing labels; use `add_labels`/`remove_labels` for incremental changes
- Setting `assignee_ids` to empty array does NOT unassign; use `[0]` instead
- `updated_at` field requires administrator or project/group owner rights

### 2. Manage Merge Requests

**When to use**: User wants to list, filter, or review merge requests in a project

**Tool sequence**:
1. `GITLAB_GET_PROJECT` - Get project details and verify access [Prerequisite]
2. `GITLAB_GET_PROJECT_MERGE_REQUESTS` - List and filter merge requests [Required]
3. `GITLAB_GET_REPOSITORY_BRANCHES` - Verify source/target branches [Optional]
4. `GITLAB_LIST_ALL_PROJECT_MEMBERS` - Find reviewers/assignees [Optional]

**Key parameters**:
- `id`: Project ID or URL-encoded path
- `state`: `"opened"`, `"closed"`, `"locked"`, `"merged"`, or `"all"`
- `scope`: `"created_by_me"` (default), `"assigned_to_me"`, or `"all"`
- `source_branch` / `target_branch`: Filter by branch names
- `author_id` / `author_username`: Filter by MR author
- `assignee_id`: Filter by assignee (use `None` for unassigned, `Any` for assigned)
- `reviewer_id` / `reviewer_username`: Filter by reviewer
- `labels`: Comma-separated label filter
- `search`: Search in title and description
- `wip`: `"yes"` for draft MRs, `"no"` for non-draft
- `order_by`: `"created_at"` (default), `"title"`, `"merged_at"`, `"updated_at"`
- `view`: `"simple"` for minimal fields
- `iids[]`: Filter by specific MR internal IDs

**Pitfalls**:
- Default `scope` is `"created_by_me"` which limits results; use `"all"` for complete listings
- `author_id` and `author_username` are mutually exclusive
- `reviewer_id` and `reviewer_username` are mutually exclusive
- `approved` filter requires the `mr_approved_filter` feature flag (disabled by default)
- Large MR histories can be noisy; use filters and moderate `per_page` values

### 3. Manage Projects and Repositories

**When to use**: User wants to list projects, create new projects, or manage branches

**Tool sequence**:
1. `GITLAB_GET_PROJECTS` - List all accessible projects with filters [Required]
2. `GITLAB_GET_PROJECT` - Get detailed info for a specific project [Optional]
3. `GITLAB_LIST_USER_PROJECTS` - List projects owned by a specific user [Optional]
4. `GITLAB_CREATE_PROJECT` - Create a new project [Required for create]
5. `GITLAB_GET_REPOSITORY_BRANCHES` - List branches in a project [Required for branch ops]
6. `GITLAB_CREATE_REPOSITORY_BRANCH` - Create a new branch [Optional]
7. `GITLAB_GET_REPOSITORY_BRANCH` - Get details of a specific branch [Optional]
8. `GITLAB_LIST_REPOSITORY_COMMITS` - View commit history [Optional]
9. `GITLAB_GET_PROJECT_LANGUAGES` - Get language breakdown [Optional]

**Key parameters**:
- `name` / `path`: Project name and URL-friendly path (both required for creation)
- `visibility`: `"private"`, `"internal"`, or `"public"`
- `namespace_id`: Group or user ID for project placement
- `search`: Case-insensitive substring search for projects
- `membership`: `true` to limit to projects user is a member of
- `owned`: `true` to limit to user-owned projects
- `project_id`: Project ID for branch operations
- `branch_name`: Name for new branch
- `ref`: Source branch or commit SHA for new branch creation
- `order_by`: `"id"`, `"name"`, `"path"`, `"created_at"`, `"updated_at"`, `"star_count"`, `"last_activity_at"`

**Pitfalls**:
- `GITLAB_GET_PROJECTS` pagination is required for complete coverage; stopping at first page misses projects
- Some responses place items under `data.details`; parse the actual returned list structure
- Most follow-up calls depend on correct `project_id`; verify with `GITLAB_GET_PROJECT` first
- Invalid `branch_name`/`ref`/`sha` causes client errors; verify branch existence via `GITLAB_GET_REPOSITORY_BRANCHES` first
- Both `name` and `path` are required for `GITLAB_CREATE_PROJECT`

### 4. Monitor CI/CD Pipelines

**When to use**: User wants to check pipeline status, list jobs, or monitor CI/CD runs

**Tool sequence**:
1. `GITLAB_GET_PROJECT` - Verify project access [Prerequisite]
2. `GITLAB_LIST_PROJECT_PIPELINES` - List pipelines with filters [Required]
3. `GITLAB_GET_SINGLE_PIPELINE` - Get detailed info for a specific pipeline [Optional]
4. `GITLAB_LIST_PIPELINE_JOBS` - List jobs within a pipeline [Optional]

**Key parameters**:
- `id`: Project ID or URL-encoded path
- `status`: Filter by `"created"`, `"waiting_for_resource"`, `"preparing"`, `"pending"`, `"running"`, `"success"`, `"failed"`, `"canceled"`, `"skipped"`, `"manual"`, `"scheduled"`
- `scope`: `"running"`, `"pending"`, `"finished"`, `"branches"`, `"tags"`
- `ref`: Branch or tag name
- `sha`: Specific commit SHA
- `source`: Pipeline source (use `"parent_pipeline"` for child pipelines)
- `order_by`: `"id"` (default), `"status"`, `"ref"`, `"updated_at"`, `"user_id"`
- `created_after` / `created_before`: ISO 8601 date filters
- `pipeline_id`: Specific pipeline ID for job listing
- `include_retried`: `true` to include retried jobs (default `false`)

**Pitfalls**:
- Large pipeline histories can be noisy; use `status`, `ref`, and date filters to narrow results
- Use moderate `per_page` values to keep output manageable
- Pipeline job `scope` accepts single status string or array of statuses
- `yaml_errors: true` returns only pipelines with invalid configurations

### 5. Manage Users and Members

**When to use**: User wants to find users, list project members, or check user status

**Tool sequence**:
1. `GITLAB_GET_USERS` - Search and list GitLab users [Required]
2. `GITLAB_GET_USER` - Get details for a specific user by ID [Optional]
3. `GITLAB_GET_USERS_ID_STATUS` - Get user status message and availability [Optional]
4. `GITLAB_LIST_ALL_PROJECT_MEMBERS` - List all project members (direct + inherited) [Required for member listing]
5. `GITLAB_LIST_PROJECT_USERS` - List project users with search filter [Optional]

**Key parameters**:
- `search`: Search by name, username, or public email
- `username`: Get specific user by username
- `active` / `blocked`: Filter by user state
- `id`: Project ID for member listing
- `query`: Filter members by name, email, or username
- `state`: Filter members by `"awaiting"` or `"active"` (Premium/Ultimate)
- `user_ids`: Filter by specific user IDs

**Pitfalls**:
- Many user filters (admins, auditors, extern_uid, two_factor) are admin-only
- `GITLAB_LIST_ALL_PROJECT_MEMBERS` includes direct, inherited, and invited members
- User search is case-insensitive but may not match partial email domains
- Premium/Ultimate features (state filter, seat info) are not available on free plans

## Common Patterns

### ID Resolution
GitLab uses two identifier formats for projects:
- **Numeric ID**: Integer project ID (e.g., `123`)
- **URL-encoded path**: Namespace/project format (e.g., `"my-group%2Fmy-project"` or `"my-group/my-project"`)
- **Issue IID vs ID**: `issue_iid` is the project-internal number (#42); the global `id` is different
- **User ID**: Numeric; resolve via `GITLAB_GET_USERS` with `search` or `username`

### Pagination
GitLab uses offset-based pagination:
- Set `page` (starting at 1) and `per_page` (1-100, default 20)
- Continue incrementing `page` until response returns fewer items than `per_page` or is empty
- Total count may be available in response headers (`X-Total`, `X-Total-Pages`)
- Always paginate to completion for accurate results

### URL-Encoded Paths
When using project paths as identifiers:
- Forward slashes must be URL-encoded: `my-group/my-project` becomes `my-group%2Fmy-project`
- Some tools accept unencoded paths; check schema for each tool
- Prefer numeric IDs when available for reliability

## Known Pitfalls

### ID Formats
- Project `id` field accepts both integer and string (URL-encoded path)
- Issue `issue_iid` is project-scoped; do not confuse with global issue ID
- Pipeline IDs are project-scoped integers
- User IDs are global integers across the GitLab instance

### Rate Limits
- GitLab has per-user rate limits (typically 300-2000 requests/minute depending on plan)
- Large pipeline/issue histories should use date and status filters to reduce result sets
- Paginate responsibly with moderate `per_page` values

### Parameter Quirks
- `labels` field replaces ALL labels; use `add_labels`/`remove_labels` for incremental changes
- `assignee_ids: [0]` unassigns all; empty array does nothing
- `scope` defaults vary: `"created_by_me"` for MRs, `"all"` for issues
- `author_id` and `author_username` are mutually exclusive in MR filters
- Date parameters use ISO 8601 format: `"2024-01-15T10:30:00Z"`

### Plan Restrictions
- Some features require Premium/Ultimate: `epic_id`, `weight`, `iteration_id`, `approved_by_ids`, member `state` filter
- Admin-only features: user management filters, `updated_at` override, custom attributes
- The `mr_approved_filter` feature flag is disabled by default

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List projects | `GITLAB_GET_PROJECTS` | `search`, `membership`, `visibility` |
| Get project details | `GITLAB_GET_PROJECT` | `id` |
| User's projects | `GITLAB_LIST_USER_PROJECTS` | `id`, `search`, `owned` |
| Create project | `GITLAB_CREATE_PROJECT` | `name`, `path`, `visibility` |
| List issues | `GITLAB_LIST_PROJECT_ISSUES` | `id`, `state`, `labels`, `search` |
| Create issue | `GITLAB_CREATE_PROJECT_ISSUE` | `id`, `title`, `description`, `labels` |
| Update issue | `GITLAB_UPDATE_PROJECT_ISSUE` | `id`, `issue_iid`, `state_event` |
| List merge requests | `GITLAB_GET_PROJECT_MERGE_REQUESTS` | `id`, `state`, `scope`, `labels` |
| List branches | `GITLAB_GET_REPOSITORY_BRANCHES` | `project_id`, `search` |
| Get branch | `GITLAB_GET_REPOSITORY_BRANCH` | `project_id`, `branch_name` |
| Create branch | `GITLAB_CREATE_REPOSITORY_BRANCH` | `project_id`, `branch_name`, `ref` |
| List commits | `GITLAB_LIST_REPOSITORY_COMMITS` | project ID, branch ref |
| Project languages | `GITLAB_GET_PROJECT_LANGUAGES` | project ID |
| List pipelines | `GITLAB_LIST_PROJECT_PIPELINES` | `id`, `status`, `ref` |
| Get pipeline | `GITLAB_GET_SINGLE_PIPELINE` | `project_id`, `pipeline_id` |
| List pipeline jobs | `GITLAB_LIST_PIPELINE_JOBS` | `id`, `pipeline_id`, `scope` |
| Search users | `GITLAB_GET_USERS` | `search`, `username`, `active` |
| Get user | `GITLAB_GET_USER` | user ID |
| User status | `GITLAB_GET_USERS_ID_STATUS` | user ID |
| List project members | `GITLAB_LIST_ALL_PROJECT_MEMBERS` | `id`, `query`, `state` |
| List project users | `GITLAB_LIST_PROJECT_USERS` | `id`, `search` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/jira-automation/SKILL.md

---
name: jira-automation
description: "Automate Jira tasks via Rube MCP (Composio): issues, projects, sprints, boards, comments, users. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Jira Automation via Rube MCP

Automate Jira operations through Composio's Jira toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Jira connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `jira`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `jira`
3. If connection is not ACTIVE, follow the returned auth link to complete Jira OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Search and Filter Issues

**When to use**: User wants to find issues using JQL or browse project issues

**Tool sequence**:
1. `JIRA_SEARCH_FOR_ISSUES_USING_JQL_POST` - Search with JQL query [Required]
2. `JIRA_GET_ISSUE` - Get full details of a specific issue [Optional]

**Key parameters**:
- `jql`: JQL query string (e.g., `project = PROJ AND status = "In Progress"`)
- `maxResults`: Max results per page (default 50, max 100)
- `startAt`: Pagination offset
- `fields`: Array of field names to return
- `issueIdOrKey`: Issue key like 'PROJ-123' for GET_ISSUE

**Pitfalls**:
- JQL field names are case-sensitive and must match Jira configuration
- Custom fields use IDs like `customfield_10001`, not display names
- Results are paginated; check `total` vs `startAt + maxResults` to continue

### 2. Create and Edit Issues

**When to use**: User wants to create new issues or update existing ones

**Tool sequence**:
1. `JIRA_GET_ALL_PROJECTS` - List projects to find project key [Prerequisite]
2. `JIRA_GET_FIELDS` - Get available fields and their IDs [Prerequisite]
3. `JIRA_CREATE_ISSUE` - Create a new issue [Required]
4. `JIRA_EDIT_ISSUE` - Update fields on an existing issue [Optional]
5. `JIRA_ASSIGN_ISSUE` - Assign issue to a user [Optional]

**Key parameters**:
- `project`: Project key (e.g., 'PROJ')
- `issuetype`: Issue type name (e.g., 'Bug', 'Story', 'Task')
- `summary`: Issue title
- `description`: Issue description (Atlassian Document Format or plain text)
- `issueIdOrKey`: Issue key for edits

**Pitfalls**:
- Issue types and required fields vary by project; use GET_FIELDS to check
- Custom fields require exact field IDs, not display names
- Description may need Atlassian Document Format (ADF) for rich content

### 3. Manage Sprints and Boards

**When to use**: User wants to work with agile boards, sprints, and backlogs

**Tool sequence**:
1. `JIRA_LIST_BOARDS` - List all boards [Prerequisite]
2. `JIRA_LIST_SPRINTS` - List sprints for a board [Required]
3. `JIRA_MOVE_ISSUE_TO_SPRINT` - Move issue to a sprint [Optional]
4. `JIRA_CREATE_SPRINT` - Create a new sprint [Optional]

**Key parameters**:
- `boardId`: Board ID from LIST_BOARDS
- `sprintId`: Sprint ID for move operations
- `name`: Sprint name for creation
- `startDate`/`endDate`: Sprint dates in ISO format

**Pitfalls**:
- Boards and sprints are specific to Jira Software (not Jira Core)
- Only one sprint can be active at a time per board

### 4. Manage Comments

**When to use**: User wants to add or view comments on issues

**Tool sequence**:
1. `JIRA_LIST_ISSUE_COMMENTS` - List existing comments [Optional]
2. `JIRA_ADD_COMMENT` - Add a comment to an issue [Required]

**Key parameters**:
- `issueIdOrKey`: Issue key like 'PROJ-123'
- `body`: Comment body (supports ADF for rich text)

**Pitfalls**:
- Comments support ADF (Atlassian Document Format) for formatting
- Mentions use account IDs, not usernames

### 5. Manage Projects and Users

**When to use**: User wants to list projects, find users, or manage project roles

**Tool sequence**:
1. `JIRA_GET_ALL_PROJECTS` - List all projects [Optional]
2. `JIRA_GET_PROJECT` - Get project details [Optional]
3. `JIRA_FIND_USERS` / `JIRA_GET_ALL_USERS` - Search for users [Optional]
4. `JIRA_GET_PROJECT_ROLES` - List project roles [Optional]
5. `JIRA_ADD_USERS_TO_PROJECT_ROLE` - Add user to role [Optional]

**Key parameters**:
- `projectIdOrKey`: Project key
- `query`: Search text for FIND_USERS
- `roleId`: Role ID for role operations

**Pitfalls**:
- User operations use account IDs (not email or display name)
- Project roles differ from global permissions

## Common Patterns

### JQL Syntax

**Common operators**:
- `project = "PROJ"` - Filter by project
- `status = "In Progress"` - Filter by status
- `assignee = currentUser()` - Current user's issues
- `created >= -7d` - Created in last 7 days
- `labels = "bug"` - Filter by label
- `priority = High` - Filter by priority
- `ORDER BY created DESC` - Sort results

**Combinators**:
- `AND` - Both conditions
- `OR` - Either condition
- `NOT` - Negate condition

### Pagination

- Use `startAt` and `maxResults` parameters
- Check `total` in response to determine remaining pages
- Continue until `startAt + maxResults >= total`

## Known Pitfalls

**Field Names**:
- Custom fields use IDs like `customfield_10001`
- Use JIRA_GET_FIELDS to discover field IDs and names
- Field names in JQL may differ from API field names

**Authentication**:
- Jira Cloud uses account IDs, not usernames
- Site URL must be configured correctly in the connection

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| Search issues (JQL) | JIRA_SEARCH_FOR_ISSUES_USING_JQL_POST | jql, maxResults |
| Get issue | JIRA_GET_ISSUE | issueIdOrKey |
| Create issue | JIRA_CREATE_ISSUE | project, issuetype, summary |
| Edit issue | JIRA_EDIT_ISSUE | issueIdOrKey, fields |
| Assign issue | JIRA_ASSIGN_ISSUE | issueIdOrKey, accountId |
| Add comment | JIRA_ADD_COMMENT | issueIdOrKey, body |
| List comments | JIRA_LIST_ISSUE_COMMENTS | issueIdOrKey |
| List projects | JIRA_GET_ALL_PROJECTS | (none) |
| Get project | JIRA_GET_PROJECT | projectIdOrKey |
| List boards | JIRA_LIST_BOARDS | (none) |
| List sprints | JIRA_LIST_SPRINTS | boardId |
| Move to sprint | JIRA_MOVE_ISSUE_TO_SPRINT | sprintId, issues |
| Create sprint | JIRA_CREATE_SPRINT | name, boardId |
| Find users | JIRA_FIND_USERS | query |
| Get fields | JIRA_GET_FIELDS | (none) |
| List filters | JIRA_LIST_FILTERS | (none) |
| Project roles | JIRA_GET_PROJECT_ROLES | projectIdOrKey |
| Project versions | JIRA_GET_PROJECT_VERSIONS | projectIdOrKey |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/linear-automation/SKILL.md

---
name: linear-automation
description: "Automate Linear tasks via Rube MCP (Composio): issues, projects, cycles, teams, labels. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Linear Automation via Rube MCP

Automate Linear operations through Composio's Linear toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Linear connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `linear`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `linear`
3. If connection is not ACTIVE, follow the returned auth link to complete Linear OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Manage Issues

**When to use**: User wants to create, search, update, or list Linear issues

**Tool sequence**:
1. `LINEAR_GET_ALL_LINEAR_TEAMS` - Get team IDs [Prerequisite]
2. `LINEAR_LIST_LINEAR_STATES` - Get workflow states for a team [Prerequisite]
3. `LINEAR_CREATE_LINEAR_ISSUE` - Create a new issue [Optional]
4. `LINEAR_SEARCH_ISSUES` / `LINEAR_LIST_LINEAR_ISSUES` - Find issues [Optional]
5. `LINEAR_GET_LINEAR_ISSUE` - Get issue details [Optional]
6. `LINEAR_UPDATE_ISSUE` - Update issue properties [Optional]

**Key parameters**:
- `team_id`: Team ID (required for creation)
- `title`: Issue title
- `description`: Issue description (Markdown supported)
- `state_id`: Workflow state ID
- `assignee_id`: Assignee user ID
- `priority`: 0 (none), 1 (urgent), 2 (high), 3 (medium), 4 (low)
- `label_ids`: Array of label IDs

**Pitfalls**:
- Team ID is required when creating issues; use GET_ALL_LINEAR_TEAMS first
- State IDs are team-specific; use LIST_LINEAR_STATES with the correct team
- Priority uses integer values 0-4, not string names

### 2. Manage Projects

**When to use**: User wants to create or update Linear projects

**Tool sequence**:
1. `LINEAR_LIST_LINEAR_PROJECTS` - List existing projects [Optional]
2. `LINEAR_CREATE_LINEAR_PROJECT` - Create a new project [Optional]
3. `LINEAR_UPDATE_LINEAR_PROJECT` - Update project details [Optional]

**Key parameters**:
- `name`: Project name
- `description`: Project description
- `team_ids`: Array of team IDs associated with the project
- `state`: Project state (e.g., 'planned', 'started', 'completed')

**Pitfalls**:
- Projects span teams; they can be associated with multiple teams

### 3. Manage Cycles

**When to use**: User wants to work with Linear cycles (sprints)

**Tool sequence**:
1. `LINEAR_GET_ALL_LINEAR_TEAMS` - Get team ID [Prerequisite]
2. `LINEAR_GET_CYCLES_BY_TEAM_ID` / `LINEAR_LIST_LINEAR_CYCLES` - List cycles [Required]

**Key parameters**:
- `team_id`: Team ID for cycle operations
- `number`: Cycle number

**Pitfalls**:
- Cycles are team-specific; always scope by team_id

### 4. Manage Labels and Comments

**When to use**: User wants to create labels or comment on issues

**Tool sequence**:
1. `LINEAR_CREATE_LINEAR_LABEL` - Create a new label [Optional]
2. `LINEAR_CREATE_LINEAR_COMMENT` - Comment on an issue [Optional]
3. `LINEAR_UPDATE_LINEAR_COMMENT` - Edit a comment [Optional]

**Key parameters**:
- `name`: Label name
- `color`: Label color (hex)
- `issue_id`: Issue ID for comments
- `body`: Comment body (Markdown)

**Pitfalls**:
- Labels can be team-scoped or workspace-scoped
- Comment body supports Markdown formatting

### 5. Custom GraphQL Queries

**When to use**: User needs advanced queries not covered by standard tools

**Tool sequence**:
1. `LINEAR_RUN_QUERY_OR_MUTATION` - Execute custom GraphQL [Required]

**Key parameters**:
- `query`: GraphQL query or mutation string
- `variables`: Variables for the query

**Pitfalls**:
- Requires knowledge of Linear's GraphQL schema
- Rate limits apply to GraphQL queries

## Common Patterns

### ID Resolution

**Team name -> Team ID**:
```
1. Call LINEAR_GET_ALL_LINEAR_TEAMS
2. Find team by name in response
3. Extract id field
```

**State name -> State ID**:
```
1. Call LINEAR_LIST_LINEAR_STATES with team_id
2. Find state by name
3. Extract id field
```

### Pagination

- Linear tools return paginated results
- Check for pagination cursors in responses
- Pass cursor to next request for additional pages

## Known Pitfalls

**Team Scoping**:
- Issues, states, and cycles are team-specific
- Always resolve team_id before creating issues

**Priority Values**:
- 0 = No priority, 1 = Urgent, 2 = High, 3 = Medium, 4 = Low
- Use integer values, not string names

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List teams | LINEAR_GET_ALL_LINEAR_TEAMS | (none) |
| Create issue | LINEAR_CREATE_LINEAR_ISSUE | team_id, title, description |
| Search issues | LINEAR_SEARCH_ISSUES | query |
| List issues | LINEAR_LIST_LINEAR_ISSUES | team_id, filters |
| Get issue | LINEAR_GET_LINEAR_ISSUE | issue_id |
| Update issue | LINEAR_UPDATE_ISSUE | issue_id, fields |
| List states | LINEAR_LIST_LINEAR_STATES | team_id |
| List projects | LINEAR_LIST_LINEAR_PROJECTS | (none) |
| Create project | LINEAR_CREATE_LINEAR_PROJECT | name, team_ids |
| Update project | LINEAR_UPDATE_LINEAR_PROJECT | project_id, fields |
| List cycles | LINEAR_LIST_LINEAR_CYCLES | team_id |
| Get cycles | LINEAR_GET_CYCLES_BY_TEAM_ID | team_id |
| Create label | LINEAR_CREATE_LINEAR_LABEL | name, color |
| Create comment | LINEAR_CREATE_LINEAR_COMMENT | issue_id, body |
| Update comment | LINEAR_UPDATE_LINEAR_COMMENT | comment_id, body |
| List users | LINEAR_LIST_LINEAR_USERS | (none) |
| Current user | LINEAR_GET_CURRENT_USER | (none) |
| Run GraphQL | LINEAR_RUN_QUERY_OR_MUTATION | query, variables |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/monday-automation/SKILL.md

---
name: monday-automation
description: "Automate Monday.com work management including boards, items, columns, groups, subitems, and updates via Rube MCP (Composio). Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Monday.com Automation via Rube MCP

Automate Monday.com work management workflows including board creation, item management, column value updates, group organization, subitems, and update/comment threads through Composio's Monday toolkit.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Monday.com connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `monday`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `monday`
3. If connection is not ACTIVE, follow the returned auth link to complete Monday.com OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Boards

**When to use**: User wants to create a new board, list existing boards, or set up workspace structure.

**Tool sequence**:
1. `MONDAY_GET_WORKSPACES` - List available workspaces and resolve workspace ID [Prerequisite]
2. `MONDAY_LIST_BOARDS` - List existing boards to check for duplicates [Optional]
3. `MONDAY_CREATE_BOARD` - Create a new board with name, kind, and workspace [Required]
4. `MONDAY_CREATE_COLUMN` - Add columns to the new board [Optional]
5. `MONDAY_CREATE_GROUP` - Add groups to organize items [Optional]
6. `MONDAY_BOARDS` - Retrieve detailed board metadata [Optional]

**Key parameters**:
- `board_name`: Name for the new board (required)
- `board_kind`: "public", "private", or "share" (required)
- `workspace_id`: Numeric workspace ID; omit for default workspace
- `folder_id`: Folder ID; must be within `workspace_id` if both provided
- `template_id`: ID of accessible template to clone

**Pitfalls**:
- `board_kind` is required and must be one of: "public", "private", "share"
- If both `workspace_id` and `folder_id` are provided, the folder must exist within that workspace
- `template_id` must reference a template the authenticated user can access
- Board IDs are large integers; always use the exact value from API responses

### 2. Create and Manage Items

**When to use**: User wants to add tasks/items to a board, list existing items, or move items between groups.

**Tool sequence**:
1. `MONDAY_LIST_BOARDS` - Resolve board name to board ID [Prerequisite]
2. `MONDAY_LIST_GROUPS` - List groups on the board to get group_id [Prerequisite]
3. `MONDAY_LIST_COLUMNS` - Get column IDs and types for setting values [Prerequisite]
4. `MONDAY_CREATE_ITEM` - Create a new item with name and column values [Required]
5. `MONDAY_LIST_BOARD_ITEMS` - List all items on the board [Optional]
6. `MONDAY_MOVE_ITEM_TO_GROUP` - Move an item to a different group [Optional]
7. `MONDAY_ITEMS_PAGE` - Paginated item retrieval with filtering [Optional]

**Key parameters**:
- `board_id`: Board ID (required, integer)
- `item_name`: Item name, max 256 characters (required)
- `group_id`: Group ID string to place the item in (optional)
- `column_values`: JSON object or string mapping column IDs to values

**Pitfalls**:
- `column_values` must use column IDs (not titles); get them from `MONDAY_LIST_COLUMNS`
- Column value formats vary by type: status uses `{"index": 0}` or `{"label": "Done"}`, date uses `{"date": "YYYY-MM-DD"}`, people uses `{"personsAndTeams": [{"id": 123, "kind": "person"}]}`
- `item_name` has a 256-character maximum
- Subitem boards are NOT supported by `MONDAY_CREATE_ITEM`; use GraphQL via `MONDAY_CREATE_OBJECT`

### 3. Update Item Column Values

**When to use**: User wants to change status, date, text, or other column values on existing items.

**Tool sequence**:
1. `MONDAY_LIST_COLUMNS` or `MONDAY_COLUMNS` - Get column IDs and types [Prerequisite]
2. `MONDAY_LIST_BOARD_ITEMS` or `MONDAY_ITEMS_PAGE` - Find the target item ID [Prerequisite]
3. `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` - Update text, status, or dropdown with a string value [Required]
4. `MONDAY_UPDATE_ITEM` - Update complex column types (timeline, people, date) with JSON [Required]

**Key parameters for MONDAY_CHANGE_SIMPLE_COLUMN_VALUE**:
- `board_id`: Board ID (integer, required)
- `item_id`: Item ID (integer, required)
- `column_id`: Column ID string (required)
- `value`: Simple string value (e.g., "Done", "Working on it")
- `create_labels_if_missing`: true to auto-create status/dropdown labels (default true)

**Key parameters for MONDAY_UPDATE_ITEM**:
- `board_id`: Board ID (integer, required)
- `item_id`: Item ID (integer, required)
- `column_id`: Column ID string (required)
- `value`: JSON object matching the column type schema
- `create_labels_if_missing`: false by default; set true for status/dropdown

**Pitfalls**:
- Use `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` for simple text/status/dropdown updates (string value)
- Use `MONDAY_UPDATE_ITEM` for complex types like timeline, people, date (JSON value)
- Column IDs are lowercase strings with underscores (e.g., "status_1", "date_2", "text"); get them from `MONDAY_LIST_COLUMNS`
- Status values can be set by label name ("Done") or index number ("1")
- `create_labels_if_missing` defaults differ: true for CHANGE_SIMPLE, false for UPDATE_ITEM

### 4. Work with Groups and Board Structure

**When to use**: User wants to organize items into groups, add columns, or inspect board structure.

**Tool sequence**:
1. `MONDAY_LIST_BOARDS` - Resolve board ID [Prerequisite]
2. `MONDAY_LIST_GROUPS` - List all groups on a board [Required]
3. `MONDAY_CREATE_GROUP` - Create a new group [Optional]
4. `MONDAY_LIST_COLUMNS` or `MONDAY_COLUMNS` - Inspect column structure [Required]
5. `MONDAY_CREATE_COLUMN` - Add a new column to the board [Optional]
6. `MONDAY_MOVE_ITEM_TO_GROUP` - Reorganize items across groups [Optional]

**Key parameters**:
- `board_id`: Board ID (required for all group/column operations)
- `group_name`: Name for new group (CREATE_GROUP)
- `column_type`: Must be a valid GraphQL enum token in snake_case (e.g., "status", "text", "long_text", "numbers", "date", "dropdown", "people")
- `title`: Column display title
- `defaults`: JSON string for status/dropdown labels, e.g., `'{"labels": ["To Do", "In Progress", "Done"]}'`

**Pitfalls**:
- `column_type` must be exact snake_case values; "person" is NOT valid, use "people"
- Group IDs are strings (e.g., "topics", "new_group_12345"), not integers
- `MONDAY_COLUMNS` accepts an array of `board_ids` and returns column metadata including settings
- `MONDAY_LIST_COLUMNS` is simpler and takes a single `board_id`

### 5. Manage Subitems and Updates

**When to use**: User wants to view subitems of a task or add comments/updates to items.

**Tool sequence**:
1. `MONDAY_LIST_BOARD_ITEMS` - Find parent item IDs [Prerequisite]
2. `MONDAY_LIST_SUBITEMS_BY_PARENT` - Retrieve subitems with column values [Required]
3. `MONDAY_CREATE_UPDATE` - Add a comment/update to an item [Optional]
4. `MONDAY_CREATE_OBJECT` - Create subitems via GraphQL mutation [Optional]

**Key parameters for MONDAY_LIST_SUBITEMS_BY_PARENT**:
- `parent_item_ids`: Array of parent item IDs (integer array, required)
- `include_column_values`: true to include column data (default true)
- `include_parent_fields`: true to include parent item info (default true)

**Key parameters for MONDAY_CREATE_OBJECT** (GraphQL):
- `query`: Full GraphQL mutation string
- `variables`: Optional variables object

**Pitfalls**:
- Subitems can only be queried through their parent items
- To create subitems, use `MONDAY_CREATE_OBJECT` with a `create_subitem` GraphQL mutation
- `MONDAY_CREATE_UPDATE` is for adding comments/updates to items (Monday's "updates" feature), not for modifying item values
- `MONDAY_CREATE_OBJECT` is a raw GraphQL endpoint; ensure correct mutation syntax

## Common Patterns

### ID Resolution
Always resolve display names to IDs before operations:
- **Board name -> board_id**: `MONDAY_LIST_BOARDS` and match by name
- **Group name -> group_id**: `MONDAY_LIST_GROUPS` with `board_id`
- **Column title -> column_id**: `MONDAY_LIST_COLUMNS` with `board_id`
- **Workspace name -> workspace_id**: `MONDAY_GET_WORKSPACES` and match by name
- **Item name -> item_id**: `MONDAY_LIST_BOARD_ITEMS` or `MONDAY_ITEMS_PAGE`

### Pagination
Monday.com uses cursor-based pagination for items:
- `MONDAY_ITEMS_PAGE` returns a `cursor` in the response for the next page
- Pass the `cursor` to the next call; `board_id` and `query_params` are ignored when cursor is provided
- Cursors are cached for 60 minutes
- Maximum `limit` is 500 per page
- `MONDAY_LIST_BOARDS` and `MONDAY_GET_WORKSPACES` use page-based pagination with `page` and `limit`

### Column Value Formatting
Different column types require different value formats:
- **Status**: `{"index": 0}` or `{"label": "Done"}` or simple string "Done"
- **Date**: `{"date": "YYYY-MM-DD"}`
- **People**: `{"personsAndTeams": [{"id": 123, "kind": "person"}]}`
- **Text/Numbers**: Plain string or number
- **Timeline**: `{"from": "YYYY-MM-DD", "to": "YYYY-MM-DD"}`

## Known Pitfalls

### ID Formats
- Board IDs and item IDs are large integers (e.g., 1234567890)
- Group IDs are strings (e.g., "topics", "new_group_12345")
- Column IDs are short strings (e.g., "status_1", "date4", "text")
- Workspace IDs are integers

### Rate Limits
- Monday.com GraphQL API has complexity-based rate limits
- Large boards with many columns increase query complexity
- Use `limit` parameter to reduce items per request if hitting limits

### Parameter Quirks
- `column_type` for CREATE_COLUMN must be exact snake_case enum values; "people" not "person"
- `column_values` in CREATE_ITEM accepts both JSON string and object formats
- `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` auto-creates missing labels by default; `MONDAY_UPDATE_ITEM` does not
- `MONDAY_CREATE_OBJECT` is a raw GraphQL interface; use it for operations without dedicated tools (e.g., create_subitem, delete_item, archive_board)

### Response Structure
- Board items are returned as arrays with `id`, `name`, and `state` fields
- Column values include both raw `value` (JSON) and rendered `text` (display string)
- Subitems are nested under parent items and cannot be queried independently

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List workspaces | `MONDAY_GET_WORKSPACES` | `kind`, `state`, `limit` |
| Create workspace | `MONDAY_CREATE_WORKSPACE` | `name`, `kind` |
| List boards | `MONDAY_LIST_BOARDS` | `limit`, `page`, `state` |
| Create board | `MONDAY_CREATE_BOARD` | `board_name`, `board_kind`, `workspace_id` |
| Get board metadata | `MONDAY_BOARDS` | `board_ids`, `board_kind` |
| List groups | `MONDAY_LIST_GROUPS` | `board_id` |
| Create group | `MONDAY_CREATE_GROUP` | `board_id`, `group_name` |
| List columns | `MONDAY_LIST_COLUMNS` | `board_id` |
| Get column metadata | `MONDAY_COLUMNS` | `board_ids`, `column_types` |
| Create column | `MONDAY_CREATE_COLUMN` | `board_id`, `column_type`, `title` |
| Create item | `MONDAY_CREATE_ITEM` | `board_id`, `item_name`, `column_values` |
| List board items | `MONDAY_LIST_BOARD_ITEMS` | `board_id` |
| Paginated items | `MONDAY_ITEMS_PAGE` | `board_id`, `limit`, `query_params` |
| Update column (simple) | `MONDAY_CHANGE_SIMPLE_COLUMN_VALUE` | `board_id`, `item_id`, `column_id`, `value` |
| Update column (complex) | `MONDAY_UPDATE_ITEM` | `board_id`, `item_id`, `column_id`, `value` |
| Move item to group | `MONDAY_MOVE_ITEM_TO_GROUP` | `item_id`, `group_id` |
| List subitems | `MONDAY_LIST_SUBITEMS_BY_PARENT` | `parent_item_ids` |
| Add comment/update | `MONDAY_CREATE_UPDATE` | `item_id`, `body` |
| Raw GraphQL mutation | `MONDAY_CREATE_OBJECT` | `query`, `variables` |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/trello-automation/SKILL.md

---
name: trello-automation
description: "Automate Trello boards, cards, and workflows via Rube MCP (Composio). Create cards, manage lists, assign members, and search across boards programmatically."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Trello Automation via Rube MCP

Automate Trello board management, card creation, and team workflows through Composio's Rube MCP integration.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Trello connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `trello`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `trello`
3. If connection is not ACTIVE, follow the returned auth link to complete Trello auth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create a Card on a Board

**When to use**: User wants to add a new card/task to a Trello board

**Tool sequence**:
1. `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` - List boards to find target board ID [Prerequisite]
2. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get lists on board to find target list ID [Prerequisite]
3. `TRELLO_ADD_CARDS` - Create the card on the resolved list [Required]
4. `TRELLO_ADD_CARDS_CHECKLISTS_BY_ID_CARD` - Add a checklist to the card [Optional]
5. `TRELLO_ADD_CARDS_CHECKLIST_CHECK_ITEM_BY_ID_CARD_BY_ID_CHECKLIST` - Add items to the checklist [Optional]

**Key parameters**:
- `idList`: 24-char hex ID (NOT list name)
- `name`: Card title
- `desc`: Card description (supports Markdown)
- `pos`: Position ('top'/'bottom')
- `due`: Due date (ISO 8601 format)

**Pitfalls**:
- Store returned id (idCard) immediately; downstream checklist operations fail without it
- Checklist payload may be nested (data.data); extract idChecklist from inner object
- One API call per checklist item; large checklists can trigger rate limits

### 2. Manage Boards and Lists

**When to use**: User wants to view, browse, or restructure board layout

**Tool sequence**:
1. `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` - List all boards for the user [Required]
2. `TRELLO_GET_BOARDS_BY_ID_BOARD` - Get detailed board info [Required]
3. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get lists (columns) on the board [Optional]
4. `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD` - Get board members [Optional]
5. `TRELLO_GET_BOARDS_LABELS_BY_ID_BOARD` - Get labels on the board [Optional]

**Key parameters**:
- `idMember`: Use 'me' for authenticated user
- `filter`: 'open', 'starred', or 'all'
- `idBoard`: 24-char hex or 8-char shortLink (NOT board name)

**Pitfalls**:
- Some runs return boards under response.data.details[]—don't assume flat top-level array
- Lists may be nested under results[0].response.data.details—parse defensively
- ISO 8601 timestamps with trailing 'Z' must be parsed as timezone-aware

### 3. Move Cards Between Lists

**When to use**: User wants to change a card's status by moving it to another list

**Tool sequence**:
1. `TRELLO_GET_SEARCH` - Find the card by name or keyword [Prerequisite]
2. `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` - Get destination list ID [Prerequisite]
3. `TRELLO_UPDATE_CARDS_BY_ID_CARD` - Update card's idList to move it [Required]

**Key parameters**:
- `idCard`: Card ID from search
- `idList`: Destination list ID
- `pos`: Optional ordering within new list

**Pitfalls**:
- Search returns partial matches; verify card name before updating
- Moving doesn't update position within new list; set pos if ordering matters

### 4. Assign Members to Cards

**When to use**: User wants to assign team members to cards

**Tool sequence**:
1. `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD` - Get member IDs from the board [Prerequisite]
2. `TRELLO_ADD_CARDS_ID_MEMBERS_BY_ID_CARD` - Add a member to the card [Required]

**Key parameters**:
- `idCard`: Target card ID
- `value`: Member ID to assign

**Pitfalls**:
- UPDATE_CARDS_ID_MEMBERS replaces entire member list; use ADD_CARDS_ID_MEMBERS to append
- Member must have board permissions

### 5. Search and Filter Cards

**When to use**: User wants to find specific cards across boards

**Tool sequence**:
1. `TRELLO_GET_SEARCH` - Search by query string [Required]

**Key parameters**:
- `query`: Search string (supports board:, list:, label:, is:open/archived operators)
- `modelTypes`: Set to 'cards'
- `partial`: Set to 'true' for prefix matching

**Pitfalls**:
- Search indexing has delay; newly created cards may not appear for several minutes
- For exact name matching, use TRELLO_GET_BOARDS_CARDS_BY_ID_BOARD and filter locally
- Query uses word tokenization; common words may be ignored as stop words

### 6. Add Comments and Attachments

**When to use**: User wants to add context to an existing card

**Tool sequence**:
1. `TRELLO_ADD_CARDS_ACTIONS_COMMENTS_BY_ID_CARD` - Post a comment on the card [Required]
2. `TRELLO_ADD_CARDS_ATTACHMENTS_BY_ID_CARD` - Attach a file or URL [Optional]

**Key parameters**:
- `text`: Comment text (1-16384 chars, supports Markdown and @mentions)
- `url` OR `file`: Attachment source (not both)
- `name`: Attachment display name
- `mimeType`: File MIME type

**Pitfalls**:
- Comments don't support file attachments; use the attachment tool separately
- Attachment deletion is irreversible

## Common Patterns

### ID Resolution
Always resolve display names to IDs before operations:
- **Board name → Board ID**: `TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER` with idMember='me'
- **List name → List ID**: `TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD` with resolved board ID
- **Card name → Card ID**: `TRELLO_GET_SEARCH` with query string
- **Member name → Member ID**: `TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD`

### Pagination
Most list endpoints return all items. For boards with 1000+ cards, use `limit` and `before` parameters on card listing endpoints.

### Rate Limits
300 requests per 10 seconds per token. Use `TRELLO_GET_BATCH` for bulk read operations to stay within limits.

## Known Pitfalls

- **ID Requirements**: Nearly every tool requires IDs, not display names. Always resolve names to IDs first.
- **Board ID Format**: Board IDs must be 24-char hex or 8-char shortLink. URL slugs like 'my-board' are NOT valid.
- **Search Delays**: Search indexing has delays; newly created/updated cards may not appear immediately.
- **Nested Responses**: Response data is often nested (data.data or data.details[]); parse defensively.
- **Rate Limiting**: 300 req/10s per token. Batch reads with TRELLO_GET_BATCH.

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| List user's boards | TRELLO_GET_MEMBERS_BOARDS_BY_ID_MEMBER | idMember='me', filter='open' |
| Get board details | TRELLO_GET_BOARDS_BY_ID_BOARD | idBoard (24-char hex) |
| List board lists | TRELLO_GET_BOARDS_LISTS_BY_ID_BOARD | idBoard |
| Create card | TRELLO_ADD_CARDS | idList, name, desc, pos, due |
| Update card | TRELLO_UPDATE_CARDS_BY_ID_CARD | idCard, idList (to move) |
| Search cards | TRELLO_GET_SEARCH | query, modelTypes='cards' |
| Add checklist | TRELLO_ADD_CARDS_CHECKLISTS_BY_ID_CARD | idCard, name |
| Add comment | TRELLO_ADD_CARDS_ACTIONS_COMMENTS_BY_ID_CARD | idCard, text |
| Assign member | TRELLO_ADD_CARDS_ID_MEMBERS_BY_ID_CARD | idCard, value (member ID) |
| Attach file/URL | TRELLO_ADD_CARDS_ATTACHMENTS_BY_ID_CARD | idCard, url OR file |
| Get board members | TRELLO_GET_BOARDS_MEMBERS_BY_ID_BOARD | idBoard |
| Batch read | TRELLO_GET_BATCH | urls (comma-separated paths) |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

## Source: references/skills/github-automation/references/legacy/wrike-automation/SKILL.md

---
name: wrike-automation
description: "Automate Wrike project management via Rube MCP (Composio): create tasks/folders, manage projects, assign work, and track progress. Always search tools first for current schemas."
risk: unknown
source: community
date_added: "2026-02-27"
---

# Wrike Automation via Rube MCP

Automate Wrike project management operations through Composio's Wrike toolkit via Rube MCP.

## Prerequisites

- Rube MCP must be connected (RUBE_SEARCH_TOOLS available)
- Active Wrike connection via `RUBE_MANAGE_CONNECTIONS` with toolkit `wrike`
- Always call `RUBE_SEARCH_TOOLS` first to get current tool schemas

## Setup

**Get Rube MCP**: Add `https://rube.app/mcp` as an MCP server in your client configuration. No API keys needed — just add the endpoint and it works.


1. Verify Rube MCP is available by confirming `RUBE_SEARCH_TOOLS` responds
2. Call `RUBE_MANAGE_CONNECTIONS` with toolkit `wrike`
3. If connection is not ACTIVE, follow the returned auth link to complete Wrike OAuth
4. Confirm connection status shows ACTIVE before running any workflows

## Core Workflows

### 1. Create and Manage Tasks

**When to use**: User wants to create, assign, or update tasks in Wrike

**Tool sequence**:
1. `WRIKE_GET_FOLDERS` - Find the target folder/project [Prerequisite]
2. `WRIKE_GET_ALL_CUSTOM_FIELDS` - Get custom field IDs if needed [Optional]
3. `WRIKE_CREATE_TASK` - Create a new task [Required]
4. `WRIKE_MODIFY_TASK` - Update task properties [Optional]

**Key parameters**:
- `folderId`: Parent folder ID where the task will be created
- `title`: Task title
- `description`: Task description (supports HTML)
- `responsibles`: Array of user IDs to assign
- `status`: 'Active', 'Completed', 'Deferred', 'Cancelled'
- `importance`: 'High', 'Normal', 'Low'
- `customFields`: Array of {id, value} objects
- `dates`: Object with type, start, due, duration

**Pitfalls**:
- folderId is required; tasks must belong to a folder
- responsibles requires Wrike user IDs, not emails or names
- Custom field IDs must be obtained from GET_ALL_CUSTOM_FIELDS
- priorityBefore and priorityAfter are mutually exclusive
- Status field may not be available on Team plan
- dates.start and dates.due use 'YYYY-MM-DD' format

### 2. Manage Folders and Projects

**When to use**: User wants to create, modify, or organize folders and projects

**Tool sequence**:
1. `WRIKE_GET_FOLDERS` - List existing folders [Required]
2. `WRIKE_CREATE_FOLDER` - Create a new folder/project [Optional]
3. `WRIKE_MODIFY_FOLDER` - Update folder properties [Optional]
4. `WRIKE_LIST_SUBFOLDERS_BY_FOLDER_ID` - List subfolders [Optional]
5. `WRIKE_DELETE_FOLDER` - Delete a folder permanently [Optional]

**Key parameters**:
- `folderId`: Parent folder ID for creation; target folder ID for modification
- `title`: Folder name
- `description`: Folder description
- `customItemTypeId`: Set to create as a project instead of a folder
- `shareds`: Array of user IDs or emails to share with
- `project`: Filter for projects (true) or folders (false) in GET_FOLDERS

**Pitfalls**:
- DELETE_FOLDER is permanent and removes ALL contents (tasks, subfolders, documents)
- Cannot modify rootFolderId or recycleBinId as parents
- Folder creation auto-shares with the creator
- customItemTypeId converts a folder into a project
- GET_FOLDERS with descendants=true returns folder tree (may be large)

### 3. Retrieve and Track Tasks

**When to use**: User wants to find tasks, check status, or monitor progress

**Tool sequence**:
1. `WRIKE_FETCH_ALL_TASKS` - List tasks with optional filters [Required]
2. `WRIKE_GET_TASK_BY_ID` - Get detailed info for a specific task [Optional]

**Key parameters**:
- `status`: Filter by task status ('Active', 'Completed', etc.)
- `dueDate`: Filter by due date range (start/end/equal)
- `fields`: Additional response fields to include
- `page_size`: Results per page (1-100)
- `taskId`: Specific task ID for detailed retrieval
- `resolve_user_names`: Auto-resolve user IDs to names (default true)

**Pitfalls**:
- FETCH_ALL_TASKS paginates at max 100 items per page
- dueDate filter supports 'equal', 'start', and 'end' fields
- Date format: 'yyyy-MM-dd' or 'yyyy-MM-ddTHH:mm:ss'
- GET_TASK_BY_ID returns read-only detailed information
- customFields are returned by default for single task queries

### 4. Launch Task Blueprints

**When to use**: User wants to create tasks from predefined templates

**Tool sequence**:
1. `WRIKE_LIST_TASK_BLUEPRINTS` - List available blueprints [Prerequisite]
2. `WRIKE_LIST_SPACE_TASK_BLUEPRINTS` - List blueprints in a specific space [Alternative]
3. `WRIKE_LAUNCH_TASK_BLUEPRINT_ASYNC` - Launch a blueprint [Required]

**Key parameters**:
- `task_blueprint_id`: ID of the blueprint to launch
- `title`: Title for the root task
- `parent_id`: Parent folder/project ID (OR super_task_id)
- `super_task_id`: Parent task ID (OR parent_id)
- `reschedule_date`: Target date for task rescheduling
- `reschedule_mode`: 'RescheduleStartDate' or 'RescheduleFinishDate'
- `entry_limit`: Max tasks to copy (1-250)

**Pitfalls**:
- Either parent_id or super_task_id is required, not both
- Blueprint launch is asynchronous; tasks may take time to appear
- reschedule_date requires reschedule_mode to be set
- entry_limit caps at 250 tasks/folders per blueprint launch
- copy_descriptions defaults to false; set true to include task descriptions

### 5. Manage Workspace and Members

**When to use**: User wants to manage spaces, members, or invitations

**Tool sequence**:
1. `WRIKE_GET_SPACE` - Get space details [Optional]
2. `WRIKE_GET_CONTACTS` - List workspace contacts/members [Optional]
3. `WRIKE_CREATE_INVITATION` - Invite a user to the workspace [Optional]
4. `WRIKE_DELETE_SPACE` - Delete a space permanently [Optional]

**Key parameters**:
- `spaceId`: Space identifier
- `email`: Email for invitation
- `role`: User role ('Admin', 'Regular User', 'External User')
- `firstName`/`lastName`: Invitee name

**Pitfalls**:
- DELETE_SPACE is irreversible and removes all space contents
- userTypeId and role/external are mutually exclusive in invitations
- Custom email subjects/messages require a paid Wrike plan
- GET_CONTACTS returns workspace-level contacts, not task-specific assignments

## Common Patterns

### Folder ID Resolution

```
1. Call WRIKE_GET_FOLDERS (optionally with project=true for projects only)
2. Navigate folder tree to find target
3. Extract folder id (e.g., 'IEAGKVLFK4IHGQOI')
4. Use as folderId in task/folder creation
```

### Custom Field Setup

```
1. Call WRIKE_GET_ALL_CUSTOM_FIELDS to get definitions
2. Find field by name, extract id and type
3. Format value according to type (text, dropdown, number, date)
4. Include as {id: 'FIELD_ID', value: 'VALUE'} in customFields array
```

### Task Assignment

```
1. Call WRIKE_GET_CONTACTS to find user IDs
2. Use user IDs in responsibles array when creating tasks
3. Or use addResponsibles/removeResponsibles when modifying tasks
```

### Pagination

- FETCH_ALL_TASKS: Use page_size (max 100) and check for more results
- GET_FOLDERS: Use nextPageToken when descendants=false and pageSize is set
- LIST_TASK_BLUEPRINTS: Use next_page_token and page_size (default 100)

## Known Pitfalls

**ID Formats**:
- Wrike IDs are opaque alphanumeric strings (e.g., 'IEAGTXR7I4IHGABC')
- Task IDs, folder IDs, space IDs, and user IDs all use this format
- Custom field IDs follow the same pattern
- Never guess IDs; always resolve from list/search operations

**Permissions**:
- Operations depend on user role and sharing settings
- Shared folders/tasks are visible only to shared users
- Admin operations require appropriate role
- Some features (custom statuses, billing types) are plan-dependent

**Deletion Safety**:
- DELETE_FOLDER removes ALL contents permanently
- DELETE_SPACE removes the entire space and contents
- Consider using MODIFY_FOLDER to move to recycle bin instead
- Restore from recycle bin is possible via MODIFY_FOLDER with restore=true

**Date Handling**:
- Dates use 'yyyy-MM-dd' format
- DateTime uses 'yyyy-MM-ddTHH:mm:ssZ' or with timezone offset
- Task dates include type ('Planned', 'Actual'), start, due, duration
- Duration is in minutes

## Quick Reference

| Task | Tool Slug | Key Params |
|------|-----------|------------|
| Create task | WRIKE_CREATE_TASK | folderId, title, responsibles, status |
| Modify task | WRIKE_MODIFY_TASK | taskId, title, status, addResponsibles |
| Get task by ID | WRIKE_GET_TASK_BY_ID | taskId |
| Fetch all tasks | WRIKE_FETCH_ALL_TASKS | status, dueDate, page_size |
| Get folders | WRIKE_GET_FOLDERS | project, descendants |
| Create folder | WRIKE_CREATE_FOLDER | folderId, title |
| Modify folder | WRIKE_MODIFY_FOLDER | folderId, title, addShareds |
| Delete folder | WRIKE_DELETE_FOLDER | folderId |
| List subfolders | WRIKE_LIST_SUBFOLDERS_BY_FOLDER_ID | folderId |
| Get custom fields | WRIKE_GET_ALL_CUSTOM_FIELDS | (none) |
| List blueprints | WRIKE_LIST_TASK_BLUEPRINTS | limit, page_size |
| Launch blueprint | WRIKE_LAUNCH_TASK_BLUEPRINT_ASYNC | task_blueprint_id, title, parent_id |
| Get space | WRIKE_GET_SPACE | spaceId |
| Delete space | WRIKE_DELETE_SPACE | spaceId |
| Get contacts | WRIKE_GET_CONTACTS | (none) |
| Invite user | WRIKE_CREATE_INVITATION | email, role |

## When to Use
This skill is applicable to execute the workflow or actions described in the overview.

