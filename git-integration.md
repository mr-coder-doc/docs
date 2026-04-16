# Git Integration — Developer Documentation

## Overview

BetterDocs Git Integration enables bidirectional synchronization between your WordPress documentation and a Git repository. Write docs in WordPress and push to Git, or maintain docs in Git and pull into WordPress. Supports both **GitHub** (one-click OAuth) and **GitLab** (access token).

## Provider Comparison

| | GitHub | GitLab |
|---|---|---|
| **Authentication** | One-click OAuth via proxy | Manual Project ID + Access Token |
| **Setup complexity** | Click "Connect with GitHub" | Create token in GitLab, paste credentials |
| **Repo selection** | Dropdown (auto-populated) | User enters Project ID |
| **Self-hosted** | Not supported | Supported via custom instance URL |
| **Token expiry** | No expiry (OAuth) | 365 days max (GitLab 16.0+) |
| **Proxy required** | Yes (betterdocs-auth) | No |

---

## Settings Reference

All settings are under **BetterDocs Settings > Git Integration** tab.

### Enable Git Integration

| Type | Default |
|------|---------|
| Toggle | Off |

Master switch for the entire feature. When disabled, no Git-related UI appears anywhere — not in settings, not in the editor toolbar. Enabling this:
- Shows the Repo Settings panel and other Git settings
- Adds the Git Sync button to the Gutenberg editor toolbar on docs
- Registers all Git-related AJAX handlers and webhooks

### Repo Settings (GitHub / GitLab)

This is not a single setting but a composite UI component that manages four underlying settings:

| Setting Key | Stored In | Purpose |
|-------------|-----------|---------|
| `git_provider` | `betterdocs_settings` | `'github'` or `'gitlab'` |
| `git_repository_url` | `betterdocs_settings` | `owner/repo` (GitHub) or project ID (GitLab) |
| `git_branch` | `betterdocs_settings` | Branch name (e.g., `main`) |
| `git_docs_directory` | `betterdocs_settings` | Directory path in repo (e.g., `docs`) |

These values are saved immediately when selected — they don't wait for the "Save Settings" button. This is because they're saved via a dedicated AJAX endpoint (`betterdocs_git_save_repo_setting`) that writes directly to the `betterdocs_settings` option, bypassing the quickbuilder form framework.

#### GitHub Tab

After clicking "Connect with GitHub":
1. OAuth popup opens → user authorizes → popup closes
2. **Repository** dropdown populates with all repos the user can push to (grouped by owner/org)
3. **Branch** dropdown populates when a repo is selected (default branch auto-selected)
4. **Docs directory** dropdown shows folders at the repo root

#### GitLab Tab

Manual credential entry:
1. **Instance URL** (optional) — defaults to `https://gitlab.com`. Set to your self-hosted GitLab URL if applicable.
2. **Project ID** (required) — numeric ID found in GitLab → Settings → General.
3. **Access Token** (required) — personal or project access token with `api` + `read_api` scopes.
4. Click "Connect GitLab" → plugin validates credentials against GitLab API
5. **Branch** and **Docs directory** dropdowns appear after successful connection

### File Naming Convention

| Type | Default | Options |
|------|---------|---------|
| Select | `slug` | Use Post Slug (recommended), Use Post ID, Use Post Title |

Controls how markdown files are named in the repository:

- **Post Slug** (recommended): `getting-started.md` — clean, URL-friendly, matches WordPress permalinks
- **Post ID**: `123.md` — guaranteed unique but not human-readable
- **Post Title**: `Getting Started Guide.md` — readable but may contain special characters

**Draft/unsaved fallback**: If a doc has no slug or title yet (e.g., a new draft), all naming modes fall back to `doc-{post_id}.md` to prevent empty filenames.

The docs directory + file naming combine to form the full path. Example with slug naming and `docs/` directory: `docs/getting-started.md`.

### Auto Sync on Save

| Type | Default |
|------|---------|
| Toggle | Off |

When enabled, saving a doc in WordPress automatically pushes it to the Git repository. The sync happens in the background via `wp_schedule_single_event` so it doesn't block the editor.

**How it works:**
1. User saves/updates a doc in WordPress
2. `save_post` hook fires
3. If the doc has `_betterdocs_git_sync_enabled` meta set to true, a background sync is scheduled
4. The sync converts content to markdown, commits to the configured branch

**When to enable:** Good for teams that want Git to always mirror WordPress. Not recommended if you primarily edit in Git and pull to WordPress (would cause circular sync loops).

---

## Editor Sync Panel

When Git Integration is enabled, a **GitHub Sync** or **GitLab Sync** button appears in the Gutenberg editor toolbar for docs, next to the "Write with AI" button.

Clicking the button opens a dropdown panel showing:

