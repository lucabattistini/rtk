# RTK Plugin for OpenClaw

Transparently rewrites shell commands executed via OpenClaw's `exec` tool to their RTK equivalents, cutting up to 90% of the bash output that reaches the LLM context.

This is the OpenClaw equivalent of the Claude Code hooks in `hooks/rtk-rewrite.sh`.

## How it works

The plugin registers a `before_tool_call` hook that intercepts `exec` tool calls. When the agent runs a command like `git status`, the plugin delegates to `rtk rewrite` which returns the optimized command (e.g. `rtk git status`). The compressed output enters the agent's context window, saving tokens.

All rewrite logic lives in RTK itself (`rtk rewrite`). This plugin is a thin delegate -- when new filters are added to RTK, the plugin picks them up automatically with zero changes.

## Installation

### Prerequisites

RTK must be installed and available in `$PATH`:

```bash
brew install rtk
# or
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh
```

### Install the plugin

```bash
# Copy the plugin to OpenClaw's extensions directory
mkdir -p ~/.openclaw/extensions/rtk-rewrite
cp openclaw/index.ts openclaw/openclaw.plugin.json ~/.openclaw/extensions/rtk-rewrite/

# Restart the gateway
openclaw gateway restart
```

### Or install via OpenClaw CLI

```bash
openclaw plugins install ./openclaw
```

## Configuration

In `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "rtk-rewrite": {
        enabled: true,
        config: {
          enabled: true,    // Toggle rewriting on/off
          verbose: false     // Log rewrites to console
        }
      }
    }
  }
}
```

## Permissions

RTK keeps the deny gate. OpenClaw owns approval.

The plugin runs `rtk rewrite` with `RTK_REWRITE_HOST=openclaw`. That tells RTK this host applies its own exec policy -- `tools.exec.mode`, `security`, `ask` -- to whatever the `before_tool_call` hook returns, so RTK rewrites without prompting.

Without it, RTK evaluates every command against Claude Code's four settings files (`.claude/settings.json`, `.claude/settings.local.json`, and the two under `~/.claude/`) and returns "ask" for anything they do not explicitly allow. The plugin turned that into a blocking approval that denied on timeout, so a host running `tools.exec.mode=full` still stopped on every rewritable command, waiting on a decision derived from another agent's config file. See [#3908](https://github.com/rtk-ai/rtk/issues/3908).

What does **not** change:

- A command matching a `permissions.deny` rule in those Claude Code settings files is still refused, and the plugin blocks the tool call. Naming the host only relaxes an ask.
- A command containing a command substitution (`` ` ``, `$(...)`) or a redirect to a file is never rewritten, on any host.

What does change: the plugin no longer raises an approval prompt of its own. Any approval prompt you still see comes from OpenClaw.

### Writing exec rules

The plugin replaces `params.command` in `before_tool_call`, and OpenClaw folds hook adjustments into the parameters it passes to the exec tool. The tool therefore receives `rtk git push`, not `git push`. Write OpenClaw's exec allow/deny rules against the `rtk` form. This was already true before the permission change.

### rtk version

No minimum. `RTK_REWRITE_HOST` travels in the environment rather than in argv precisely so that an rtk which does not know it simply ignores it: you get the previous behaviour, a prompt on exit 3, rather than a gate that silently stops matching. The plugin treats exit 3 as "rewrite it" for the same reason, which is the convention `hooks/pi/README.md` already states for the other delegates.

## What gets rewritten

Everything that `rtk rewrite` supports (30+ commands). See the [full command list](https://github.com/rtk-ai/rtk#commands).

## What's NOT rewritten

Handled by `rtk rewrite` guards:
- Commands already using `rtk`
- Piped commands (`|`, `&&`, `;`)
- Heredocs (`<<`)
- Commands without an RTK filter

## Measured savings

| Command | Output reduction |
|---------|--------------|
| `git log --stat` | 87% |
| `ls -la` | 78% |
| `git status` | 66% |
| `grep` (single file) | 52% |
| `find -name` | 48% |

## License

Apache 2.0 -- same as RTK.
