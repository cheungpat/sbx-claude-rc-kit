# sbx-claude-rc-kit

A [Docker Sandboxes (`sbx`)](https://docs.docker.com/ai/sandboxes/) sandbox kit that runs Claude Code with **Remote Control working, including server mode** (`claude remote-control`).

It is Docker's official [`claude` kit](https://github.com/docker/sbx-kits-contrib/tree/main/claude) with one change: the top-level `credentials:` block is removed. The kit is also renamed `claude-rc`.

## Why

The stock `claude` agent makes the sbx proxy overwrite `Authorization` on `api.anthropic.com` so it can inject your host credential. Remote Control's worker and poll endpoints use their own scoped tokens, and the proxy clobbers them, so you get a 401.

- In-session `/remote-control` is fixed on the sbx nightly when `claude.remoteControl=true` is set ([#664](https://github.com/docker/sbx-releases/issues/664)).
- Server mode (`claude remote-control`) still fails on nightly with `Poll: Authentication failed (401)` ([#695](https://github.com/docker/sbx-releases/issues/695), [#483](https://github.com/docker/sbx-releases/issues/483)).

Without the `credentials:` block, the proxy has nothing to inject on `api.anthropic.com`. Requests go `forward-bypass`, and Remote Control's own tokens arrive intact. Network policy is still enforced.

## Trade-off

**You give up credential masking.** Your claude.ai OAuth token lives in the sandbox at `~/.claude/.credentials.json`. The agent can read it, and so can anything the agent runs. Only use this kit with subscription (OAuth) login. Don't put a real API key in the VM.

## Usage

```bash
git clone https://github.com/cheungpat/sbx-claude-rc-kit.git
sbx kit validate ./sbx-claude-rc-kit/claude-rc
sbx run ./sbx-claude-rc-kit/claude-rc --name claude-rc .      # sbx >= 0.42
```

Inside the sandbox:

1. Run `/login` and choose the Claude subscription option.
2. The browser won't open from the VM. Copy the URL to your host browser and paste the code back.
3. Exit.

Then start server mode:

```bash
sbx exec -it claude-rc claude remote-control
# or: sbx run claude-rc -- remote-control
```

To verify, run:

```bash
sbx policy log claude-rc
```

`api.anthropic.com:443` should show `forward-bypass`. If it shows `forward`, injection is still active and you'll get the 401 again.

To load the kit straight from git instead of a local clone, allow the source first:

```bash
sbx settings set kit.allowedSources '["docker.io/","github.com/cheungpat/"]'
sbx run "git+https://github.com/cheungpat/sbx-claude-rc-kit.git#dir=claude-rc" --name claude-rc .
```

## Keeping it in sync

`claude-rc/spec.yaml` is generated. Don't edit it by hand.

```bash
./scripts/sync-upstream.sh     # pulls docker/sbx-kits-contrib main, strips credentials:, renames
git diff                       # review, then commit
```

`claude-rc/UPSTREAM_SHA` records the upstream commit the spec was generated from. A weekly GitHub Action runs the same script and opens a PR when upstream changes.

The script fails loudly if upstream stops having a top-level `credentials:` block, or if the kit is no longer named `claude`. If that happens, review the upstream spec by hand.

## When to delete this repo

Delete it once [docker/sbx-releases#695](https://github.com/docker/sbx-releases/issues/695) ships in a stable sbx release. Then go back to the stock `claude` agent with `sbx settings set claude.remoteControl true`.

## Licence

Apache-2.0. The spec is derived from [docker/sbx-kits-contrib](https://github.com/docker/sbx-kits-contrib); see `NOTICE`.
