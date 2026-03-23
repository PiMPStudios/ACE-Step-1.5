# ACE-Step MCP Server

Exposes ACE-Step as a set of [Model Context Protocol](https://modelcontextprotocol.io) tools so AI coding assistants (Claude Code, Cursor, etc.) can generate music programmatically — no manual UI interaction required.

## How It Works

The MCP server uses **stdio transport**: your MCP client (e.g. Claude Code) spawns the server process on demand. When a tool is first called, the server automatically boots `acestep-api` in the background, waits for it to be ready, then proxies all tool calls to the REST API at `localhost:8001`. When the MCP client disconnects, the API server is shut down cleanly.

No manual startup required.

## Requirements

- [uv](https://docs.astral.sh/uv/) installed (`curl -LsSf https://astral.sh/uv/install.sh | sh`)
- ACE-Step repo cloned and synced (`uv sync`)
- MCP client that supports stdio transport (Claude Code, Cursor, etc.)

The MCP server script uses [PEP 723 inline dependencies](https://peps.python.org/pep-0723/), so `uv run` automatically installs `mcp` and `httpx` into an isolated cache on first run — no manual `pip install` needed.

## Setup

Add the following entry to your MCP client config (e.g. `~/.claude.json` for Claude Code):

```json
"acestep-mcp": {
  "type": "stdio",
  "command": "uv",
  "args": [
    "run",
    "--script",
    "/path/to/ACE-Step-1.5/acestep_mcp_server.py"
  ],
  "env": {
    "ACESTEP_DIR": "/path/to/ACE-Step-1.5",
    "PORT": "8001",
    "ACESTEP_LM_BACKEND": "mlx",
    "ACESTEP_DEVICE": "mps",
    "ACESTEP_LM_MODEL_PATH": "acestep-5Hz-lm-1.7B",
    "ACESTEP_CONFIG_PATH": "acestep-v15-turbo",
    "ACESTEP_START_TIMEOUT": "180"
  }
}
```

Replace `/path/to/ACE-Step-1.5` with the actual path to your cloned repo.

### Platform-Specific Env Vars

| Platform | `ACESTEP_LM_BACKEND` | `ACESTEP_DEVICE` | Recommended LM Model |
|---|---|---|---|
| macOS Apple Silicon | `mlx` | `mps` | `acestep-5Hz-lm-1.7B` |
| Linux/Windows CUDA | `vllm` | `cuda` | `acestep-5Hz-lm-1.7B` or `4B` |
| CPU only | `pt` | `cpu` | `acestep-5Hz-lm-0.6B` |

### All Environment Variables

| Variable | Default | Description |
|---|---|---|
| `ACESTEP_DIR` | Script directory | Path to ACE-Step repo root |
| `PORT` | `8001` | API server port |
| `ACESTEP_LM_BACKEND` | `mlx` | LM inference backend (`mlx`, `vllm`, `pt`) |
| `ACESTEP_DEVICE` | `mps` | Compute device (`mps`, `cuda`, `cpu`) |
| `ACESTEP_LM_MODEL_PATH` | `acestep-5Hz-lm-1.7B` | Language model size |
| `ACESTEP_CONFIG_PATH` | `acestep-v15-turbo` | DiT model variant |
| `ACESTEP_API_KEY` | _(none)_ | Optional API key for the server |
| `ACESTEP_START_TIMEOUT` | `180` | Seconds to wait for server startup |

## Available Tools

### `generate_music`

Generate one or more music tracks. Returns a list of output file paths.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `prompt` | str | required | Style/mood/instrument description |
| `duration` | float | `120.0` | Length in seconds |
| `lyrics` | str | `"[instrumental]"` | Lyrics text, or `[instrumental]` |
| `bpm` | int | `72` | Beats per minute (explicit, not inferred from prompt) |
| `key` | str | `"C minor"` | Musical key e.g. `"C minor"`, `"A major"` |
| `seed` | int | `-1` | Random seed; `-1` = random. Use fixed seeds for variants |
| `num_samples` | int | `1` | Variants to generate in one call (1–8) |
| `guidance_scale` | float | `10.0` | Prompt adherence (higher = more literal) |
| `steps` | int | `20` | Inference steps (8 = fast draft, 20 = clean, 50 = high quality) |
| `audio_format` | str | `"wav"` | Output format: `wav`, `mp3`, `flac`, `ogg`, `aac` |
| `output_path` | str | `./acestep_output/` | Directory to write output files |

**Example — generate 3 variants for comparison:**
```
generate_music(
    prompt="dark ambient, drone synth pads, sub-bass pulse, clinical cold atmosphere",
    duration=120,
    bpm=72,
    key="C minor",
    num_samples=3,
    seed=1,        # seeds 1, 2, 3 across three calls for reproducible variants
    guidance_scale=10.0,
    steps=20,
    audio_format="wav"
)
```

### `format_music_prompt`

Use ACE-Step's built-in LLM to expand a rough description into a structured prompt with suggested BPM, key, duration, and time signature.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `prompt` | str | required | Rough description e.g. `"tense ambient for a datacenter game"` |
| `lyrics` | str | `""` | Optional starting lyrics |
| `temperature` | float | `0.85` | LLM creativity (0 = deterministic, 1 = creative) |

### `get_server_health`

Check whether the ACE-Step API is running and which models are loaded. Safe to call before any generation — will not boot the server.

### `list_models`

List available DiT and LM models, showing which are currently loaded.

### `load_lora`

Load a LoRA adapter for style fine-tuning.

| Parameter | Type | Description |
|---|---|---|
| `lora_path` | str | Path to the LoRA weights directory |
| `adapter_name` | str | Optional name for the adapter |

### `set_lora_strength`

Adjust LoRA influence on generation (0.0 = no effect, 1.0 = full effect).

### `unload_lora`

Remove the currently active LoRA adapter.

## Typical Workflow

```
1. Call generate_music with num_samples=3, seeds 1/2/3
   → Returns 3 WAV files in ./acestep_output/

2. Import all 3 into Audacity (via Audacity MCP)
   → Compare and select the best variant

3. Process selected file in Audacity
   → Normalize, fade, loop trim

4. Export as OGG 44.1kHz for final use
```

## Startup Time

On first tool call, the server boots `acestep-api` and loads the models. Expected cold-start times:

| Hardware | Approximate Load Time |
|---|---|
| Apple M4 Max (64GB, MLX) | ~30–60s |
| RTX 3090 (vllm) | ~45–90s |
| CPU only | 2–5 min |

Subsequent calls in the same session are instant — the server stays running until the MCP client disconnects.

## Troubleshooting

**Server fails to start within timeout**
- Increase `ACESTEP_START_TIMEOUT` to `300`
- Check that `uv run acestep-api` works manually from the repo directory

**`ModuleNotFoundError: No module named 'mcp'`**
- Make sure you're running via `uv run --script acestep_mcp_server.py`, not directly with `python`
- uv auto-installs the inline dependencies on first run

**Port already in use**
- Another process is on port 8001. Change `PORT` in the env block, or kill the conflicting process.

**Audio files not found after generation**
- Check `output_path` is writable
- Default is `./acestep_output/` relative to where Claude Code is running
