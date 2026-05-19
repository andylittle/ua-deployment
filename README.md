# ua-deployment

GitLab CI/CD pipeline for deploying Assure1 content and supporting scripts to Assure1 servers.

## Overview

This repo manages two things:

1. **Assure1 content** — anything under a1svn control (the `default/` directory) that gets rsynced to the server and committed into Assure1's internal SVN (`a1svn`). SNMP trap rules are one example of content managed this way.
2. **Deployment scripts** — helper scripts (under `scripts/`) that are synced to the target server and invoked during the pipeline.

## Pipeline Stages

### `devDeploy`

Triggered on merge requests targeting the `development` branch.

1. Syncs `scripts/` to the dev server.
2. Copies trap rules to a temp directory (`/opt/tmp`) on the dev server.
3. Runs `check-trap-rules` against the temp directory to validate syntax.
4. Checks that the a1svn working copy is clean (no uncommitted changes) before overwriting files.
5. Rsyncs the `default/` directory (excluding the `json/` subdir) to the dev server checkout.
6. Adds any new files to a1svn and commits with a message containing the GitLab user, short SHA, and commit message.

### `prodDeploy`

Triggered on merge requests targeting `main` from the `development` branch.

Same steps as dev, minus the syntax check, against the production server.

## Scripts

### `check-trap-rules`

```
check-trap-rules <path>
```

Finds all `*rules` files under `<path>` (modified in the last 24 hours, excluding `_vendor/`) and runs each through `/opt/assure1/bin/CheckSyntax`. Exits non-zero on the first syntax error found.

### `check-trap-rules-verbose`

Same as `check-trap-rules` but prints every file and its result regardless of pass/fail. Hardcoded to the live checkout path — intended for manual debugging on the server, not pipeline use.

### `svn-commit-all`

Wraps a1svn operations against the Assure1 checkout directory (`/opt/assure1/var/checkouts/core`).

| Flag | Description |
|------|-------------|
| `-s` | Show a1svn status. Exits non-zero if the working copy is dirty. |
| `-a` | Add all untracked files (`svn add`). |
| `-c <message>` | Commit with a plaintext message. |
| `-e <encoded>` | Commit with a base64-encoded message (used by the pipeline to safely pass the GitLab commit message over SSH). |
| `-h` | Print help. |

## Variables

Configured in `.gitlab-ci.yml`:

| Variable | Description |
|----------|-------------|
| `DEV_SERVER` | Dev server IP |
| `PROD_SERVER` | Prod server IP |
| `SDIR` | Local source directory (`default/`) |
| `DDIR` | Remote destination path on the server |
| `TDIR` | Temp directory used for syntax checking on dev |
| `SCRIPT_DIR` | Remote path where scripts are deployed |
| `PACKG_DIR` | Assure1 custom packages directory (referenced but not currently used in pipeline steps) |
| `TOOLS_DIR` | Assure1 tools/event directory (referenced but not currently used in pipeline steps) |
| `RSYNC_ARGS` | Common rsync flags used across all sync steps |

## Best Practices

**Do not use Assure1's default rule directory names.** Because the pipeline uses rsync to push content to the server, deploying into a default directory (e.g. `default/collection/event/tcp-client/`) will overwrite Assure1's out-of-the-box rules.

To add a new rule set, copy the relevant default directory on the UA server to a custom-named one, then pull that renamed copy into this repo:

```
# Example: on the UA server
cp -r /opt/assure1/var/checkouts/core/default/collection/event/tcp-client \
       /opt/assure1/var/checkouts/core/default/collection/event/tcp-client-stp

# Register the new directory with a1svn and commit it
a1svn add /opt/assure1/var/checkouts/core/default/collection/event/tcp-client-stp
a1svn commit -m "[<username>] first commit of stp rules"
```

Then copy `tcp-client-stp/` into this repo under `default/collection/event/` and commit it. Only custom-named directories belong in this repo — never the original Assure1 default directories.

**Note:** Keeping git and a1svn in sync is a manual process. The pipeline handles git → a1svn on each deploy, but any changes made directly in a1svn won't automatically be reflected back in git.

**Trap rules are the only exception.** The `default/collection/event/trap/` directory is included in this repo because those rules are fully managed here and the defaults are intentionally replaced.

## Branch Flow

```
feature branch → development (devDeploy runs) → main (prodDeploy runs)
```
