# NanoPilot

> ⚠️ **Work in progress — public release on May 5, 2026.** APIs, skills, and the
> deployment surface may still change. Feedback welcome; production use at your own risk.

### Your own AI agent. In Telegram. In 5–10 minutes. On hardware you already own.

[![Platform](https://img.shields.io/badge/platform-NanoPilot-5b9bd5?style=flat-square)](https://github.com/floatingsidewal/nanopilot)
[![License: MIT](https://img.shields.io/badge/license-MIT-4c1?style=flat-square)](https://github.com/floatingsidewal/nanopilot/blob/main/LICENSE)
[![Latest release](https://img.shields.io/github/v/release/floatingsidewal/nanopilot?include_prereleases&sort=semver&style=flat-square)](https://github.com/floatingsidewal/nanopilot/releases)
[![Runs on](https://img.shields.io/badge/runs%20on-Docker%20%7C%20LXC%20%7C%20ACI-0db7ed?style=flat-square)](#-pick-your-stack)
[![LLM](https://img.shields.io/badge/LLM-Ollama%20%7C%20GitHub%20Models-8a2be2?style=flat-square)](#-pick-your-stack)

> 🪄 Clone this repo. Type one slash command. Your agent is alive —
> with **your** name, **your** personality, **your** memory on **your** disk,
> reachable from anywhere you are.

You don't clone the platform. You don't learn Docker. You don't hand-edit a
`.env`. You tell the setup skill what you want, it interviews you about the
rest, then it builds it.

---

## ✨ What you get

- 🤖 A **personal AI agent** with a name, personality, and skills you define
- 💬 Reach it over **Telegram** or **Microsoft Teams** — or both
- 🖥️ Runs on your **laptop**, a **Proxmox LXC**, a **remote SSH host**, or **Azure Container Instances**
- 🧠 Uses a **local LLM via Ollama** — fully offline, no API key, no cloud — or **GitHub Models** if you prefer
- 🛡️ **Data protection built in:** 154 secret patterns, 45+ PII entity types, prompt-injection and system-prompt-leak detection
- 🔁 **Upgrade with one command** — `/upgrade-nanopilot` pulls the latest image, checks env compatibility, prompts if new vars are needed, and rolls back on failure

---

## 🚀 Quick start (5–10 minutes)

```bash
git clone https://github.com/floatingsidewal/nanopilot-workspace.git my-agent
cd my-agent
```

Open the folder in **Claude Code** or **Copilot CLI**, then run:

```
/setup-nanopilot
```

The skill reads your existing `.env`, detects what's already configured, builds
a deployment plan, and shows it to you **before** it does anything. Pick a
model, pick a channel, pick a target — it handles every command, file, and
credential.

Five to ten minutes later, you're messaging your agent.

---

## 🧩 Pick your stack

Every capability is a focused, confirm-before-acting skill. Run them one at a
time or let `/setup-nanopilot` orchestrate the whole thing.

| You want…                          | Run                                                      |
|------------------------------------|----------------------------------------------------------|
| 🦙 A local LLM, no API keys         | `/add-ollama`                                            |
| ☁️ A hosted LLM (GitHub Models)     | Set `NANOPILOT_API_KEY` to your PAT; leave `NANOPILOT_API_ENDPOINT` empty |
| 📱 Chat over Telegram               | `/add-telegram`                                          |
| 🏢 Chat over Microsoft Teams        | `/add-teams`                                             |
| 🎭 Define the agent's personality   | `/add-persona`                                           |
| 📦 Deploy local or to a remote host | `/add-target`                                            |
| 🩺 Diagnose a running agent         | `/debug`                                                 |
| ⬆️  Upgrade to the latest platform  | `/upgrade-nanopilot`                                     |

All skills live in `.claude/skills/` and auto-discover in Claude Code and
Copilot CLI. No install step.

---

## 🎭 Make it yours

Everything that makes your agent *your* agent lives in one folder:

```
workspace/
  personality.md    who your agent is
  rules.md          reflection rules it reinforces over time
  skills/           drop in Markdown; your agent learns new abilities
```

Edit. Restart. Done.

Don't want to write a persona from scratch? `/add-persona` offers an interview
mode or a library of prebuilt personalities under `.templates/`.

---

## 🛡️ Safe by default

- **Bulkhead** scans every outbound message and every tool result — 154 secret
  patterns, 45+ PII entity types, prompt-injection and system-prompt-leak
  detection. Moderate redacts; strict blocks. Enabled out of the box.
- **Allowlists everywhere** — Telegram user IDs are explicit; nothing the agent
  can reach is implicit.
- **Credentials survive upgrades** — persisted on the host in `./credentials`
  and `./ssh-keys`, never baked into images, never committed.

---

## 🆚 Why NanoPilot?

| Most "personal AI" projects        | NanoPilot                                    |
|------------------------------------|----------------------------------------------|
| Learn their framework              | Write Markdown, restart the container        |
| Use their cloud                    | Run anywhere Docker runs — laptop to LXC     |
| Trust their data plane             | Your disk, your network, your keys           |
| Hand-write Dockerfiles and `.env`  | One slash command interviews you             |
| Hope an upgrade doesn't break you  | `/upgrade-nanopilot` pulls, diffs, verifies, rolls back |

**You own your agent. You own its data. You own where it runs.**

---

## 📁 What's in this repo

```
workspace/          your agent's persona, rules, and skills
.claude/skills/     orchestrator and sub-skills (/setup-nanopilot and friends)
.templates/         prebuilt personalities you can clone
docker-compose.yml  runtime definition
Dockerfile          tool container
.env.example        every variable /setup-nanopilot will populate
INSTALL.md          manual deployment reference (the skill is the preferred path)
```

---

## 🔀 Mirror provenance

This repo is auto-generated from
[`floatingsidewal/nanopilot/agents/.template/`](https://github.com/floatingsidewal/nanopilot/tree/main/agents/.template)
on every push to main. Don't open PRs here — open them against the platform
repo. Source commit: `dbe1f41`.
