# PR preview deploys — setup

This publishes each pull request's build to a hosted, gated preview URL so decks
and Engagement Wrapped PRs can be smoke-tested on a real phone before merge. It
replaces the ad-hoc "serve it off a laptop on the LAN" step, which isn't
reachable from outside the local network.

## Activate the workflow (one step, needs the `workflow` token scope)

The workflow YAML is shipped here as **`.github/preview-workflow.yml.txt`**, not as
a live `.github/workflows/*.yml`. Reason: pushing a file under `.github/workflows/`
requires a git token with the **`workflow`** OAuth scope, and the token this repo
was pushed with only has `repo`. To turn it on, do **one** of:

- **Grant the scope, then rename.** Give the push token the `workflow` scope
  (`gh auth refresh -h github.com -s workflow`, or add it to the PAT), then
  `git mv .github/preview-workflow.yml.txt .github/workflows/pr-preview.yml` and push.
- **Add it in the browser.** GitHub web UI → Add file → Create new file →
  path `.github/workflows/pr-preview.yml` → paste the contents of
  `preview-workflow.yml.txt` → commit. (Web commits carry the scope automatically.)

Until it lives under `.github/workflows/`, it does not run. Nothing about
production changes either way.

## Why not GitHub Pages

Production is GitHub Pages, served from **`main` → decks.scrollmedia.co**. A repo's
GitHub Pages serves exactly **one** source. Per-PR previews on that same site would
require either committing preview builds into `main` (pollutes production) or moving
production to a `gh-pages` branch (touches production). Neither is acceptable.

**Cloudflare Pages** gives a separate per-branch preview URL on `*.pages.dev` and
**never touches the GitHub Pages production site.** `main` → decks.scrollmedia.co is
completely unaffected; it still deploys via GitHub Pages on merge, exactly as today.

## One-time setup (what you flip)

1. **Cloudflare account** (free tier is fine). Note your **Account ID** (Dashboard →
   any domain → right sidebar, or Workers & Pages → Account details).
2. **Create the Pages project** once, named exactly **`scroll-proposals-preview`**:
   Workers & Pages → Create → Pages → **Direct Upload** → name it → Create. Leave it
   empty; the workflow uploads to it. (Set its *production branch* to `main` in the
   project settings so PR branches are always treated as **preview** deployments.)
3. **API token**: My Profile → API Tokens → Create Token → template **"Edit Cloudflare
   Workers"**, or a custom token with **Account › Cloudflare Pages › Edit**. Copy it.
4. **Add two GitHub repo secrets** (Settings → Secrets and variables → Actions → New
   repository secret):
   - `CLOUDFLARE_API_TOKEN` — the token from step 3
   - `CLOUDFLARE_ACCOUNT_ID` — the Account ID from step 1

**No DNS changes.** Previews live on `*.pages.dev`. `decks.scrollmedia.co` DNS and the
GitHub Pages config stay exactly as they are.

## How the preview URL is produced

- On every PR (same-repo branch), the workflow runs `wrangler pages deploy .` against
  the `scroll-proposals-preview` project with `--branch=<the PR branch>`. Because the
  branch is not the project's production branch, Cloudflare records it as a **preview**
  deployment and returns a preview URL.
- The workflow posts (and updates) a PR comment with:
  - `https://<hash>.scroll-proposals-preview.pages.dev/` (root)
  - `…/launch-party-wrapped/` with the gate code `launchparty.scroll`
- The whole repo root is deployed, so **every deck** in a PR gets a preview path, not
  just Launch Party Wrapped.

## Notes

- Gates are client-side JS and work identically on the preview host.
- Preview pages keep their `noindex,nofollow` meta.
- Fork PRs are skipped (GitHub does not expose secrets to forks); same-repo branches
  (the normal `claude/*` flow) are covered.
- Until the two secrets exist, the workflow runs and no-ops on the deploy step — it
  will not fail production and cannot affect `main`.
