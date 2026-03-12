---
name: media
description: Generate images and videos using Google Gemini API (Nano Banana 2 for images, Veo 3 for videos)
argument-hint: <image|video> <prompt> [options]
allowed-tools: Bash, Read, Write, Glob
---

# Media Generation Skill

Generate images and videos using Google's Gemini API models.

## Usage

```
/media image A beautiful sunset over the ocean
/media video A cat playing with a ball of yarn
```

## Models

- **Images**: `gemini-3.1-flash-image-preview` (Nano Banana 2) - fast, high-quality image generation
- **Videos**: `veo-3.0-generate-preview` (Veo 3) - video generation

## How to Process

### Step 1: Parse Arguments

`$ARGUMENTS` contains the full command. The first word is the type (`image` or `video`), the rest is the prompt.

If the user doesn't specify a type, ask them whether they want an image or a video.

### Step 2: Generate

#### For Images

Use curl to call the Gemini API. The API key is available as `$GEMINI_API_KEY` environment variable.

```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-image-preview:generateContent" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H 'Content-Type: application/json' \
  -X POST \
  -d '{
    "contents": [{
      "parts": [{"text": "Generate an image: PROMPT_HERE"}]
    }],
    "generationConfig": {
      "responseModalities": ["image", "text"],
      "imageConfig": {
        "aspectRatio": "16:9"
      }
    }
  }'
```

**Response handling**: The response JSON contains `candidates[0].content.parts[]`. Look for parts with `inlineData` containing `mimeType` (e.g. `image/png`) and `data` (base64-encoded image). Decode and save to a file.

Use this bash pattern to extract and save:

```bash
# Save response to temp file, then extract base64 image data
RESPONSE=$(curl -s ...)
echo "$RESPONSE" | python3 -c "
import json, sys, base64
data = json.load(sys.stdin)
for part in data.get('candidates', [{}])[0].get('content', {}).get('parts', []):
    if 'inlineData' in part:
        img_data = base64.b64decode(part['inlineData']['data'])
        mime = part['inlineData']['mimeType']
        ext = 'png' if 'png' in mime else 'jpg' if 'jpeg' in mime else 'webp'
        with open('OUTPUT_FILENAME.' + ext, 'wb') as f:
            f.write(img_data)
        print(f'Saved: OUTPUT_FILENAME.{ext}')
    elif 'text' in part:
        print(part['text'])
"
```

#### For Videos

Use Veo 3 for video generation. This is an async operation:

**Step 1 - Start generation:**
```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models/veo-3.0-generate-preview:predictLongRunning" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H 'Content-Type: application/json' \
  -X POST \
  -d '{
    "instances": [{
      "prompt": "PROMPT_HERE"
    }],
    "parameters": {
      "sampleCount": 1
    }
  }'
```

If the predictLongRunning endpoint doesn't work, try the generateVideos endpoint:
```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/models/veo-3.0-generate-preview:generateVideos" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -H 'Content-Type: application/json' \
  -X POST \
  -d '{
    "contents": [{
      "parts": [{"text": "PROMPT_HERE"}]
    }]
  }'
```

**Step 2 - Poll for completion** using the operation name from the response:
```bash
curl -s "https://generativelanguage.googleapis.com/v1beta/OPERATION_NAME" \
  -H "x-goog-api-key: $GEMINI_API_KEY"
```

Poll every 10 seconds until `done: true`. Then extract and save the video.

### Step 3: Save Output

- Save files to the **current working directory**
- Use descriptive filenames based on the prompt (e.g., `sunset_ocean.png`)
- Keep filenames short, lowercase, with underscores
- Show the user the saved file path
- If on macOS, offer to open the file with `open <filename>`

### Step 4: Show Result

After saving:
1. Tell the user the file was saved and where
2. Show any text response from the model
3. Open the file automatically on macOS with `open <filename>`

## Aspect Ratio Options

For images, the user can specify aspect ratio. Supported values:
- `1:1` (square)
- `16:9` (landscape, default)
- `9:16` (portrait/mobile)
- `4:3` (standard)
- `3:4` (portrait standard)

If the user mentions "portrait", "vertical", or "mobile" use `9:16`.
If the user mentions "square" use `1:1`.
Otherwise default to `16:9`.

## Error Handling

- If the API returns an error, show the error message to the user
- Common errors: invalid API key, quota exceeded, content policy violation
- If content is blocked by safety filters, inform the user and suggest rephrasing

## Important Notes

- Never hardcode the API key - always use `$GEMINI_API_KEY` from environment
- The image model returns base64-encoded image data inline
- Video generation is async and may take 1-2 minutes
- Always clean up temporary files
