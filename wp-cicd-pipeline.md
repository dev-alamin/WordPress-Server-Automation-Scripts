# WordPress Git-Based CI/CD — The Complete Cheat Sheet

A step-by-step, battle-tested guide for deploying **your own custom themes and plugins**
to staging/production WordPress servers via Git + GitHub Actions. This deliberately
ignores WordPress.org/marketplace plugin management — that's a separate concern
(Composer + WPackagist, covered briefly at the end).

Written from real production experience setting up CI/CD for a live WooCommerce store.

---

## 0. The Core Philosophy — Read This First

> **Git tracks your code. It does not manage your WordPress installation.**

This is the single idea that makes everything else make sense:

- WordPress core, third-party plugins, uploads, and the database are **infrastructure** — installed once, managed by other tools (WP-CLI, backups, manual updates), and **never touched by your deploy pipeline.**
- Your custom theme and custom plugin(s) are **code** — authored by you, versioned in Git, and the *only* thing your CI/CD pipeline overwrites on every deploy.
- The git repository lives **directly inside** the WordPress docroot on each server. There is no separate "build and copy" step — the working directory *is* the live site.

Once this clicks, every weird git error you'll hit ("untracked files would be overwritten," "dubious ownership," etc.) becomes easy to reason about: it's almost always git colliding with a file that already existed on disk before you started tracking things.

---

## 1. Deciding What to Track — The Whitelist Model

**Don't try to `.gitignore` every WordPress core file individually.** Instead: ignore
everything by default, then explicitly un-ignore only your own code. This is the
only approach that survives WordPress core updates, new plugins being installed, and
directory structure surprises without constant `.gitignore` maintenance.

### What to track
- `wp-content/themes/your-custom-theme/` (or child theme)
- `wp-content/plugins/your-custom-plugin/` (each custom plugin, one line per plugin)
- `wp-content/mu-plugins/` if you author must-use plugins
- A **sanitized config template** (`wp-config-sample.php` or `.env.example`) — never the real secrets file
- Deploy scripts, CI workflow files, root-level docs (`README.md`, `CHANGELOG.md`)

### What to never track
- WordPress core (`wp-admin/`, `wp-includes/`, root PHP files) — reinstalled/updated independently
- Any third-party/marketplace plugin (WooCommerce, Elementor, ACF, etc.) — see §9 for how to manage these properly
- `wp-content/uploads/` — media, way too large, not code
- `wp-content/cache/`, `wpo-cache/`, `litespeed/`, any cache plugin's working directory
- `wp-content/debug.log` — grows unbounded, pure noise in git history
- `wp-config.php` — contains DB credentials and secret keys, must never be committed
- Backup plugin working directories (UpdraftPlus, All-in-One Migration, etc.)

### The `.gitignore` pattern

```gitignore
# WordPress core — never tracked
/wp-admin/
/wp-includes/
/index.php
/wp-*.php
/xmlrpc.php
/license.txt
/readme.html
/wp-config.php
!wp-config-sample.php

# wp-content: ignore everything, then whitelist only our code
/wp-content/*
!/wp-content/themes/
!/wp-content/plugins/

# Themes: ignore all, un-ignore only ours
/wp-content/themes/*
!/wp-content/themes/your-custom-theme/

# Plugins: ignore all, un-ignore only ours (one line per custom plugin)
/wp-content/plugins/*
!/wp-content/plugins/your-custom-plugin/

# Never track these even if they end up inside a tracked folder
/wp-content/uploads/
/wp-content/cache/
/wp-content/debug.log
/wp-content/advanced-cache.php
/wp-content/object-cache.php

# Build artifacts / secrets / OS cruft
**/node_modules/
**/vendor/
.env
.env.*
!.env.example
.DS_Store
*.sql
*.sql.gz
```

