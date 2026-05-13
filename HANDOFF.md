# rask CLI — first-deploy checklist

The CLI is built and the server already accepts CLI tokens (auth helper +
route guard merged into `rask-podcast-manager`). To make it actually work for
the team, you need to:

1. Mint a real token and store it in Secret Manager.
2. Bind the secret to the live Cloud Run service.
3. Create two GitHub repos and push.
4. Cut a v0.1.0 release of `rask-cli`.
5. Patch the formula with the real sha256.
6. Run `brew install benchristie02/tap/rask` from a teammate's laptop to verify.

Total time: ~20-30 minutes.

---

## 1. Mint the CLI token and store it in Secret Manager

```bash
# Generate a strong token (45 chars of base64).
TOKEN=$(openssl rand -base64 32)
echo "$TOKEN"   # Copy this — you'll paste it into `rask login` later.

# Store in Secret Manager. The `:latest` reference Cloud Run uses will resolve here.
printf '%s' "$TOKEN" | gcloud secrets create RASK_CLI_TOKEN \
  --project=rask-podcast-manager-5c2dd \
  --replication-policy=automatic \
  --data-file=-

# Grant the runtime SA read access.
gcloud secrets add-iam-policy-binding RASK_CLI_TOKEN \
  --project=rask-podcast-manager-5c2dd \
  --member=serviceAccount:rask-runtime@rask-podcast-manager-5c2dd.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

## 2. Bind the secret to Cloud Run

```bash
gcloud run services update rask \
  --region=australia-southeast2 \
  --update-secrets=RASK_CLI_TOKEN=RASK_CLI_TOKEN:latest
```

This triggers a new revision. The next deploy (built from the latest commit
of `rask-podcast-manager`, which already contains the `requireUserOrCliToken`
change) will pick the token up automatically.

If you haven't deployed since today's commits, do that now:

```bash
cd ~/Desktop/rask-podcast-manager
gcloud builds submit --tag australia-southeast2-docker.pkg.dev/rask-podcast-manager-5c2dd/rask/app:latest
gcloud run deploy rask \
  --image=australia-southeast2-docker.pkg.dev/rask-podcast-manager-5c2dd/rask/app:latest \
  --region=australia-southeast2
```

## 3. Create the two GitHub repos

You don't have `gh` installed. Either install it (`brew install gh && gh auth login`) and use the commands below, or create the repos through github.com and skip to the `git remote add` step.

### With gh

```bash
brew install gh
gh auth login

# rask-cli (source repo)
cd ~/Desktop/rask-cli
git add -A
git commit -m "Initial commit: rask CLI v0.1.0"
gh repo create rask-cli --public --source=. --remote=origin --push

# homebrew-tap
cd ~/Desktop/homebrew-tap
git add -A
git commit -m "Initial commit: rask formula"
gh repo create homebrew-tap --public --source=. --remote=origin --push
```

### Without gh (via web)

1. Go to https://github.com/new
2. Create `benchristie02/rask-cli` — public, no README, no .gitignore (we have both)
3. Create `benchristie02/homebrew-tap` — same
4. Then in each local repo:
   ```bash
   git add -A
   git commit -m "Initial commit"
   git remote add origin git@github.com:benchristie02/<repo-name>.git
   git push -u origin main
   ```

> **Note on public-vs-private.** The Homebrew tap repo must be public (or
> teammates need a `GITHUB_TOKEN` configured). The `rask-cli` source repo can
> be either — but if private, the tarball URL in the formula needs an auth
> mechanism, which adds friction. Recommendation: keep both public, since
> the CLI script only contains the API URL (which is not a secret) and
> users still need a token to do anything.

## 4. Tag and release v0.1.0

```bash
cd ~/Desktop/rask-cli
git tag v0.1.0
git push origin v0.1.0

# With gh
gh release create v0.1.0 --title "v0.1.0" --notes "Initial release. \`rask chapters\` for YouTube chapter generation."

# Or in the GitHub UI: Releases → "Draft a new release" → choose v0.1.0 tag → publish
```

GitHub auto-generates a source tarball at:
`https://github.com/benchristie02/rask-cli/archive/refs/tags/v0.1.0.tar.gz`

## 5. Patch the formula with the real sha256

```bash
SHA=$(curl -sL https://github.com/benchristie02/rask-cli/archive/refs/tags/v0.1.0.tar.gz | shasum -a 256 | awk '{print $1}')
echo "$SHA"

cd ~/Desktop/homebrew-tap
# Replace REPLACE_WITH_SHA256_OF_TARBALL in Formula/rask.rb with the value above.
sed -i '' "s/REPLACE_WITH_SHA256_OF_TARBALL/$SHA/" Formula/rask.rb
git add Formula/rask.rb
git commit -m "rask 0.1.0: set tarball sha256"
git push
```

## 6. Verify install on your laptop

```bash
brew install benchristie02/tap/rask

rask version             # should print: rask 0.1.0
rask whoami              # should print prod API URL, no token

rask login               # paste the token from step 1
rask whoami              # should now show "token: stored"

# Real test (burns ~$0.50 of Deepgram + Anthropic):
rask chapters "https://www.youtube.com/watch?v=YOUR_TEST_VIDEO" \
  --prompt "Return YouTube-style chapter markers. Format: MM:SS Title, one per line. No commentary."
```

If `rask chapters` errors out, check Cloud Run logs:

```bash
gcloud run services logs read rask --region=australia-southeast2 --limit=50
```

## After it works

- Share the install command + a token with each teammate (via 1Password or
  similar — **do not** paste tokens in Slack).
- Delete the legacy `YT_COOKIES` secret while you're in Secret Manager:
  ```bash
  gcloud secrets delete YT_COOKIES --project=rask-podcast-manager-5c2dd
  ```
- Token rotation procedure: generate a new secret version
  (`gcloud secrets versions add RASK_CLI_TOKEN --data-file=-`), redeploy
  the service, ask everyone to `rask login` with the new value. The old
  version can be disabled after a few days.
