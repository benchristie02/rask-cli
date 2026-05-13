# rask

Internal command-line tool for [Rask Australia](https://rask.au). Currently
ships one user-facing feature: generating YouTube chapter markers from a video
URL by downloading the audio locally and uploading it to the rask app's
chapters endpoint.

YouTube extraction is intentionally done on the user's machine rather than on
the server, because cloud-IP YouTube requests hit Google's anti-bot wall.

## Install

```bash
brew install benchristie02/tap/rask
```

(That command also installs `yt-dlp` and `jq` automatically.)

One-time setup:

```bash
rask login
# paste your API token when prompted
```

Ask whoever owns the rask app for a token if you don't have one.

## Use

```bash
# Inline prompt
rask chapters "https://youtu.be/abc123" --prompt "Return YouTube-style chapter markers, one per line, in MM:SS Title format."

# Prompt from a file (recommended for anything non-trivial)
rask chapters "https://youtu.be/abc123" --prompt-file ~/prompts/chapters.txt

# Prompt piped on stdin
pbpaste | rask chapters "https://youtu.be/abc123" --prompt-stdin
```

The command prints the generated chapters to stdout. Pipe it to `pbcopy` to
copy directly to the clipboard:

```bash
rask chapters "https://youtu.be/abc123" --prompt-file ~/chapters.txt | pbcopy
```

For the full JSON response (chapters + transcript preview + duration):

```bash
rask chapters "https://youtu.be/abc123" --prompt-file ~/chapters.txt --json
```

## Configuration

| Variable        | Purpose                                                  |
| --------------- | -------------------------------------------------------- |
| `RASK_API_URL`  | Override the API base URL (defaults to the prod app).    |
| `RASK_TOKEN`    | Override the stored token (useful for CI / one-off use). |

The token is stored at `~/.config/rask/credentials` (mode 600).

## Other commands

```bash
rask login      # store a token
rask logout     # forget the stored token
rask whoami     # show current API URL and credential state
rask version    # print version
rask help       # print this list
```

## Local development

The script is plain bash, no build step. Edit `bin/rask` and run it directly:

```bash
./bin/rask whoami
```

When iterating against a local dev server:

```bash
export RASK_API_URL=http://localhost:3000
export RASK_TOKEN=$(grep RASK_CLI_TOKEN ../rask-podcast-manager/.env.local | cut -d= -f2)
./bin/rask chapters "https://youtu.be/dQw4w9WgXcQ" --prompt "Summary in 5 chapters."
```

## Releasing

1. Bump `VERSION` in `bin/rask`.
2. Tag and push: `git tag v0.1.0 && git push origin v0.1.0`.
3. Create a GitHub release from the tag. GitHub auto-generates a source tarball
   at `https://github.com/benchristie02/rask-cli/archive/refs/tags/v0.1.0.tar.gz`.
4. Compute the tarball sha256: `shasum -a 256 <(curl -sL <tarball-url>)`.
5. Update `Formula/rask.rb` in `benchristie02/homebrew-tap` with the new
   `url` + `sha256` + `version`, commit, push.

Users get the new version with `brew upgrade rask`.

## License

MIT — see `LICENSE`.
