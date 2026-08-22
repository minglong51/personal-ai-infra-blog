# personal-ai-infra-blog

Mirrors of my writing on personal AI infrastructure — agent fleets, deterministic LLM
workflows, attention design, local model serving — plus the configs those posts reference.

**My site is canonical: [minglongpan.com/writing](https://www.minglongpan.com/writing).**
The files here are point-in-time copies. If a post has been revised, the site has the
current version.

## Writing

| Date | Post | Mirror |
|---|---|---|
| 2026-08-20 | [A coherent virtual pet without an LLM](https://www.minglongpan.com/writing/a-coherent-virtual-pet-without-an-llm) | [`writing/a-coherent-virtual-pet-without-an-llm.md`](./writing/a-coherent-virtual-pet-without-an-llm.md) |
| 2026-08-10 | [Why tmux is the wrong IPC layer for an agent fleet](https://www.minglongpan.com/writing/why-tmux-is-the-wrong-ipc-layer-for-an-agent-fleet) | [`writing/why-tmux-is-the-wrong-ipc-layer-for-an-agent-fleet.md`](./writing/why-tmux-is-the-wrong-ipc-layer-for-an-agent-fleet.md) |
| 2026-08-08 | [Why a self-improving agent's accept rate isn't a quality metric](https://www.minglongpan.com/writing/accept-rate-is-not-a-quality-metric) | [`writing/accept-rate-is-not-a-quality-metric.md`](./writing/accept-rate-is-not-a-quality-metric.md) |
| 2026-07-24 | [finance-os: a decision system, not a stock picker](https://www.minglongpan.com/writing/finance-os-design-notes) | [`writing/finance-os-design-notes.md`](./writing/finance-os-design-notes.md) |
| 2026-07-24 | [How I run a personal agent fleet](https://www.minglongpan.com/writing/how-i-run-a-personal-agent-fleet) | [`writing/how-i-run-a-personal-agent-fleet.md`](./writing/how-i-run-a-personal-agent-fleet.md) |
| 2026-07-24 | [Persona-routed agent tooling: one stdio server, every team's capabilities](https://www.minglongpan.com/writing/mcp-resource-layering-pattern) | [`writing/mcp-resource-layering-pattern.md`](./writing/mcp-resource-layering-pattern.md) |
| 2026-07-24 | [Valhalla: an attention layer for a fleet of agents](https://www.minglongpan.com/writing/valhalla-design-notes) | [`writing/valhalla-design-notes.md`](./writing/valhalla-design-notes.md) |
| 2026-07-20 | [Three agent frameworks converged on a control-plane protocol this month. I run my fleet on tmux and chose not to adopt it.](https://www.minglongpan.com/writing/three-agent-frameworks-converged-on-a-control-plane-protocol) | [`writing/three-agent-frameworks-converged-on-a-control-plane-protocol.md`](./writing/three-agent-frameworks-converged-on-a-control-plane-protocol.md) |
| 2026-06-01 | [Anti-Engagement AI: Software You Can Walk Away From](https://www.minglongpan.com/writing/anti-engagement-ai) | [`writing/anti-engagement-ai.md`](./writing/anti-engagement-ai.md) |
| 2026-06-01 | [ThreadLang: a deterministic DSL for LLM workflows](https://www.minglongpan.com/writing/threadlang) | [`writing/threadlang.md`](./writing/threadlang.md) |

Also on Substack: [Anti-Engagement AI: Software You Can Walk Away From](https://minglong1.substack.com/p/anti-engagement-ai-software-you-can).

## Configs

### `post-01-tailscale-ollama/`

Binding Ollama to `0.0.0.0` and installing Tailscale does not restrict it to the tailnet.
`0.0.0.0` means every interface — home Wi-Fi, Ethernet, and whatever café network the
laptop joins next. Binding to the Tailscale interface IP instead means the kernel only
answers traffic that arrived over the tunnel.

- **`com.minglongpan.ollama.plist`** — macOS LaunchAgent for `ollama serve`, bound to a
  Tailscale IP. Edit the IP and log paths, drop it in `~/Library/LaunchAgents/`, then
  `launchctl load -w` it. `brew services` and `.zshrc` exports both fail here: launchd
  does not see your shell environment, so the bind silently stays on `127.0.0.1`.
- **`tailscale-acl.json`** — a minimal ACL scoping port `11434` to a tagged device, so
  the bind is backed by a tailnet rule as well.

Both are sanitized templates. Replace `100.x.x.x` with your own Tailscale IP
(`tailscale ip -4`) and `/Users/me` with your home directory.

On Linux the same pattern works through a systemd drop-in:

```
# /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_HOST=100.x.x.x:11434"
```

Then `sudo systemctl daemon-reload && sudo systemctl restart ollama`.

## Use at your own risk

These are the configs I run, sanitized. They work on my machines. Read them, understand
the pattern, and adapt rather than copy.

## License

MIT.
