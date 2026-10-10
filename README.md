# English with Hindi Reel Renderer

Design v2 uses a warm cream background, coral/mint/ink lesson panels, larger headings, learning cues, three-step navigation, and gentle zoom/fade motion.

This bundle renders a narrated, three-card, 1080 × 1920 English lesson for Hindi-speaking learners as an H.264/AAC MP4. The teacher's narration is in Hindi (Devanagari); English appears only as the words being taught. An n8n workflow dispatches the render via `workflow_dispatch` and polls the Pages manifest before publishing to Instagram, Facebook and YouTube.

## 1. Upload and enable GitHub Pages

1. Create a **public** GitHub repository with default branch `main`.
2. Copy this folder's contents into the repository root. Include the hidden `.github/workflows/render-reel.yml` file.
3. Open Settings → Pages → Build and deployment → Source → **GitHub Actions**.
4. In Settings → Actions → General, ensure repository policy allows the workflow's write permissions. If deployment needs approval, approve the `github-pages` environment or adjust its protection settings.
5. Open Actions → Render English for Hindi Learners → Run workflow. Enter a fresh request ID, such as `hi-demo-20261010-001`, and the JSON from `sample-lesson.json`.
6. Wait for the render job to succeed. Open `PAGES_BASE_URL/requests/hi-demo-20261010-001.json`. Open its `video_url` and verify that the reel plays with Hindi speech and Devanagari cards (not boxes).

The workflow stores previously rendered files in a generated `gh-pages` branch. Use this repository only for this renderer. It preserves all old videos so that a new render does not remove a video Instagram is still fetching. Periodically remove old files from `gh-pages` once publishing is confirmed.

## 2. Qwen output

Ask Qwen to return only this JSON object, without markdown fences:

```json
{
  "title": "Sound more natural in English",
  "phrase": "I'm on my way.",
  "phrase_hi": "मैं रास्ते में हूँ।",
  "meaning_hi": "जब आप किसी से मिलने निकल चुके हों, तब यह वाक्य बोलें।",
  "example": "Don't worry, I'm on my way.",
  "example_hi": "चिंता मत करो, मैं रास्ते में हूँ।",
  "say_it": "आज एक बहुत काम का अंग्रेज़ी वाक्य सीखो: I'm on my way, यानी मैं रास्ते में हूँ।",
  "get_it": "जब आप किसी से मिलने निकल चुके हों, तो यही बोलें।",
  "use_it": "मेरे साथ एक बार दोहराओ: Don't worry, I'm on my way।",
  "level": "Beginner", "category": "vocabulary", "topic": "on my way",
  "caption": "bilingual caption, max 1800 chars"
}
```

The teacher speaks Hindi (Devanagari script) and teaches English words. `say_it`/`get_it`/`use_it` are the Hindi narration scripts (max 120 chars each) with English words woven in. Limits: title 100; phrase/phrase_hi/example/example_hi 180 each; meaning_hi 300. Hindi text needs a Devanagari font — the workflow installs `fonts-noto-core` and fails loudly if none is found (DejaVu cannot render Hindi). The renderer rejects oversized text and videos above 120 seconds or 90 MiB.

## 3. Dispatch the render (n8n)

HTTP request:

- Method: `POST`
- URL: `https://api.github.com/repos/OWNER/REPO/actions/workflows/render-reel.yml/dispatches`
- `Accept`: `application/vnd.github+json`
- `Authorization`: `Bearer YOUR_TOKEN`
- `X-GitHub-Api-Version`: `2022-11-28`
- `Content-Type`: `application/json`

Use a fine-grained GitHub token scoped to this repository with **Actions: read and write**. `make-dispatch-body.json` shows the shape of the body. `inputs.lesson_json` is a JSON **string**, so the caller must escape it correctly.

Create a fresh request ID once per lesson (prefix `hi-`) and never reuse one. A successful dispatch starts the workflow; its response body is not the video URL.

## 4. Poll the manifest

Read:

`PAGES_BASE_URL/requests/REQUEST_ID.json`

A successful result has this form:

```json
{
  "request_id": "hi-demo-20261010-001",
  "status": "ready",
  "video_url": "https://OWNER.github.io/REPO/reels/hi-demo-20261010-001/reel.mp4",
  "design_version": 2,
  "test_audio": false
}
```

Handle HTTP 404 as "still waiting". Require matching request ID, `status = ready`, `test_audio = false` and `design_version = 2` before publishing. Stop waiting after ~20 minutes, mark failed, and inspect the matching Actions run. A missing JSON is never permission to publish an earlier video.

For the Instagram Reel, map only the returned `video_url` and use `cover_url` as the thumbnail source. Confirm the URL is publicly fetchable before the first real publish.

## 5. Local checks

Requires Python 3.12, FFmpeg/ffprobe and a Devanagari font (e.g. `fonts-noto-core`, or set `REEL_FONT` to a suitable TTF).

```sh
pip install -r requirements.txt
python render_reel.py --lesson sample-lesson.json --request-id local-demo-001 --base-url https://example.com --public public
```

`edge-tts` uses Microsoft's online speech service via an unofficial client (voice `hi-IN-SwaraNeural`). It does not require an API key, but needs internet access and can fail if the service changes. The renderer retries three times and fails the job if narration fails.

For offline media verification only, add `--test-audio`. This generates tones instead of speech and marks the manifest `test_audio: true`; never publish such a result.

## Included files

- `.github/workflows/render-reel.yml`: dispatch, render, preserve and deploy
- `render_reel.py`: Hindi narration, Devanagari card rendering, encoding and validation
- `requirements.txt`: Python dependencies
- `sample-lesson.json`: lesson schema/example
- `make-dispatch-body.json`: example GitHub dispatch HTTP body
- `README.md`: setup and integration

No GitHub credentials, tokens or Instagram secrets are included. Live verification requires your actual repository and n8n workflow.
