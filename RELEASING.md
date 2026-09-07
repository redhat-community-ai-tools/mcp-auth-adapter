# Releasing

## How release notes are generated

GitHub automatically generates release notes from **merged PRs** between the
previous tag and the new tag. For each PR, the **title** and **author** are
shown (the PR body is not included). PRs are grouped into sections based on
their GitHub **labels** (configured in `.github/release.yml`).

This means:
- All changes should go through PRs with **clear, descriptive titles**.
- Apply labels (`breaking`, `enhancement`, `bug`, `documentation`, etc.) to PRs
  so they are grouped correctly. Labels can be added **at any time before the
  release is created** -- not necessarily at merge time.
- Direct commits to `main` (not via PR) appear as raw commit hashes -- avoid
  these for user-visible changes.

After the release is created, you can **edit the notes in the GitHub UI** to
add context, highlight important changes, or remove noise.

## Prerequisites

- Permission to open PRs to `main` and **push `v*` tags** (direct pushes to
  `main` are blocked by branch protection in this org)
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
  (`gh auth login`) -- used by `npm run changelog` to fetch release notes
- **npm Trusted Publisher** configured on npmjs.com -- see "Trusted Publisher
  setup" below. Publish uses OIDC only; no `NPM_TOKEN` / `NODE_AUTH_TOKEN`.
- (Bootstrap only) For the very first publish of a brand-new package name
  (before a trusted publisher can be attached), see "Initial npm token setup
  (bootstrap)" below.

## Steps

`main` requires changes through a PR. The **version bump** merges via PR; the
**release** is triggered by pushing a tag (not by pushing to `main`).

1. Ensure `main` is green (CI passing)
2. Review merged PRs since the last tag -- add/fix labels if needed
3. Bump the version on a branch and open a PR:

   ```bash
   git checkout main && git pull
   git checkout -b release/X.Y.Z   # or chore/bump-X.Y.Z

   npm version patch --no-git-tag-version   # or minor|major

   git add package.json package-lock.json
   git commit -m "chore: bump version to X.Y.Z"
   git push -u origin HEAD
   ```

   Open a PR, get CI green, and **merge** to `main`.

   Use `--no-git-tag-version` so `npm version` only edits `package.json` and
   `package-lock.json`. Do **not** run `npm version patch` without that flag on
   `main`; it creates a commit and tag locally and `git push origin main` will
   be rejected.