**How `.gitignore` actually works (common misconception):**
- It only affects **untracked** files. If a file is already committed, adding it to `.gitignore` does nothing — you must `git rm --cached <file>` first to stop tracking it.
- It's evaluated top-to-bottom; `!` un-ignores a previously-ignored path, but only works if a parent directory wasn't itself excluded first (order matters — see the pattern above: ignore the parent's contents with `/*`, *then* un-ignore specific children).
- For genuinely **per-server** exclusions that shouldn't affect the shared repo (e.g., one box has a stray debug plugin folder), use `.git/info/exclude` instead — same syntax, but local-only, never committed, never synced.

---

## 2. Branching Strategy

For a typical solo-dev or small-team WP project, skip GitFlow — it's too much ceremony.
Use **trunk-based with environment branches**:

```
main       → always deployable, mirrors production exactly
staging    → merge target for testing before prod
feature/*  → short-lived branches for individual changes, never deployed directly
```

**Flow:**
```
feature/new-widget → PR → merge into staging → auto-deploys to staging
                                              → test on staging URL
staging → PR → merge into main → auto-deploys to production
```

**Tag every production release:**
```bash
git tag -a v1.4.0 -m "Add related collections widget"
git push origin v1.4.0
```
Loosely follow semver: patch = bugfix/no behavior change, minor = new feature, major = breaking change. This gives you instant rollback capability — `git checkout v1.3.0` and redeploy if a release breaks something.

**Trade-off to know:** trunk-based + environment branches is simple but assumes one active line of work at a time. If you ever have multiple developers shipping unrelated features in parallel that need independent testing, you'll want feature-branch preview environments — a level of complexity not covered here.

---

## 3. Server-Side Setup — One-Time, Per Server

Do this **once per server** (staging and production separately — never share credentials
between them).

### 3.1 Create a dedicated deploy user

Never use your personal account or root for CI/CD. A dedicated user with minimal
permissions limits blast radius if a credential ever leaks.

```bash
sudo adduser --system --shell /bin/bash --home /home/deploy --group deploy
sudo usermod -aG www-data deploy
```

### 3.2 Grant filesystem access via ACLs (not broad chmod)

```bash
sudo apt install acl -y   # if not already present
sudo setfacl -R -d -m u:deploy:rwx /var/www/your-site
sudo setfacl -R -m u:deploy:rwx /var/www/your-site
```
The `-d` flag sets a **default ACL** — meaning files `www-data` creates *after* this
point (uploads, generated CSS, cache files) automatically inherit deploy-user write
access too. Without `-d`, you'll hit permission errors again the next time a new file
appears.

**Verify:**
```bash
getfacl /var/www/your-site | grep deploy
```

### 3.3 Two separate SSH keys — don't conflate them

This trips up almost everyone the first time. There are **two distinct authentication
relationships**, each needing its own key:

| Key | Direction | Purpose | Where it's stored |
|---|---|---|---|
| **A: Actions → Server** | GitHub Actions runner → your VPS | Lets CI log in over SSH to run deploy commands | Private half → GitHub environment secret (`VPS_SSH_KEY`). Public half → server's `~deploy/.ssh/authorized_keys` |
| **B: Server → GitHub** | Your VPS → GitHub.com | Lets `git fetch`/`git pull` on the server authenticate to your **private** repo | Private half → stays on server only (`~deploy/.ssh/`). Public half → GitHub repo's Deploy Keys (read-only) |

**Generate Key A** (Actions logging into server):
```bash
sudo -u deploy ssh-keygen -t ed25519 -f /home/deploy/.ssh/id_ed25519_actions -N ""
sudo -u deploy sh -c 'cat /home/deploy/.ssh/id_ed25519_actions.pub >> /home/deploy/.ssh/authorized_keys'
sudo -u deploy chmod 700 /home/deploy/.ssh
sudo -u deploy chmod 600 /home/deploy/.ssh/authorized_keys
sudo cat /home/deploy/.ssh/id_ed25519_actions   # → paste into VPS_SSH_KEY secret
```

**Generate Key B** (server pulling from GitHub):
```bash
sudo -u deploy ssh-keygen -t ed25519 -f /home/deploy/.ssh/hv_repo_deploy_key -N "" -C "deploy-$(hostname)"
sudo cat /home/deploy/.ssh/hv_repo_deploy_key.pub   # → add as a read-only Deploy Key on the GitHub repo
```

Point git at Key B via an SSH config alias (cleaner than fighting with `GIT_SSH_COMMAND`):
```bash
sudo -u deploy tee -a /home/deploy/.ssh/config <<'EOF'
Host github-yoursite
  HostName github.com
  User git
  IdentityFile /home/deploy/.ssh/hv_repo_deploy_key
  IdentitiesOnly yes
EOF
sudo -u deploy chmod 600 /home/deploy/.ssh/config
```

**Pitfall:** Deploy Keys in GitHub are scoped to a single repo — this is a feature, not
a limitation. Don't reuse one deploy key across multiple repos even if convenient; if
it leaks, you want the blast radius contained to one project. Always set **read-only**
unless the server genuinely needs to push (it almost never should).

### 3.4 Scoped, minimal sudo — only what deploy actually needs

```bash
sudo visudo -f /etc/sudoers.d/deploy-fpm
```
```
deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl reload php8.5-fpm
```
**Never** grant broad `NOPASSWD: ALL` — the entire point of a dedicated deploy user is
that a leaked CI credential can, at worst, push code and reload PHP-FPM. Nothing else.

**Pitfall:** PHP-FPM service names vary by version/distro (`php8.4-fpm`, `php8.5-fpm`,
etc.) and must match *exactly* between the sudoers rule and what you call in the deploy
script, or you'll get `Failed to reload ... Unit not found` — verify first:
```bash
systemctl list-units --type=service | grep php
```
If staging and prod run different PHP versions, either hardcode per-environment in the
workflow, or use a dynamic lookup + a wildcard sudoers rule (`php*-fpm`) — check your
sudo version supports command globbing before relying on it.

### 3.5 Git ownership handshake

```bash
sudo -u deploy git config --global --add safe.directory /var/www/your-site
```
**Pitfall:** `safe.directory` is per-user, stored in that user's `~/.gitconfig`. If you
run `git status` as root or as your personal account afterward, you'll hit the same
"dubious ownership" error again — it wasn't fixed globally, only for `deploy`. Always
prefix git commands on the server with `sudo -u deploy` (or better: never run them
manually at all once CI/CD is live — see §7).

---

## 4. Bootstrapping the Repo on an Already-Live Server

This is the messiest part in practice, because the server's directory almost always
already has files that collide with what you're about to check out.

```bash
cd /var/www/your-site
sudo -u deploy git init
sudo -u deploy git remote add origin git@github-yoursite:you/your-repo.git
sudo -u deploy git fetch origin
```

**Before checking out — check for collisions:**
```bash
ls wp-content/themes/ | grep your-theme-name
ls wp-content/plugins/ | grep your-plugin-name
```

If they already exist (near-certain on a live server that had manual deploys before),
**back up before removing** — never blind-delete:
```bash
sudo -u deploy mkdir -p /tmp/pre-git-backup
sudo -u deploy cp -a wp-content/themes/your-theme-name /tmp/pre-git-backup/
sudo -u deploy cp -a wp-content/plugins/your-plugin-name /tmp/pre-git-backup/
sudo -u deploy rm -rf wp-content/themes/your-theme-name
sudo -u deploy rm -rf wp-content/plugins/your-plugin-name
```

```bash
sudo -u deploy git checkout main   # or staging
```

**After checkout, always diff against the backup** — this tells you whether the live
server had drifted from what's in git (someone tested a change directly on the server
that never got committed):
```bash
diff -r /tmp/pre-git-backup/your-theme-name wp-content/themes/your-theme-name
```
- No output → server was already in sync, safe to discard the backup.
- Output → real divergence. Don't discard yet — decide whether those changes need to
  be ported into a commit, or were abandoned experiments safe to drop.

