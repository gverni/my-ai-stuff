---
name: wp-local-setup
description: Set up a local development copy of an existing (live-hosted) WordPress site — install prerequisites, pull the site files and database from the host, wire up local URLs, and create start/stop scripts. Use when the user wants to run a WordPress site locally for dev/testing, migrate a site from a host (SiteGround, Bluehost, etc.) to local, or debug a "works on live but not locally" WordPress issue.
---

# WordPress local dev environment setup

Turns a copy of a live-hosted WordPress site into a working local dev
environment: correct PHP/MySQL versions, imported database, rewritten URLs,
and simple start/stop scripts. Written from a real SiteGround migration; the
same steps apply to any host that gives FTP/SFTP + phpMyAdmin (or an
equivalent DB export) access.

Work through the steps below in order. Stop and ask the user before doing
anything destructive (overwriting an existing local DB, force-pushing,
deleting files) — see the "Safety notes" section throughout.

## 0. Clarify with the user first

Before starting, confirm:

1. **Where do the site files live right now?** A folder already downloaded
   locally, or does it need to be fetched from the host? If not fetched yet,
   go to step 2.
2. **What's the local PHP version target?** Ideally: whatever the live
   server runs (see step 3 for how to find this from the DB dump). Mismatches
   are the single biggest source of "it crashes locally but not on live"
   bugs — see Troubleshooting.
3. **Which port for the local server?** Default to `8888` unless something
   else is already using it or the user has a preference.
4. **Git repo needed?** If yes, see step 8 for what to exclude.

Don't ask more than needed — reasonable defaults (port 8888, PHP version
matched to the DB dump, git repo yes if working in a repo already) are fine
to just state and proceed with.

## 1. Install prerequisites (macOS + Homebrew)

Check what's already installed before installing:

```bash
brew list --versions php mysql wp-cli 2>&1
```

Install whatever's missing:

```bash
brew install mysql wp-cli
# PHP version comes later, once we know what the live server runs (step 3).
# If a version is already known, e.g. 8.2:
brew install php@8.2
```

`php@8.x` formulas are keg-only (not symlinked into PATH) — always invoke
via full path, e.g. `/usr/local/opt/php@8.2/bin/php` (Intel Mac) or
`/opt/homebrew/opt/php@8.2/bin/php` (Apple Silicon). Confirm the actual path
with `brew --prefix php@8.2`.

Start MySQL as a background service:

```bash
brew services start mysql
```

## 2. Get the site files and database from the host

**Site files**: download the *entire* web root (e.g. `public_html/`), not
just `wp-content`/plugins/themes. Sites are sometimes set up with WordPress
core in its own subdirectory and custom paths for uploads/plugins (a
`wp-config.php` with `WP_CONTENT_DIR`/`WP_CONTENT_URL` pointing somewhere
other than `wp-content/`) — grabbing only part of the tree breaks this
silently later. Use whichever the host offers:

- FTP/SFTP client (FileZilla, Cyberduck) — select the whole web root folder
- Hosting panel's File Manager → "Compress" the whole folder, then download
  the archive
- If SSH is available, `rsync`/`scp` is far faster than FTP for large sites
  (see "Sync workflow" below) — check the host's control panel for SSH
  access before defaulting to FTP.

**Database**: needs to be exported separately; downloading files does not
include it.

- phpMyAdmin (usually under the host's "Site Tools" / control panel) →
  select the database → Export → SQL format, uncompressed
- Or use the host's own backup feature if it produces a raw `.sql` (some
  produce a proprietary archive instead — phpMyAdmin export is more
  portable)

**Safety note**: this is a copy of someone's production data. If it's a
DB with any real user/member/customer data, flag that to the user and keep
it out of version control (see step 8) regardless of what they explicitly
asked to exclude — don't wait to be told.

## 3. Determine the target PHP version

Open the `.sql` dump and check its header comment — most exports include
the exporting environment's info, and WordPress's own serialized option
data or plugin metadata sometimes reveals the PHP version indirectly. More
reliably: check the host's control panel (PHP version selector) or ask the
user.

