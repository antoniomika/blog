---
title: Running coding agents without handing them my credentials
tags:
  - blog
emoji: ""
link: ""
date: "2026-11-13"
draft: true
---

Most of my coding now happens through agents, and I usually have several running at once: Claude Code working on one repo, Codex reviewing a branch in another, and something long-running in the background. <!-- TODO: adjust to the real mix --> None of them run on my laptop. They run on VMs I can throw away, and the laptop keeps what I care about: my SSH keys, my 1Password vault, and the `gh` and cloud CLIs that are logged in as me.

That split works well until an agent needs one of those things. This post covers the setup I use to run agents in parallel, [herdr](https://github.com/herdrdev/herdr) and [Ovis](https://ovis.sh), and the tool I wrote for the credential problem, [lend](https://lend.antonio.cx).

## herdr

herdr is a terminal multiplexer built around coding agents. It has the usual workspaces, tabs, and panes. It also recognizes which agent is running in a pane and what state it's in (working, blocked on an approval or a question, or done) and shows that in the sidebar for every workspace. I can tell at a glance which agent is waiting on me.

Its CLI talks to the running session, so agents can drive each other. An orchestrating agent can start Codex in another pane, hand it a prompt, wait for it to finish, and read the result:

```sh
herdr agent start reviewer --kind codex --pane "$PANE"
herdr agent prompt reviewer "Review the diff on am/fix-retries" --wait --until done
herdr agent read reviewer --source recent
```

Remote hosts work the same way as local ones. `herdr machine add agent-vm` installs herdr on the host if needed, starts its server, and adds its sessions to the local client. Those sessions keep running after I disconnect.

## Ovis

Ovis is a browser and phone client for herdr. It attaches to the same panes as the TUI and shows each agent's transcript as a chat, with the live terminal one tap away. Approval prompts and questions become cards I can answer, and it sends a push notification when an agent needs input or finishes. It reads the machines saved in herdr, so the remote VMs show up with no extra setup. I reach it over Tailscale.

When an agent stops on a question while I'm away from the desk, I answer it from my phone and the other agents keep going.

## The credential problem

Agents on a remote VM still need to act as me. They open PRs with `gh`, call APIs with tokens from 1Password, sign commits, and SSH into other hosts. The usual ways to allow that both have problems.

Copying credentials onto the VM means a token in `~/.config/gh/hosts.yml` is readable by every process on that machine, including the agent and anything it runs, for as long as the token is valid.

Forwarding my SSH agent with `ssh -A` keeps the private keys on the laptop. The remote side can still use every key in the agent with no prompt and no record of what it signed, and it only covers SSH keys.

I wanted the credentials to stay on the laptop and the commands that use them to run there too. The agent asks to run something, I approve it with Touch ID, and it only gets the output back.

## lend

lend is a proxy that sits in front of your existing SSH agent. You forward its socket in place of the real one. SSH and Git keep working because lend speaks the agent protocol, and it adds a command execution channel over an SSH agent extension. When the agent asks to run `gh`, the request crosses the socket and `gh` runs on my laptop with my login. The token never leaves the laptop, so the agent never has it. Every request, a signature or a command, is checked against a policy on the laptop and needs local approval before anything happens.

```text
Remote: lend exec -- command args
                  |
          forwarded SSH agent socket
                  |
Laptop: lend agent -> policy + local approval -> local command
                                           -> original SSH agent (signing)
```

The policy lives on the laptop and lists the commands and keys the remote side can use. Everything else is denied. Mine is short:

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

A rule can also take an `args_prefix` such as `["pr", "view"]` to allow only some subcommands. `biometric:inline` puts Touch ID inside the request dialog, so one touch approves the request I'm reading. `biometric` asks for Touch ID in a second prompt after I click Allow.

From the laptop, `lend ssh` starts a proxy for that one connection and stops it when SSH exits:

```sh
lend ssh --config ./policy.json -- agent-vm
```

<!-- TODO: confirm how this lines up with herdr remote sessions. Panes started under an earlier SSH session keep a stale SSH_AUTH_SOCK. -->
I run herdr on the VM, and the agents in its panes use the forwarded socket. Claude Code reaches it through `lend mcp`, an MCP server with one tool, `run_local_command`, which takes the command and a required reason:

```json
{
  "mcpServers": {
    "laptop": {"command": "lend", "args": ["mcp"]}
  }
}
```

Here I asked Claude Code on the VM for the lend repo's description. It called the tool with `gh repo view`, my Mac showed the request, and one touch approved it:

<img src="/lend-claude.webp" style="max-width: 100%;" alt="Claude Code on the VM asks to run gh repo view on the laptop. The lend dialog on the Mac lists the request and the caller, Touch ID approves it, and Claude prints the repo description." />

The dialog shows the resolved executable, every argument, the local working directory, the reason, and what the remote process reported about itself. Once I touch, `gh` runs on the laptop, and stdout, stderr, and the exit code stream back to the agent. The reason and caller details are labeled as unverified and can't change what the policy allows. They're there to help me decide.

Without MCP, the same request is a CLI call on the VM:

```sh
lend exec --reason "Read the repo description" -- gh repo view antoniomika/lend
```

### Secrets the agent never sees

`gh` works because it's already logged in on the laptop. Some tools need a token passed in. A command rule can pull secrets from a local reader such as `op read` and pass them to the approved process as environment variables:

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

The agent asks to run `gh pr view`. Once I approve, lend reads the token from 1Password, runs `gh` with `GH_TOKEN` set, and returns the output. The token stays on the laptop. The remote request can't choose which variables get set or which references get read. The prompt shows the variable names and references, and the values never appear in prompts or history.

Here the VM runs `gh api user` with a rule that injects `GH_TOKEN`. After I approve in lend, 1Password asks for its own Touch ID before `op read` returns the token. The VM gets my login back, and its own environment has no `GH_TOKEN`:

<img src="/lend-secret.webp" style="max-width: 100%;" alt="lend exec runs gh api user from the VM. The lend dialog lists GH_TOKEN and its 1Password reference with the value hidden, 1Password asks for Touch ID, the login prints on the VM, and env | grep -c GH_TOKEN prints 0." />

This only holds if the approved command doesn't print its environment, so rules should name commands that won't. Approving `env` or `sh` would hand the values back. The same goes for allowing `op` as a command: `op read` prints the secret, and that output goes to the agent.

### Tighter SSH forwarding

Keys get their own policy. Unlisted keys are hidden from the remote side, and listed keys can be limited by what they sign. One of my keys only signs Git commits (`"purposes": ["sshsig"], "namespaces": ["git"]`), so the VM can sign commits with it and can't use it to log in anywhere. `max_signatures` caps how many signatures a key produces in a session. Adding, removing, and locking keys in the upstream agent are separate operations with their own rules.

### Leases

Approving every `git commit` with Touch ID gets old. A rule can offer a lease at approval time, such as five minutes or three uses. Leases exist only inside a `lend ssh` session and match the exact command and arguments I approved. They're dropped when the session ends or the screen locks.

### Seeing what happened

`lend status --all --watch` shows live connections, pending requests, running commands, and active leases. `lend pause --all` cancels everything and refuses new requests until I resume. `lend history` records what ran and how it was approved. Arguments, output, and secrets are left out unless you opt into logging arguments.

### Answering agent prompts from the laptop

Agents ask permission for things that never touch my credentials too: running a shell command on the VM, editing a file, fetching a URL. `lend approve` sends those questions to the same dialog. It runs nothing on the laptop. It asks, and the requester decides what the answer allows.

For Claude Code, a `PermissionRequest` hook on the VM does it:

```json
{
  "hooks": {
    "PermissionRequest": [
      {"matcher": "*", "hooks": [{"type": "command", "command": "lend approve --hook claude-code"}]}
    ]
  }
}
```

When Claude Code would stop and ask in the terminal, my Mac shows the command or tool input instead, and Touch ID answers it. The laptop policy lists which topics can ask, each with its own approval:

```json
"approvals": {
  "topics": [{"name": "claude-code", "approval": "biometric:inline"}]
}
```

There are adapters for Codex, Antigravity, and pi, and a plain JSON format for anything else. Scripts can use it directly: `lend approve --topic deploy --title "terraform apply" && terraform apply` only applies after I approve on the laptop. If the laptop can't be reached, the hook makes no decision and the agent asks in its own UI as usual.

## What lend doesn't protect against

An approved command runs as my user with access to my files. Approval decides what runs and doesn't sandbox it. If I approve a shell, an interpreter, or anything that reads instructions from stdin, I've approved arbitrary execution on my laptop, since stdin streams after approval and isn't shown in the prompt.

The caller details in the prompt come from the remote side and can be spoofed. Anyone who can reach the forwarded socket on the VM, including root, can make requests while the session is open. The policy limits what they can ask for, and every request still needs my approval.

lend is an early release for macOS and Linux. It doesn't allocate a PTY, so interactive programs don't work through it. On Linux, approval is a Zenity dialog or a terminal prompt, and biometric approval is macOS only.

## Prior art

[ssh-rev-exec](https://github.com/KOBA789/ssh-rev-exec) carries local execution requests through an SSH agent proxy, and [op-forward](https://github.com/ekovshilovsky/op-forward) forwards 1Password CLI commands to a machine with desktop authentication. lend is an independent implementation that uses the same agent extension transport and adds per-request policy, local approval, and key filtering.

## Trying it

Install lend on both the laptop and the remote host:

```sh
brew install antoniomika/tap/lend
# or
curl -fsSL https://github.com/antoniomika/lend/releases/latest/download/install.sh | sh
```

`lend doctor` on the laptop checks your policy, executables, upstream agent, and approval backend. `lend doctor --remote` on the VM checks that the forwarded socket is a lend proxy. The [getting started guide](https://lend.antonio.cx/getting-started.html) walks through a first session, and the [docs](https://lend.antonio.cx) cover policies, approval, and MCP. Issues and feedback are welcome on [GitHub](https://github.com/antoniomika/lend).
