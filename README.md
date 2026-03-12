# /media - Claude Code Skill for AI Image & Video Generation

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) custom skill that generates images and videos directly from your terminal using Google's Gemini API.

Type `/media` in Claude Code and describe what you want to create.

## What It Does

- **Image Generation** via `gemini-3.1-flash-image-preview` (Nano Banana 2) - Google's high-efficiency image generation model
- **Video Generation** via `veo-3.0-generate-preview` (Veo 3) - Google's video generation model

## Usage

```
/media image a futuristic city at sunset
/media video ocean waves crashing on rocks
/media image portrait a woman in traditional clothing
/media image square a minimalist logo design
```

### Aspect Ratios (Images)

The skill auto-detects aspect ratio from your prompt:

| Keyword | Ratio | Use Case |
|---------|-------|----------|
| _(default)_ | 16:9 | Landscape |
| `portrait`, `vertical`, `mobile` | 9:16 | Phone wallpapers, stories |
| `square` | 1:1 | Social media, avatars |
| `4:3` | 4:3 | Standard photos |

## Installation

### 1. Get a Gemini API Key

Go to [Google AI Studio](https://aistudio.google.com/apikey) and create a free API key.

### 2. Set the Environment Variable

Add to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):

```bash
export GEMINI_API_KEY="your-api-key-here"
```

Then reload: `source ~/.zshrc`

### 3. Install the Skill

**Option A: Personal skill** (available in all your projects)

```bash
mkdir -p ~/.claude/skills/media
curl -o ~/.claude/skills/media/SKILL.md \
  https://raw.githubusercontent.com/bulgariamitko/cc-media/main/.claude/skills/media/SKILL.md
```

**Option B: Project skill** (available only in a specific project)

```bash
cd your-project
mkdir -p .claude/skills/media
curl -o .claude/skills/media/SKILL.md \
  https://raw.githubusercontent.com/bulgariamitko/cc-media/main/.claude/skills/media/SKILL.md
```

### 4. Verify

Open Claude Code and type `/media` - it should appear in the skill autocomplete.

## How It Works

1. You run `/media image <prompt>` or `/media video <prompt>`
2. Claude calls the Gemini API via REST (curl)
3. The response (base64-encoded image or video) is decoded and saved to your current directory
4. The file opens automatically on macOS

No dependencies required beyond `curl` and `python3` (both pre-installed on macOS/Linux).

## Models

| Model | ID | Use |
|-------|----|-----|
| **Nano Banana 2** | `gemini-3.1-flash-image-preview` | Fast, high-quality image generation |
| **Veo 3** | `veo-3.0-generate-preview` | Video generation (async, takes ~1-2 min) |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- A [Google Gemini API key](https://aistudio.google.com/apikey) (free tier available)
- `curl` and `python3` (pre-installed on macOS/Linux)

## Example Output

Prompt: `/media image bulgarian revolutional woman from 1888 looking at the camera`

<img src="example.png" width="600" alt="Example generated image">

## License

MIT
