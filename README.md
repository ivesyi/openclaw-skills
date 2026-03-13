# OpenClaw Skills

Custom skills for [OpenClaw](https://openclaw.ai) — the self-hosted AI gateway.

## Skills

| Skill | Description |
|-------|-------------|
| [page-agent](./page-agent/) | Inject [page-agent](https://github.com/nicepkg/page-agent) into the browser for intelligent, multi-step web automation |

## Installation

### Via OpenClaw config

Add this repo as an extra skill directory in `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    load: {
      extraDirs: ["/path/to/openclaw-skills"]
    }
  }
}
```

### Manual

Copy the skill folder into your OpenClaw skills directory:

```bash
cp -r page-agent ~/.openclaw/skills/
```

## License

MIT