- **Repository chip** — connected repo name
- **Branch chip** — configured branch
- **Diff viewer** — shows changes between WordPress content and the Git file (additions in green, deletions in red)
- **Commit message** input (required) — write a meaningful commit message. Push is blocked with a red highlight if empty.
- **Push button** — converts the doc to markdown and commits to Git. Detects identical content and shows "No changes to push" instead of creating a duplicate commit. Shows SweetAlert2 success toast on completion.
- **Pull button** — fetches the markdown from Git and updates the WordPress content. Shows SweetAlert2 success toast before page reload.
- **Import MD Files** — lists all `.md` files in the configured directory for bulk import. Shows overwrite confirmation (SweetAlert2) when importing files that already exist as docs. Single-file import redirects to the post editor; multi-file import shows a count summary and redirects to All Docs.
- **Last sync info** — commit hash and time since last sync

All notifications in the editor dropdown use **SweetAlert2** (`Swal.fire()`) with `customClass: { popup: 'betterdocs-bulk-action-popup' }`, matching the pattern used in the settings page and metabox. SweetAlert2 is enqueued on the docs editor screen by `GitHubSync::enqueue_sweetalert()`.

### New Document Behavior

On `post-new.php` (unsaved doc), the sync panel appears but shows "Save the document first to enable sync". Push/Pull buttons are disabled until the doc is saved and has a post ID.

---

## Architecture

### Auth Proxy (GitHub only)

GitHub OAuth requires a Client ID and Secret. These are stored on a separate auth proxy server (`betterdocs-auth`) — **not in the WordPress plugin**. The proxy:

1. Redirects the user to GitHub's OAuth authorize page
2. Receives the authorization code callback
3. Exchanges the code for an access token (using stored credentials)
4. Stores the token in a one-time file
5. Redirects back to WordPress with a retrieval code
6. WordPress exchanges the code for the actual token via a server-to-server POST

The token is then stored in `wp_options` as `betterdocs_github_oauth_token`. The proxy URL defaults to `https://api.betterdocs.co` and can be overridden for local dev via the `betterdocs_github_auth_proxy_url` filter.

### GitLab (No Proxy)

GitLab connections are direct. The user provides a personal/project access token, which the plugin validates against the GitLab API and stores in the same `betterdocs_github_oauth_token` option. All GitLab API calls use the `PRIVATE-TOKEN` header.

### Core Classes

| Class | File | Purpose |
|-------|------|---------|
| `GitIntegration` | `includes/Core/GitIntegration.php` | Main sync engine — push, pull, diff, markdown conversion, webhooks |
| `GitHubOAuth` | `includes/Core/GitHubOAuth.php` | OAuth flow, token management, API relay to proxy, GitLab direct auth |
| `GitHubSync` | `includes/Core/GitHubSync.php` | Editor toolbar button + dropdown panel (JS injection via admin_footer). Enqueues SweetAlert2 on docs editor. All notifications use Swal helpers. |
| `GitIntegrationTest` | `includes/Admin/GitIntegrationTest.php` | Admin test interface for debugging |

### Markdown Conversion

**WordPress → Git (Push):**
- HTML headings → `#` / `##` / `###` etc.
- `<strong>` / `<em>` → `**bold**` / `*italic*`
- `<a href>` → `[text](url)`
- `<img>` → `![alt](src)`
- `<ul>` / `<ol>` → bullet/numbered lists
- `<code>` / `<pre>` → backticks/fenced code blocks
- Front matter added: title, slug, categories, tags, author, date

**Git → WordPress (Pull):**
- Markdown parsed back to HTML
- Front matter extracted (can update title, categories if present)
- Code blocks, lists, headings, links all converted back

### Database

**Options:**
| Key | Content |
|-----|---------|
| `betterdocs_github_oauth_token` | Access token (GitHub OAuth or GitLab PAT) |
| `betterdocs_github_oauth_user` | User info object with `provider`, `login`, `name`, `avatar_url`, `html_url`, and GitLab-specific `project_id`, `instance_url`, `project_name` |

**Post Meta (per doc):**
| Key | Type | Purpose |
|-----|------|---------|
| `_betterdocs_git_sync_enabled` | bool | Per-doc sync toggle |
| `_betterdocs_git_sync_status` | string | `pending`, `syncing`, `synced`, `error`, `conflict` |
| `_betterdocs_git_last_sync` | datetime | Last successful sync timestamp |
| `_betterdocs_git_file_path` | string | File path in repository |
| `_betterdocs_git_commit_hash` | string | Last synced commit hash |

**Settings (in `betterdocs_settings` option):**
| Key | Default | Description |
|-----|---------|-------------|
| `enable_git_integration` | `false` | Master enable |
| `git_provider` | `'github'` | `'github'` or `'gitlab'` |
| `git_repository_url` | `''` | `owner/repo` or GitLab project ID |
| `git_branch` | `'main'` | Target branch |
| `git_docs_directory` | `'docs'` | Docs directory in repo |
| `git_file_naming` | `'slug'` | `'slug'`, `'id'`, or `'title'` |
| `git_auto_sync` | `false` | Auto-sync on save |