Match the local PHP version to this exactly if possible. A newer PHP
running older/poorly-maintained plugins is the most common source of fatal
errors that don't exist on the live site — see Troubleshooting.

```bash
brew install php@<matched-version>
```

## 4. Create the local database and import

Read the DB credentials already in the site's `wp-config.php`:

```bash
grep -E "DB_NAME|DB_USER|DB_PASSWORD|DB_HOST" path/to/wp-config.php
```

**Recommended**: create a local MySQL user/database with the *same*
name/user/password as production. This means `wp-config.php` needs zero
credential edits (one less thing to accidentally commit or leak), and one
less thing to redo every time the site files get re-synced from the host
(see Troubleshooting — this file tends to get overwritten on re-sync).

```bash
mysql -u root -e "
CREATE DATABASE IF NOT EXISTS \`<db_name>\`;
CREATE USER IF NOT EXISTS '<db_user>'@'localhost' IDENTIFIED BY '<db_password>';
GRANT ALL PRIVILEGES ON \`<db_name>\`.* TO '<db_user>'@'localhost';
FLUSH PRIVILEGES;
"

mysql -u root <db_name> < path/to/dump.sql
```

If MySQL isn't accepting connections yet right after `brew services start
mysql`, poll briefly rather than failing immediately — it can take a few
seconds to come up (this is baked into `start.sh` below).

## 5. Point the site at localhost

Two separate things need to change — missing either one causes bugs:

**A. `wp-config.php` constants** — these are PHP constants and always win
over whatever's in the database, so they must be edited directly:

```php
define('WP_HOME','http://localhost:8888');
define('WP_SITEURL','http://localhost:8888/wp');   // adjust path if WP core isn't at web root

// Only if the site uses a custom content directory (check for this
// constant already present, pointing elsewhere, e.g. a renamed
// "content/" folder instead of "wp-content/"):
define('WP_CONTENT_URL', 'http://localhost:8888/content');
```

**Do not skip this even if the DB's `siteurl`/`home` options already look
local** — WordPress reads the constants first, and without them requests
like `wp-login.php` will silently redirect back to the *live* production
site. This is a real risk: it can autofill a saved production password into
a form pointed at the live site. Warn the user if you see this happen.

**B. Rewrite the domain everywhere else in the database.** The constants
above only cover `wp_options.siteurl`/`home`. Every other place the live
domain is baked into content — image URLs in post bodies, ACF/serialized
field data, widget settings — needs a real find-and-replace, which must be
serialization-aware (a plain SQL `REPLACE()` corrupts PHP-serialized data
whenever the replacement string is a different length). Use WP-CLI:

```bash
WP_CLI_PHP=/path/to/php@X.Y/bin/php wp search-replace \
  'https://live-domain.example.com' 'http://localhost:8888' \
  --all-tables --allow-root \
  --path=/path/to/site/wp
```

(`--allow-root` is only needed if running as root; drop it otherwise.)

## 6. Handle host-specific plugins

Hosting providers often bundle plugins that only function on their
infrastructure (server-level caching/security integrations, e.g.
SiteGround's `sg-security`/`sg-cachepress`, WP Engine's equivalents). These
typically don't crash anything locally, but they're dead weight and can
occasionally throw notices. Deactivate them:

```bash
WP_CLI_PHP=/path/to/php@X.Y/bin/php wp plugin deactivate <plugin-slug> \
  --allow-root --path=/path/to/site/wp