4. Tag the merged commit on `main` and push **only the tag**:

   ```bash
   git checkout main && git pull
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

   The tag must point at a commit where `package.json` already shows `X.Y.Z`.
   `release.yml` checks out that commit; `npm publish` uses the version from
   `package.json` (not the tag name). Keep the tag name and `package.json`
   version in sync (`v1.2.3` / `1.2.3`).

5. The `release.yml` workflow will automatically:
   - Run lint, test, build
   - Publish to npm with provenance via OIDC Trusted Publishing
   - Create a GitHub Release with auto-generated notes
   - Build and push a container image to `ghcr.io/redhat-community-ai-tools/mcp-auth-adapter`
     with tags `X.Y.Z`, `X.Y`, `X`, and `latest`
6. (Optional) Edit the GitHub Release notes in the UI to curate
7. Run `npm run changelog` to regenerate `CHANGELOG.md` from all GitHub
   Releases, then open a PR with the result and merge to `main`. This can also
   be re-run later if you edit release notes after the fact.

### Branch protection pitfalls

- **`git push origin main --follow-tags` fails** -- expected; use the PR +
  tag-only push flow above.
- **Tag exists but `main` still shows the old version** -- the version-bump PR
  did not merge before the tag was pushed. Open a PR to sync `package.json` on
  `main` (no new tag; the release already shipped from the tag commit).
- **Squash-merge creates a different commit than the tag** -- normal. The tag
  may point at an older SHA while `main` has an equivalent squash commit; npm
  and GHCR are built from the **tag**, not from the tip of `main`.

## Version guidance

- `patch` -- bug fixes, docs, internal refactors
- `minor` -- new features, non-breaking behavior changes
- `major` -- breaking changes (config format, removed features, API changes)

## Trusted Publisher setup

npm [Trusted Publishing](https://docs.npmjs.com/trusted-publishers/) uses
OpenID Connect (OIDC) so the GitHub Actions workflow can publish to npm
**without any long-lived npm token**. The npm registry verifies the
cryptographic identity of the workflow run instead of a stored secret.

The trusted publisher is bound to an **exact GitHub location** -- the specific
user/org, repository, and workflow filename must all match. This means only
the `release.yml` workflow in `redhat-community-ai-tools/mcp-auth-adapter` can
publish the package; a fork or a different workflow file cannot.

### Configure on npmjs.com

1. Go to [npmjs.com/package/mcp-auth-adapter/access](https://www.npmjs.com/package/mcp-auth-adapter/access)
   (or: package page > **Settings** > **Publishing access**)
2. In the **Trusted Publisher** section, click **Add trusted publisher** and
   select **GitHub Actions**
3. Fill in:
   - **Organization or user**: `redhat-community-ai-tools`
   - **Repository**: `mcp-auth-adapter`
   - **Workflow filename**: `release.yml` (filename only, not the full path;
     must include the `.yml` extension)
   - **Environment name**: leave empty (unless you add a GitHub Environment
     for deployment protection later)
   - **Allowed actions**: select `npm publish`
4. Click **Save changes**

All fields are **case-sensitive** and must exactly match the GitHub repository
and workflow file.

### Workflow requirements

The release workflow (`.github/workflows/release.yml`) is set up for OIDC-only
publish:

- `id-token: write` permission is set on the publish job (required for OIDC
  token generation)
- `actions/setup-node` is configured with `registry-url: https://registry.npmjs.org`
- `npm install -g npm@latest` runs before publish (trusted publishing needs
  npm CLI >= 11.5.1; Node 22's bundled npm is older)
- Before `npm publish`, the workflow strips `_authToken` from the `.npmrc`
  written by `setup-node`. An empty `${NODE_AUTH_TOKEN}` line makes npm skip
  OIDC and fail with `ENEEDAUTH`
- **Do not set `NODE_AUTH_TOKEN`** on the publish step — even an empty value
  (or a classic automation token) blocks Trusted Publishing / conflicts with
  package 2FA rules
- `npm publish --provenance --access public` is used
- A **GitHub-hosted runner** (`ubuntu-latest`) is used -- self-hosted runners
  are not supported for trusted publishing

Once the trusted publisher is configured on npmjs.com, the workflow
authenticates via OIDC automatically. No GitHub `NPM_TOKEN` secret is needed.

If you still have a leftover `NPM_TOKEN` secret or an old npm access token from
earlier token-based publishes, delete them:

1. Remove `NPM_TOKEN` from
   [GitHub repo secrets](https://github.com/redhat-community-ai-tools/mcp-auth-adapter/settings/secrets/actions)
2. Delete unused tokens from
   [npmjs.com/settings/tokens](https://www.npmjs.com/settings/tokens)

## Initial npm token setup (bootstrap)

An npm token is only needed for the **very first publish** of a new package
(before the package exists on npmjs.com and a trusted publisher can be
configured), or as a **temporary fallback** while setting up trusted
publishing.

1. Go to [npmjs.com/settings/tokens](https://www.npmjs.com/settings/tokens)
2. Click **Generate New Token** > **Granular Access Token**
3. Configure:
   - **Token name**: e.g. `mcp-auth-adapter-github`
   - **Expiration**: 90 days (or your preference)
   - **Bypass two-factor authentication**: **checked** (required for CI)
   - **Allowed IP ranges**: leave empty
   - **Packages and scopes**: `Read and write`, `mcp-auth-adapter` package only
   - **Organizations**: `No access`
4. Click **Generate token** and copy the value
5. Go to **GitHub repo > Settings > Secrets and variables > Actions**
   (https://github.com/redhat-community-ai-tools/mcp-auth-adapter/settings/secrets/actions)
6. Click **New repository secret**:
   - **Name**: `NPM_TOKEN`
   - **Secret**: paste the token value from step 4

Once the package is published and trusted publishing is configured (see above),
this token should be deleted.

## Hotfix

Same PR + tag process from a release branch if needed (open a PR into `main` or
into the release branch per your hotfix policy, then tag after merge).

## Manual publish (emergency)

```bash
npm login && npm publish --access public
```

## Recovery: re-run a failed release

If a release workflow fails at the `npm publish` step (look for `ENEEDAUTH`
or 401/403 in the logs):

1. Verify the Trusted Publisher on npmjs.com matches the workflow exactly
   (org, repo, filename `release.yml`, case)
2. Confirm the failing run's workflow is the OIDC version (no
   `NODE_AUTH_TOKEN`, npm upgraded, `_authToken` stripped). A re-run uses the
   workflow from the **tagged commit** — if you fixed `release.yml` only on
   `main` after the tag, move the tag to a commit that includes the fix:

   ```bash
   git checkout main
   git pull
   git tag -d vX.Y.Z
   git push origin :refs/tags/vX.Y.Z
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

3. If the tagged commit already has the correct workflow, open the failed run
   in GitHub Actions and click **"Re-run failed jobs"**

The version bump is already in place; only re-tag when the workflow file on
the tagged commit itself must change. The GitHub Release may or may not have
been created depending on which step failed -- if it was created, it stays;
if not, the new run will create it. The docker job can be re-run independently
if npm publish already succeeded.
