---
name: media
description: Generate and edit images using the OpenAI Images API (gpt-image-2.6-flare by default, gpt-image-2.6-sunburst for high quality)
argument-hint: <image|edit> [fast|hq] [portrait|square|landscape] <prompt | file... prompt>
allowed-tools: Bash, Read, Write, Glob
---

# Media Generation Skill (OpenAI)

Generate new images or edit existing ones with OpenAI's image models. OpenAI only, never call Google or any other provider.

## Usage

```
/media image A beautiful sunset over the ocean
/media image hq portrait a woman in traditional clothing
/media edit photo.png make the sky purple
/media edit cat.png hat.png put the hat on the cat
```

## Models

| Mode | Model | Speed |
|------|-------|-------|
| default | `gpt-image-2.6-flare` | ~7s, fast |
| `hq` (also "high quality", "best", "sunburst") | `gpt-image-2.6-sunburst` | ~25s, higher quality |

The API key is in the `$OPENAI_API_KEY` environment variable. Never print it or hardcode it.

## How to Process

### Step 1: Parse Arguments

`$ARGUMENTS` holds the full command.

- First word is the action: `image` (generate) or `edit`. If it is missing, but the arguments mention an existing image file, treat it as `edit`. Otherwise treat it as `image`.
- Optional modifiers anywhere before the prompt:
  - `hq` / `high quality` / `best` → model `gpt-image-2.6-sunburst`, quality `high`
  - `fast` → model `gpt-image-2.6-flare` (already the default)
  - Otherwise: model `gpt-image-2.6-flare`, quality `medium`
- Size:
  - `portrait`, `vertical`, `mobile`, `9:16`, `3:4` → `1024x1536`
  - `square`, `1:1`, `logo`, `avatar`, `icon` → `1024x1024`
  - `landscape`, `16:9`, `4:3`, or nothing specified → `1536x1024`
  - For `edit` with nothing specified → `auto` (keeps the input's shape)
- For `edit`: the words that are existing file paths (check with Glob/ls) are the input images (up to 16; png, jpg or webp). The rest is the prompt. If there is no input file, ask the user which image to edit.

If the prompt is empty, ask the user what they want.

### Step 2: Call the API

Write the response to a temp file (base64 output is large, so don't keep it in a shell variable). Build JSON with python so quotes in the prompt are escaped safely.

#### Generate (`image`)

```bash
TMP=$(mktemp -t media)
python3 -c 'import json,sys; print(json.dumps({"model":sys.argv[1],"prompt":sys.argv[2],"size":sys.argv[3],"quality":sys.argv[4]}))' \
  "MODEL" "PROMPT" "SIZE" "QUALITY" \
| curl -s https://api.openai.com/v1/images/generations \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- -o "$TMP"
```

#### Edit (`edit`)

Multipart form. Repeat `-F "image[]=@FILE"` for each input image:

```bash
TMP=$(mktemp -t media)
curl -s https://api.openai.com/v1/images/edits \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F model="MODEL" \
  -F "image[]=@INPUT_FILE" \
  -F prompt="PROMPT" \
  -F size="SIZE" \
  -F quality="QUALITY" \
  -o "$TMP"
```

(Escape any double quotes in PROMPT for the `-F prompt=` value.)

### Step 3: Save Output

Pick a short, lowercase, underscore filename based on the prompt (e.g. `sunset_ocean`). Save in the **current working directory**. Don't overwrite: if the file exists, add `_2`, `_3`, and so on.

```bash
python3 - "$TMP" "OUTPUT_BASENAME" <<'EOF'
import json, sys, base64, os
path, base = sys.argv[1], sys.argv[2]
d = json.load(open(path))
if "error" in d:
    print("ERROR:", d["error"].get("message"), f"({d['error'].get('code')})"); sys.exit(1)
ext = d.get("output_format", "png")
for i, item in enumerate(d["data"]):
    name = f"{base}{'' if i == 0 else f'_{i+1}'}.{ext}"
    n = 2
    while os.path.exists(name):
        name = f"{base}_{n}.{ext}"; n += 1
    open(name, "wb").write(base64.b64decode(item["b64_json"]))
    print("Saved:", os.path.abspath(name))
    if item.get("revised_prompt"): print("Revised prompt:", item["revised_prompt"])
EOF
rm -f "$TMP"
```

### Step 4: Show Result

1. Tell the user the full path of the saved file, plus the model and size used.
2. On macOS, open it with `open <filename>`.

## Error Handling

- `OPENAI_API_KEY` empty → tell the user to set it (`export OPENAI_API_KEY=...` in `~/.zshrc`).
- `invalid_api_key` / 401 → the key is wrong or revoked.
- "model ... does not exist" (`invalid_value` / `model_not_found`) → the account can't use that model. Run `curl -s https://api.openai.com/v1/models -H "Authorization: Bearer $OPENAI_API_KEY"` and pick the newest `gpt-image-*` model that is available.
- `moderation_blocked` / safety rejection → say the prompt was blocked and suggest rephrasing.
- `rate_limit_exceeded` / `insufficient_quota` → show the message. Don't retry in a loop.

## Notes

- This skill supports images only. Video is not supported yet.
- Always remove temp files.
