---
title: Running coding agents without handing them my credentials
tags:
  - blog
emoji: ""
link: ""
toc: true
date: "2026-11-13"
---

> I built [lend](https://github.com/antoniomika/lend) to let remote agents use tools on my laptop with my approval. Here's how it fits into my workflow.

## Background

I use agents heavily these days as part of my toolchain. I usually have a few going at once: Claude Code working in one repo, Codex reviewing a branch in another, and something long-running in the background. I often have them review each other's work too. None of them run on my laptop. They run on VMs I can throw away, and my laptop keeps the stuff I care about: my SSH keys, my 1Password vault, and the `gh` and cloud CLIs that are logged in as me.

This works great until an agent needs one of those things. In this post I'll go over how I run agents in parallel with [herdr](https://github.com/herdrdev/herdr) and [Ovis](https://ovis.sh), and the tool I wrote to deal with credentials, [lend](https://lend.antonio.cx).

## herdr

herdr is a terminal multiplexer built for coding agents. You get the usual workspaces, tabs, and panes, and it also knows which agent is running in each pane and what it's doing. The sidebar shows whether each agent is working, waiting on me, or done, so I can tell at a glance which one needs attention.

The CLI talks to the running session, so agents can drive each other. An agent can start Codex in another pane, give it a prompt, wait for it to finish, and read what it said:

```sh
herdr agent start reviewer --kind codex --pane "$PANE"
herdr agent prompt reviewer "Review the diff on am/fix-retries" --wait --until done
herdr agent read reviewer --source recent
```

Remote hosts work the same way. `herdr machine add agent-vm` sets up herdr on the VM and adds its sessions to my local client. Those sessions keep running after I disconnect.

## Ovis

Ovis is a browser and phone client for herdr. It shows each agent's transcript as a chat, with the live terminal a tap away. When an agent asks for approval or has a question, Ovis turns it into a card I can answer and sends me a push notification. It picks up the machines saved in herdr, so my VMs show up without any extra setup. I reach it over Tailscale.

If an agent gets stuck on a question while I'm away from my desk, I answer it from my phone and the other agents keep going.

## The credential problem

Agents on a VM still need to act as me. I use them to monitor CI with `gh`, call APIs with tokens from 1Password, and SSH into other hosts. There are two usual ways to make that work, and I don't love either of them.

The first is copying credentials onto the VM. A token in `~/.config/gh/hosts.yml` can be read by every process on that machine, including the agent and anything it runs, for as long as the token is valid.

The second is forwarding my SSH agent with `ssh -A`. The private keys stay on my laptop, but the VM can use every key in the agent without asking and without any record of what it signed. It also only covers SSH keys.

What I wanted was for the credentials to stay on the laptop and for the commands that need them to run there too. The agent asks to run something, I approve it with Touch ID, and the agent gets the output back.

## lend

That's where lend comes in. lend is a proxy that sits in front of your existing SSH agent, and you forward its socket in place of the real one. SSH and Git keep working like normal, and lend adds a way to run commands through an SSH agent extension.

When an agent on the VM asks to run `gh`, the request goes over the socket and `gh` runs on my laptop with my login. The token never leaves the laptop, so the agent never has it. Every request, whether it's a signature or a command, is checked against a policy on the laptop and needs my approval before anything happens.

```text
Remote: lend exec -- command args
                  |
          forwarded SSH agent socket
                  |
Laptop: lend agent -> policy + local approval -> local command
                                           -> original SSH agent (signing)
```

The policy lives on the laptop and lists what the VM is allowed to ask for. Everything else is denied. Mine is pretty short:

```json
{
  "default": "deny",
  "commands": [
    {"executable": "/opt/homebrew/bin/gh", "approval": "biometric:inline"}
  ],
  "ssh": {
    "default": "deny",
    "keys": [
      {"fingerprint": "SHA256:...", "approval": "biometric:inline"}
    ]
  }
}
```

A rule can also take an `args_prefix` like `["pr", "view"]` if you only want to allow some subcommands. `biometric:inline` puts Touch ID right in the request dialog, so I read the request and touch once to approve it. Plain `biometric` asks for Touch ID in a second prompt after I click Allow.

To connect, I run `lend ssh` from the laptop. It starts a proxy for that one connection and stops it when SSH exits:

```sh
lend ssh --config ./policy.json -- agent-vm
```

I run herdr on the VM. Claude Code talks to lend through `lend mcp`, an MCP server with a single tool called `run_local_command`. The tool takes the command to run and a reason, and the reason is required:

```json
{
  "mcpServers": {
    "laptop": {"command": "lend", "args": ["mcp"]}
  }
}
```

Here I asked Claude Code on the VM for the lend repo's description. It called the tool with `gh repo view`, my Mac showed the request, and one touch approved it:

<img src="/lend-claude.webp" style="max-width: 100%;" alt="Claude Code on the VM asks to run gh repo view on the laptop. The lend dialog on the Mac lists the request and the caller, Touch ID approves it, and Claude prints the repo description." />

The dialog shows the full path to the executable, every argument, the directory it'll run in on the laptop, the reason, and what the remote process said about itself. Once I approve, `gh` runs on the laptop and stdout, stderr, and the exit code stream back to the agent. The reason and caller details come from the VM, so they're marked as unverified. They're there to help me decide and can't change what the policy allows.

If you'd rather skip MCP, the same request works as a plain CLI call on the VM:

```sh
lend exec --reason "Read the repo description" -- gh repo view antoniomika/lend
```

### Keeping the socket stable across reconnects

herdr panes survive an SSH disconnect, but the forwarded socket belongs to that connection. When I reconnect, SSH gives me a new socket path. Agents still running in the panes keep the old `SSH_AUTH_SOCK`.

I use a symlink at a fixed path and update it from [`~/.ssh/rc`](https://man.openbsd.org/sshd.8#SSHRC) on the VM. SSH runs this before starting the shell, so it sees the socket forwarded by the new connection:

```sh
#!/bin/sh

if [ -n "${SSH_AUTH_SOCK:-}" ] &&
    [ "$SSH_AUTH_SOCK" != "$HOME/.ssh/lend/agent.sock" ] &&
    "$HOME/.local/bin/lend" doctor --remote --socket "$SSH_AUTH_SOCK" --timeout 1s >/dev/null 2>&1; then
  install -d -m 700 "$HOME/.ssh/lend" &&
    ln -sfn -- "$SSH_AUTH_SOCK" "$HOME/.ssh/lend/agent.sock"
fi

if [ -n "${DISPLAY:-}" ] && read -r _ssh_proto _ssh_cookie; then
  case $DISPLAY in
    localhost:*) _ssh_display="unix:${DISPLAY#localhost:}" ;;
    *) _ssh_display=$DISPLAY ;;
  esac
  printf 'add %s %s %s\n' "$_ssh_display" "$_ssh_proto" "$_ssh_cookie" | /usr/bin/xauth -q -
fi
```

The check confirms the forwarded socket speaks lend before updating the link. The directory is private to my user and keeps this link separate from other agent sockets. I use lend's absolute path because this runs before my shell adds `~/.local/bin` to `PATH`. The X11 block handles cookie setup, which SSH delegates to this file when it exists.

My `~/.bashrc` checks the stable path before exporting it:

```sh
if [ -S "$HOME/.ssh/lend/agent.sock" ] &&
    "$HOME/.local/bin/lend" doctor --remote --socket "$HOME/.ssh/lend/agent.sock" --timeout 1s >/dev/null 2>&1; then
  export SSH_AUTH_SOCK="$HOME/.ssh/lend/agent.sock"
fi
```

New Bash shells in herdr panes pick up the stable path when it works. If the check fails, Bash leaves `SSH_AUTH_SOCK` alone. MCP, approval hooks, and Git inherit that path when I start them. After reconnecting, the SSH hook updates the symlink and those processes use the new socket for subsequent requests. Agents already running with an old socket path need to be restarted once with the stable path in their environment.

The SSH connection from the laptop still has to stay open. Requests fail while it's disconnected, and reconnecting starts a new lend session with fresh approvals and leases. The link points at one connection at a time, so the last update wins.

### Secrets the agent never sees

`gh` works because it's already logged in on my laptop. Some tools need a token passed in. For those, a command rule can read secrets with something like `op read` and hand them to the approved command as environment variables:

```json
{
  "secret_reader": "/opt/homebrew/bin/op",
  "secret_reader_args": ["read", "--no-newline"],
  "commands": [
    {
      "executable": "/opt/homebrew/bin/gh",
      "args_prefix": ["pr", "view"],
      "approval": "biometric",
      "secrets": {"GH_TOKEN": "op://Development/github/token"}
    }
  ]
}
```

When the agent asks to run `gh pr view` and I approve it, lend reads the token from 1Password, runs `gh` with `GH_TOKEN` set, and sends back the output. The token stays on the laptop. The VM can't pick which variables get set or which references get read. The prompt shows the variable names and references, and the values never show up in prompts or history.

Here's the VM running `gh api user` with a rule that injects `GH_TOKEN`. After I approve in lend, 1Password asks for its own Touch ID before `op read` returns the token. The VM gets my login back, and its own environment has no `GH_TOKEN`:

<img src="/lend-secret.webp" style="max-width: 100%;" alt="lend exec runs gh api user from the VM. The lend dialog lists GH_TOKEN and its 1Password reference with the value hidden, 1Password asks for Touch ID, the login prints on the VM, and env | grep -c GH_TOKEN prints 0." />

There's one catch: this only works if the approved command doesn't print its environment. If you approve `env` or `sh`, you've handed the values right back. The same goes for allowing `op` itself as a command, since `op read` prints the secret and that output goes straight to the agent.

### SSH keys

Keys get their own policy too. Keys that aren't listed are hidden from the VM, and listed keys can be limited by what they're allowed to sign. `max_signatures` caps how many signatures a key can make in a session.

### Leases

A rule can offer a lease when I approve, like five minutes or three uses. Leases only exist inside a `lend ssh` session and only match the exact command and arguments I approved. They go away when the session ends or my screen locks.

### Seeing what happened

`lend status --all --watch` shows what's connected, what's waiting, what's running, and any active leases. `lend pause --all` cancels everything and refuses new requests until I resume. `lend history` keeps a record of what ran and how it was approved. It leaves out arguments, output, and secrets, and you can opt into logging arguments.

### Answering agent prompts from the laptop

Agents also ask permission for things that have nothing to do with my credentials, like running a shell command on the VM, editing a file, or fetching a URL. `lend approve` sends those questions to the same dialog on my Mac. It doesn't run anything on the laptop. It asks me, and the agent decides what to do with the answer.

For Claude Code, it's a `PermissionRequest` hook on the VM:

```json
{
  "hooks": {
    "PermissionRequest": [
      {"matcher": "*", "hooks": [{"type": "command", "command": "lend approve --hook claude-code"}]}
    ]
  }
}
```

Now when Claude Code would normally stop and ask in the terminal, my Mac shows the command or tool input and I answer with Touch ID. The laptop policy lists which topics are allowed to ask, each with its own approval:

```json
"approvals": {
  "topics": [{"name": "claude-code", "approval": "biometric:inline"}]
}
```

There are adapters for Codex, Antigravity, and pi, plus a plain JSON format for anything else. You can use it in scripts too: `lend approve --topic deploy --title "terraform apply" && terraform apply` only applies after I approve on the laptop. If the laptop can't be reached, the hook stays out of the way and the agent asks in its own UI like it normally would.

## What lend doesn't protect against

An approved command runs as my user and can touch my files. Approval decides what runs, and lend doesn't sandbox it. If I approve a shell, an interpreter, or anything that reads instructions from stdin, I've basically approved running anything on my laptop, since stdin streams after approval and isn't shown in the prompt.

The caller details in the prompt come from the VM and can be faked. Anyone who can reach the forwarded socket on the VM, root included, can make requests while the session is open. The policy still limits what they can ask for, and every request still needs my approval.

lend is still early and runs on macOS and Linux. It doesn't allocate a PTY, so interactive programs won't work through it. On Linux, approval is a Zenity dialog or a terminal prompt, and biometric approval is macOS only.

## Prior art

[ssh-rev-exec](https://github.com/KOBA789/ssh-rev-exec) sends local execution requests through an SSH agent proxy, and [op-forward](https://github.com/ekovshilovsky/op-forward) forwards 1Password CLI commands to a machine with desktop authentication. lend is a separate implementation that uses the same agent extension trick and adds per-request policy, local approval, and key filtering.

## Trying it

Install lend on both your laptop and the remote host:

```sh
brew install antoniomika/tap/lend
# or
curl -fsSL https://github.com/antoniomika/lend/releases/latest/download/install.sh | sh
```

Run `lend doctor` on the laptop to check your policy, executables, upstream agent, and approval setup. `lend doctor --remote` on the VM checks that the forwarded socket is actually lend. The [getting started guide](https://lend.antonio.cx/getting-started.html) walks through a first session, and the [docs](https://lend.antonio.cx) cover policies, approval, and MCP. If you give it a try, I'd love to hear how it goes on [GitHub](https://github.com/antoniomika/lend).