**Verify the tracking is clean:**
```bash
sudo -u deploy git status
sudo -u deploy git ls-files | grep -E "wp-admin|wp-includes|uploads|debug.log"
```
The second command must return **nothing**. If it returns results, your `.gitignore`
isn't working as intended — stop and fix it before pushing anything.

**Common gotcha:** `wp-config-sample.php` ships with WordPress core by default, so a
live server already has one, generic and untracked. It'll collide with your repo's
own version the same way the theme/plugin folders did — back it up and remove it the
same way before checkout.

---

## 5. GitHub-Side Setup

### 5.1 Use Environments, not flat repo secrets

Repo Settings → **Environments** → create `staging` and `production` separately. Each
gets its **own isolated secret set**, same key names, different values:

| Secret | Staging value | Production value |
|---|---|---|
| `VPS_HOST` | staging server IP/host | prod server IP/host |
| `VPS_USER` | `deploy` | `deploy` |
| `VPS_SSH_KEY` | staging's Key A (private) | prod's Key A (private) |
| `DEPLOY_PATH` | staging docroot path | prod docroot path |

**Why Environments over plain "Secrets and variables → Actions":** environment-scoped
secrets are only visible to a job that explicitly declares `environment: staging` (or
`production`). This makes cross-contamination between servers structurally impossible,
not just a matter of careful naming.

**Bonus:** Environments also support **required reviewers** — meaning a push to `main`
pauses for manual approval before actually deploying to production. Strongly recommend
enabling this for anything revenue-generating; it's the only manual gate in an otherwise
fully automated pipeline, and it's cheap insurance.

