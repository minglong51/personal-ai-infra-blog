# personal-ai-infra-blog

Configs, scripts, and supporting files for my Medium series on personal AI infrastructure — the boring plumbing that turns LLMs into systems you actually rely on.

## Posts

| # | Post | Code |
|---|---|---|
| 1 | [Don't 0.0.0.0 Your Ollama](https://medium.com/[YOUR_HANDLE]/[POST_SLUG]) — A Tailscale pattern most tutorials get wrong | [`post-01-tailscale-ollama/`](./post-01-tailscale-ollama) |
| 2 | *(Coming next: Claude→Ollama fallback routing on the agent side, with real cost numbers)* | — |

## What's in here

### `post-01-tailscale-ollama/`

- **`com.minglongpan.ollama.plist`** — macOS LaunchAgent for `ollama serve`, bound to the Tailscale interface IP (not `0.0.0.0`). Drop into `~/Library/LaunchAgents/`, edit the IP and paths, then `launchctl load -w` it. The post explains why bind-to-IP matters.
- **`tailscale-acl.json`** — minimal Tailscale ACL that scopes Ollama port `11434` to a specific device tag, for defense-in-depth inside your tailnet.

## Use at your own risk

These are the configs I'm running, sanitized. They work on my machines. They might not on yours. Read the post, understand the pattern, then adapt rather than copy.

## Linux equivalent

The bind pattern works the same on Linux — replace the LaunchAgent with a systemd drop-in:

```
# /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_HOST=100.x.x.x:11434"
```

Then `sudo systemctl daemon-reload && sudo systemctl restart ollama`.

## Following along

- **Medium:** [@YOUR_MEDIUM_HANDLE](https://medium.com/@YOUR_MEDIUM_HANDLE)
- **New post roughly every 10 days.**

## License

MIT. Take it, fork it, ship it.