```

## 7. Create start/stop scripts

Use PHP's built-in dev server rather than installing a full Apache/nginx
stack — it's enough for functional dev/testing.

**Important**: disable OPcache on the dev server. A long-running PHP
built-in server process caches compiled bytecode; if the user (or Claude)
edits a plugin/theme file afterward, stale cached bytecode can get served,
producing confusing, inconsistent, request-to-request-varying fatal errors
that look like flaky bugs but are actually just a cache problem. Always
pass `-d opcache.enable=0`.

`start.sh`:

```bash
#!/bin/bash
# Starts the local WordPress dev environment: MySQL (via Homebrew) + PHP's
# built-in web server serving the site's web root.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DOCROOT="$SCRIPT_DIR/<web-root-folder>"      # e.g. public_html
PID_FILE="$SCRIPT_DIR/.wp-server.pid"
LOG_FILE="$SCRIPT_DIR/.wp-server.log"
PORT="${1:-8888}"
PHP_BIN="/path/to/php@X.Y/bin/php"           # from `brew --prefix php@X.Y`

if [ ! -x "$PHP_BIN" ]; then
  echo "PHP not found at $PHP_BIN. Install it with: brew install php@X.Y" >&2
  exit 1
fi

echo "Starting MySQL..."
brew services start mysql >/dev/null

echo -n "Waiting for MySQL to accept connections"
for i in $(seq 1 15); do
  if mysql -u root -e "SELECT 1;" >/dev/null 2>&1; then
    echo " done."
    break
  fi
  echo -n "."
  sleep 1
  if [ "$i" -eq 15 ]; then
    echo
    echo "MySQL did not become ready in time." >&2
    exit 1
  fi
done

if [ -f "$PID_FILE" ] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
  echo "PHP server already running (PID $(cat "$PID_FILE")) at http://localhost:$PORT/"
  exit 0
fi

echo "Starting PHP server on port $PORT..."
cd "$DOCROOT"
# opcache disabled: long-running single process, so edited files must not
# be served from stale cached bytecode.
nohup "$PHP_BIN" -d opcache.enable=0 -S "localhost:$PORT" > "$LOG_FILE" 2>&1 &
echo $! > "$PID_FILE"
disown

sleep 1
if kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
  echo "Site is up: http://localhost:$PORT/"
  echo "wp-admin:  http://localhost:$PORT/wp/wp-login.php"   # adjust path if WP core is at web root
else
  echo "PHP server failed to start, check $LOG_FILE" >&2
  rm -f "$PID_FILE"
  exit 1
fi
```

`stop.sh`:

```bash
#!/bin/bash
# Stops the local PHP dev server. MySQL is left running (lightweight
# Homebrew service) - pass --with-mysql to stop it too.
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PID_FILE="$SCRIPT_DIR/.wp-server.pid"

if [ -f "$PID_FILE" ] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
  kill "$(cat "$PID_FILE")"
  rm -f "$PID_FILE"
  echo "PHP server stopped."
else
  PID="$(lsof -ti:8888 2>/dev/null || true)"   # match the default port used above
  if [ -n "$PID" ]; then
    kill "$PID"
    echo "PHP server stopped (found via port 8888)."
  else
    echo "PHP server was not running."
  fi
  rm -f "$PID_FILE"
fi

if [ "${1:-}" = "--with-mysql" ]; then
  brew services stop mysql
  echo "MySQL stopped."
fi
```

Make both executable: `chmod +x start.sh stop.sh`.

## 8. Git repo (if requested)

If the user wants this in a git repo, exclude:

- The uploads/plugins/themes folder (usually the heaviest part, several GB,
  mostly third-party/binary — not worth version-controlling)
- `wp-config.php` — has DB password, WordPress secret keys, and often
  additional API/auth keys from installed plugins
- Any `.sql` dump — likely contains real user/member/customer data
- Runtime PHP error logs, server PID/log files from the start/stop scripts
- Check for and remove any stray broken git submodule artifacts
  (`gitdir:` gitlink files, `.gitmodules`) left over from the original
  host's deployment tooling — these break `git add` if not cleaned up
  first.

```gitignore
# Themes, plugins, uploads — heaviest part, mostly third-party/binary.
<web-root>/<content-folder>/

# Live DB password + secret keys — local-only.
<web-root>/wp-config.php