### 5.2 The workflow file

```yaml
name: Deploy

on:
  push:
    branches:
      - main
      - staging

jobs:
  deploy-staging:
    if: github.ref_name == 'staging'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            cd ${{ secrets.DEPLOY_PATH }}
            git fetch origin staging
            git reset --hard origin/staging
            wp cache flush --path=. || true
            command -v redis-cli >/dev/null && redis-cli FLUSHALL || true
            sudo systemctl reload php8.5-fpm

  deploy-production:
    if: github.ref_name == 'main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            cd ${{ secrets.DEPLOY_PATH }}
            git fetch origin main
            git reset --hard origin/main
            wp cache flush --path=. || true
            command -v redis-cli >/dev/null && redis-cli FLUSHALL || true
            sudo systemctl reload php8.5-fpm
```

**Critical pitfall:** the workflow file must exist **on the branch being pushed**, not
just wherever you last edited it. GitHub reads `.github/workflows/*.yml` from the
commit being pushed — if you only committed it to `main`, pushing to `staging` triggers
nothing, silently. Check both:
```bash
git show staging:.github/workflows/deploy.yml
git show main:.github/workflows/deploy.yml
```

**Why `git reset --hard` and not `git pull`:** you want the server's tree to
*deterministically* match the commit — no merge conflicts from a stray manual edit at
2am. Trade-off: any hand-edit made directly on the server gets silently discarded on
next deploy. Treat that as a feature — it enforces discipline — but know it's happening.

**Why `git reset --hard` never touches uploads/core/other-plugins:** it only resets
files git *tracks*. Anything covered by `.gitignore` is invisible to git entirely,
untouched regardless of how aggressive the reset is.

---

## 6. Composer for Third-Party Plugins (Optional, Recommended at Scale)

Since third-party plugins are deliberately excluded from git (§1), you still need *some*
way to pin and reproduce exact versions across servers, rather than manually tracking
"WooCommerce 10.9.4 on prod, hope staging matches."

```json
{
  "require": {
    "composer/installers": "^2.0",
    "wpackagist-plugin/woocommerce": "10.9.4",
    "wpackagist-plugin/advanced-custom-fields": "6.8.6"
  },
  "extra": {
    "installer-paths": {
      "wp-content/plugins/{$name}/": ["type:wordpress-plugin"]
    }
  }
}
```
`composer install` on the server rebuilds the entire third-party plugin directory from
`composer.lock` — reproducible, version-pinned, and still doesn't bloat your git history
with vendor code.