---

## AJAX Endpoints

### Connection

| Action | Method | Purpose |
|--------|--------|---------|
| `betterdocs_git_oauth_start` | POST | Start GitHub OAuth (param: `provider`) |
| `betterdocs_github_oauth_callback` | GET | Receive one-time code from proxy |
| `betterdocs_gitlab_connect` | POST | Validate GitLab token + project |
| `betterdocs_git_oauth_disconnect` | POST | Clear stored token + user |
| `betterdocs_git_get_user` | POST | Get connected user info |
| `betterdocs_git_providers` | POST | Available providers |
| `betterdocs_git_save_repo_setting` | POST | Save repo/branch/directory directly |

### Repository API

| Action | Method | Purpose |
|--------|--------|---------|
| `betterdocs_git_repos` | POST | List repos (GitHub via proxy) |
| `betterdocs_git_branches` | GET | List branches (auto-routes by provider) |
| `betterdocs_git_contents` | GET | List directories (auto-routes by provider) |

### Sync Operations

| Action | Method | Purpose |
|--------|--------|---------|
| `betterdocs_git_sync_document` | POST | Push doc to Git |
| `betterdocs_git_pull_document` | POST | Pull doc from Git |
| `betterdocs_git_push_if_missing` | POST | Smart push if file missing |
| `betterdocs_git_test_connection` | POST | Test Git connection |
| `betterdocs_github_get_diff` | POST | Get diff between WP and Git content |
| `betterdocs_github_list_md_files` | POST | List markdown files in repo |
| `betterdocs_github_import_md` | POST | Import markdown files as docs |

---

## Webhook

**Endpoint:** `?betterdocs_git_webhook=1`

The webhook allows Git to notify WordPress when changes are pushed to the repository. Configure it in your Git provider's settings:

**GitHub:** Repository → Settings → Webhooks → Add webhook
- Payload URL: `https://your-site.com/?betterdocs_git_webhook=1`
- Content type: `application/json`
- Secret: (optional, match `git_webhook_secret` setting)
- Events: Push events

**GitLab:** Project → Settings → Webhooks
- URL: `https://your-site.com/?betterdocs_git_webhook=1`
- Secret token: (optional)
- Trigger: Push events

---

## Hooks & Filters

| Hook | Type | Purpose |
|------|------|---------|
| `betterdocs_github_auth_proxy_url` | filter | Override proxy URL (default: `https://api.betterdocs.co`) |
| `betterdocs_git_markdown_content` | filter | Modify markdown before pushing to Git |
| `betterdocs_git_html_content` | filter | Modify HTML after pulling from Git |
| `betterdocs_git_file_path` | filter | Override file path in repository |
| `betterdocs_git_commit_message` | filter | Modify auto-generated commit messages |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Sync button not showing | Verify `enable_git_integration` is ON and you're editing a `docs` post type |
| GitHub connect popup closes without connecting | Check proxy is running and accessible (default: `https://api.betterdocs.co`) |
| "Connecting…" stuck after closing popup | Fixed — popup.closed polling resets the UI state automatically |
| Push succeeds with no changes | Fixed — content is compared before commit; shows "No changes to push" |
| Duplicate docs created on import | Fixed — import checks `_betterdocs_git_file_path` meta before slug match |
| Files pushed as just `.md` (no name) | Fixed — drafts/empty slugs fall back to `doc-{ID}.md` |
| GitLab "Invalid access token" | Regenerate token in GitLab → Settings → Access Tokens with `api` + `read_api` scopes |
| GitLab "Project not found" | Verify the Project ID (numeric) from GitLab → Settings → General |
| Push returns 404 | Repository URL format may be wrong. GitHub uses `owner/repo`, GitLab uses numeric project ID |
| Push returns 403 | Token lacks push permissions. GitHub needs `repo` scope, GitLab needs `api` scope with Maintainer role |
| Diff shows "No local changes" | Content is identical between WP and Git. Make an edit and try again |
| Auto-sync not triggering | Check `git_auto_sync` is ON and the doc has `_betterdocs_git_sync_enabled` meta set |
| Repo/Branch dropdowns empty after connect | Check browser console for AJAX errors. The proxy API relay may have timed out |
| Settings not saving (repo, branch, directory) | These save via AJAX on selection, not via "Save Settings". Check Network tab for `betterdocs_git_save_repo_setting` requests |
| Markdown conversion losing formatting | Complex HTML (tables, nested lists, custom blocks) may not convert perfectly. Review and manually adjust in Git if needed |