# Real user data — local-only.
/*.sql

# Runtime logs / dev server artifacts.
**/php_errorlog
.wp-server.pid
.wp-server.log
```

## 9. Verify

Start the server and check the homepage and `wp-login.php` both load
without errors:

```bash
./start.sh
curl -sI http://localhost:8888/
curl -sI http://localhost:8888/wp/wp-login.php   # adjust path as needed
```

A `302` on `wp-login.php` redirecting to the *live* domain (check the
`Location:` header) means step 5A didn't take — go back and check
`wp-config.php`. If the browser is used to verify, be careful: this
particular failure mode auto-navigates to the real production site and can
autofill a saved password there. Stop and flag it to the user rather than
proceeding if seen.

Tail `.wp-server.log` for any `PHP Fatal error` — see Troubleshooting below
for how to read these.

## Sync workflow (keeping local in sync with live later)

If the user will periodically re-sync files from the host (not just a
one-time migration):

- **`wp-config.php` will very likely get overwritten** back to production
  values on any full re-sync/re-download. This isn't a bug in the sync —
  it's a real file living in the web root, and it comes back with the rest.
  Redo step 5A after every sync. Recommend excluding this file from
  whatever sync tool is used (FTP client filter, `rsync --exclude`) so it
  never needs redoing.
- If the user is on FTP and it feels slow/repetitive, and their host
  supports SSH, suggest switching to `rsync` over SSH: only transfers
  changed files, resumable, and supports a permanent exclude list instead
  of a manual per-transfer filter.
- Note any local-only code patches made during troubleshooting (see below)
  — these will also need reapplying after a sync that touches the same
  file. Keep a short list (e.g. a `TODO.md` in the repo) of what was
  patched and why, so it's not rediscovered the hard way each time.

## Troubleshooting

**Fatal error changes between requests / doesn't reproduce consistently.**
Almost always OPcache serving stale bytecode from before a file was edited.
Confirm `start.sh` passes `-d opcache.enable=0`; if the server was started
before this was added, restart it.

**A plugin throws a fatal error only locally, not on live.** First suspect
is PHP version mismatch — recheck step 3 and consider installing the exact
version the live server runs, retry with
`WP_CLI_PHP=/path/to/matched-php wp ...`. If the fatal reproduces
identically on the matched version too, it's very likely a genuine plugin
bug — real-world examples seen: a duplicate `static $var;` declaration in
the same PHP method (hard fatal in any PHP version), or a class file
referenced by a `new ClassName()` call that never actually exists in the
downloaded copy (a genuinely missing/corrupted file, common after
FTP/partial-sync migrations — check the plugin's own repo/CDN copy to see
if the class is expected to exist, and either source the missing file from
a matching plugin re-download, or patch narrowly). Document any such patch
clearly as local-only — do not upstream it — and log it in the project's
TODO/README so it survives being asked "why does this file look edited?"
later.

**"Undefined array key" or "trying to access array offset on bool"
warnings from a custom theme's ACF/Gutenberg block templates, especially
while using the block editor (which can look like the editor "freezing" or
UI crashing).** Usually a block template assumes an ACF image/relationship
field is always populated (`$image['url']` without checking `$image` is
non-empty first) and blows up in the editor's live-preview AJAX calls when
a block instance has that field empty. Fix defensively at the point of
use: `$url = $image ? $image['url'] : '';` — do not assume the field is
always filled just because it usually is.

**`wp-login.php` (or any admin URL) redirects to the live production
domain.** `wp-config.php`'s `WP_HOME`/`WP_SITEURL` constants still contain
the live domain — see step 5A. This is more than a cosmetic bug: it can
land the browser on the real live site with a saved password autofilled
into a real login form. Treat this as urgent to fix, not just untidy.

**MySQL connection refused right after starting the service.** Race
condition — MySQL takes a few seconds to accept connections after
`brew services start mysql`. Poll (`mysql -u root -e "SELECT 1;"` in a
retry loop) rather than failing on the first attempt; this is already
built into the `start.sh` template above.