**Limitation:** premium plugins without a public feed (Elementor Pro, most paid
plugins) don't have a WPackagist entry. Options: use the plugin's own update-server URL
in `composer.json` if it supports one (check the vendor's docs), or accept the gap and
maintain a simple `plugins-manual.md` documenting exact versions + install source,
installed by hand and covered by your regular backup tooling instead.

**Pragmatic middle ground for small teams:** Composer-manage what's on WPackagist,
document-and-manually-install the rest. Don't over-engineer this on day one.

---

## 7. The Post-Setup Mental Model — What Changes Day-to-Day

Once the pipeline above is live, **stop treating the server as a place you `git`
into.** The workflow becomes:

```
1. Work locally, on a feature branch
2. Merge into staging → git push origin staging → auto-deploys, test on staging URL
3. Merge staging into main → git push origin main → (optional manual approval) → auto-deploys to prod
4. Tag the release: git tag v1.x.x && git push origin v1.x.x
```

You never SSH in and run `git checkout`/`git reset`/`git pull` by hand again. The only
legitimate reasons to touch git *on* the server afterward:
- **Read-only debugging**: `git log`, `git status`, `git show` to confirm what's
  actually deployed if something looks wrong in production.
- **Emergency rollback**: re-push an older tag from local (`git push origin v1.3.0:main
  --force` handled carefully, or better, `git revert` + normal push) rather than
  hand-resetting on the server — keeps the deploy history honest and auditable.

If you ever feel the urge to manually `git pull` on a server with CI/CD live, that's a
signal something's wrong with the pipeline (fix the workflow) — not a shortcut to reach
for.

---

## 8. Database Strategy — The One Thing NOT to Automate Together With Code

For a live WooCommerce (or any real-data) site: **never wire DB sync into the same
automated trigger as code deploy.** Mixing "deploy code" and "overwrite database" into
one push-triggered pipeline is how stores lose real orders.

Recommended: keep DB sync **manual and one-directional (prod → staging only)**,
triggered separately from code deploys:
```bash
wp db export - --path=/path/to/prod | gzip > backup.sql.gz
# transfer to staging, then:
gunzip < backup.sql.gz | wp db import - --path=/path/to/staging
wp search-replace 'https://production-domain.com' 'https://staging-domain.com' --path=/path/to/staging --all-tables
```
If you want this in GitHub Actions at all, use `workflow_dispatch` (manual trigger only,
never `on: push`) so a DB sync is always a deliberate action, never an accidental
side-effect of a code push.

---

## 9. Full Pitfall Reference (Quick Lookup)

| Symptom | Cause | Fix |
|---|---|---|
| `fatal: detected dubious ownership` | Running git as a different user than the one with `safe.directory` set | `git config --global --add safe.directory <path>` **as that specific user** |
| `untracked working tree files would be overwritten` | Files already exist on disk before first checkout (theme, plugin, `wp-config-sample.php`, etc.) | Back up → remove → checkout → diff to confirm nothing lost |
| `Permission denied` during `git pull`/checkout | Deploy user lacks write access to files owned by `www-data` | ACLs with `-d` (default) flag, not just one-time `chmod` |
| Rebase fails with permission errors mid-way | Same as above, but now stuck in an interrupted rebase state | `git rebase --abort` (may itself fail — clear blocking untracked files first, from backup, then retry abort) |
| `git fetch` fails with auth error on server | Missing/wrong SSH deploy key for GitHub (Key B, not Key A) | Confirm the *public* key added to GitHub Deploy Keys matches the *private* key configured in that server's SSH config alias |
| CI job fails fast (~15s) at SSH step | Wrong `VPS_USER`/`VPS_SSH_KEY` in GitHub Environment secrets, or Key A not in `authorized_keys` | Re-verify Key A (Actions → Server) is correctly paired, test manually: `ssh -i keyfile deploy@host` |
| `Failed to reload phpX.X-fpm: Unit not found` | Sudoers rule / workflow references a PHP-FPM service name that doesn't match the actual systemd unit | `systemctl list-units --type=service \| grep php`, fix the exact name in both sudoers and workflow |
| Workflow doesn't trigger on push to a branch | `.github/workflows/*.yml` doesn't exist on that specific branch | `git show <branch>:.github/workflows/deploy.yml` to confirm; merge the workflow file into both branches |
| Repo bloats to hundreds of MB within weeks | Third-party plugins (WooCommerce, Elementor, etc.) got committed | `.gitignore` whitelist model from §1; if already committed, `git rm -r --cached <path>` then commit the removal |
| Staging shows stale/older code after first checkout | Server had manually-deployed code from before git existed, git correctly overwrote it with the current repo state | Expected behavior, not a bug — confirm via diff against backup that the *newer* code is the git version |

---

## 10. Minimal Setup Checklist (Copy This)

```
[ ] .gitignore written using whitelist model (§1)
[ ] Local repo initialized, custom theme + plugin committed, pushed to GitHub
[ ] Branch strategy decided: main / staging / feature-*
[ ] Dedicated `deploy` user created on EACH server
[ ] ACLs granted to `deploy` user on webroot (with -d default flag)
[ ] Key A generated per server (Actions → Server), public half in authorized_keys
[ ] Key B generated per server (Server → GitHub), public half added as repo Deploy Key (read-only)
[ ] SSH config alias set up per server pointing at Key B
[ ] Sudoers rule added per server, scoped to exactly the reload command needed
[ ] safe.directory set for the `deploy` user specifically
[ ] Repo bootstrapped on each server: backup existing files → remove → checkout → diff-verify
[ ] GitHub Environments created: staging + production, secrets set per-environment
[ ] Required reviewers enabled on production environment (recommended)
[ ] Workflow file committed to BOTH main and staging branches
[ ] Test push to staging, confirm Actions run succeeds end-to-end
[ ] Test push to main, confirm (with approval gate if enabled)
[ ] Tag first production release
[ ] DB sync strategy decided — kept manual/separate from code deploy pipeline
[ ] (Optional) Composer + WPackagist wired up for reproducible third-party plugin versions
```

---

*Built from real-world setup of a live WooCommerce production + staging pipeline.
Adjust paths, PHP versions, and service names to match your actual stack — the
structure and reasoning transfer directly regardless of specifics.*
