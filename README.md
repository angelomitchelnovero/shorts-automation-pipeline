# Shorts Automation Pipeline

An automated content pipeline that generates short-form video scripts with AI, converts them to voiceover, pairs them with stock footage, assembles a finished video, and uploads it to YouTube — all orchestrated through Make.com on free-tier services.

## Pipeline Overview

```
Schedule (trigger)
   ↓
Google Gemini AI → generates video script
   ↓
Google Cloud Text-to-Speech → converts script to voiceover (base64 audio)
   ↓
Google Drive (Upload) → saves voiceover as MP3 (decoded from base64)
   ↓
Google Drive (Get a Share Link) → makes the MP3 publicly fetchable
   ↓
Pexels API → searches and returns a matching stock video clip
   ↓
Shotstack (Make an API Call) → submits a render job combining video + audio
   ↓
Sleep (40s) → waits for the render to actually finish
   ↓
Shotstack (Get Render Status) → retrieves the finished video's URL
   ↓
HTTP (GET) → downloads the finished video as binary
   ↓
YouTube (Upload a Video) → publishes the final video
```

## Services Used (All Free Tier)

| Service | Purpose | Free Tier Limit |
|---|---|---|
| Make.com | Workflow orchestration | 1,000 operations/month |
| Google Gemini AI (2.5 Flash) | Script generation | Generous free tier |
| Google Cloud Text-to-Speech | Voiceover (Neural2 voices) | 1,000,000 characters/month |
| Google Drive | File staging/storage | 15 GB total (shared with Gmail) |
| Pexels API | Stock video clips | No hard cap for reasonable use |
| Shotstack (Sandbox) | Video rendering/assembly | ~20-50 render minutes/month |
| YouTube Data API v3 | Video publishing | 10,000 units/day (~6 uploads/day) |

**Known bottleneck:** Shotstack's free render-minute cap is the tightest constraint. At 3x/day, 5 days/week (~65 videos/month, ~30-40 sec each), you'll use roughly 32-43 render minutes/month — near or over the free limit depending on Shotstack's current plan. Watch this first if scaling up.

---

## Setup Steps

### 1. Google Cloud Project (shared by YouTube, Gemini, TTS)

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create a new project (e.g. "shorts-automation").
2. Enable these APIs under **APIs & Services → Library**:
   - YouTube Data API v3
   - Cloud Text-to-Speech API
3. Under **APIs & Services → OAuth consent screen (Google Auth Platform)**:
   - Choose **External** audience.
   - Add your own Google account as a **Test User** (keeps you out of Google's review process).
4. Get credentials:
   - **API Key** (for Gemini and Text-to-Speech): APIs & Services → Credentials → Create Credentials → API Key.
   - **OAuth Client** (for YouTube): only needed if Make's built-in YouTube connection doesn't work automatically — Make often has its own pre-registered app, so try connecting directly in Make first.

### 2. Make.com Account

1. Sign up free at [make.com](https://make.com).
2. Create a new scenario (e.g. "Shorts Automation Pipeline").
3. Set the scenario's run schedule via the clock icon at the bottom-left of the canvas (e.g. daily, or 3x/day on weekdays only).

### 3. Gemini AI Module (Script Generation)

1. Add a **Google Gemini AI → Generate a Response** module.
2. Connect using your Google Cloud API key.
3. **System Instructions:** define the permanent style rules (tone, length, no stage directions — plain spoken narration only, since the full text gets read aloud by TTS).
4. **Messages → Item 1 → Role:** `User`. **Parts → Text:** the actual topic/request for this run.
5. Model: `gemini-2.5-flash` (free-tier friendly).

> ⚠️ **Important:** Keep the prompt asking for plain spoken text only — no "Visual:"/"Audio:" labels or formatting, since everything Gemini outputs gets fed directly into the voiceover.

### 4. Google Cloud Text-to-Speech (Voiceover)

Make has no native TTS module, so this uses a raw **HTTP → Make a request** module:

- **URL:** `https://texttospeech.googleapis.com/v1/text:synthesize?key=YOUR_API_KEY`
- **Method:** POST
- **Headers:** `Content-Type: application/json`
- **Body content type:** `application/json`
- **Body input method:** `Data structure` (avoids manual JSON-escaping issues with line breaks in the script)
- **Body fields:**
  - `input.text` → mapped to Gemini's `Result`
  - `voice.languageCode` → `en-US`
  - `voice.name` → `en-US-Neural2-D` (or any Neural2 voice)
  - `audioConfig.audioEncoding` → `MP3`

Response returns a base64-encoded `audioContent` field — this is text, not binary, so it needs decoding in the next step.

### 5. Google Drive — Upload Voiceover

1. Add **Google Drive → Upload a File**.
2. **File Name:** e.g. `voiceover.mp3`
3. **Data field (critical):**
   ```
   toBinary(audioContent; base64)
   ```
   This decodes the base64 text into actual binary audio. Without the `base64` parameter (unquoted — quoting it causes a "not a valid encoding" error), the file will be corrupted/unplayable.

### 6. Google Drive — Get a Share Link

1. Add **Google Drive → Get a Share Link**.
2. **File ID:** mapped from the Upload module's `File ID` output.
3. **Role:** `Reader`. **Type:** `Anyone`.
4. Use the resulting **`webContentLink`** field (format: `.../uc?id=...&export=download`) downstream — NOT the `Share Link`/`Web View Link` field, which returns an HTML viewer page instead of the raw file and will break any service trying to fetch it directly.

### 7. Pexels — Stock Video Search

Raw **HTTP → Make a request** module:

- **Authentication:** API Key, sent as an `Authorization` header (your Pexels API key, no "Bearer" prefix).
- **URL:** `https://api.pexels.com/videos/search?query=YOUR_QUERY&per_page=3`
- **Method:** GET

Response returns a `videos[]` array, each with a `video_files[]` array of different quality/resolution options. Pick the vertical HD one (typically `1080x1920`) by its array index — check the actual response each time since index position can vary between results.

### 8. Shotstack — Render the Video

Raw **Shotstack → Make an API Call** module (not "Render from Template" — that requires a pre-built template):

- **URL:** `edit/stage/render` (sandbox/free-tier endpoint — NOT `edit/v1/render`, which is paid production)
- **Method:** POST
- **Headers:** `Content-Type: application/json`
- **Body:**
  ```json
  {
    "timeline": {
      "background": "#000000",
      "tracks": [
        {
          "clips": [
            {
              "asset": { "type": "video", "src": "PEXELS_HD_VIDEO_LINK" },
              "start": 0,
              "length": 31
            }
          ]
        },
        {
          "clips": [
            {
              "asset": { "type": "audio", "src": "DRIVE_WEB_CONTENT_LINK" },
              "start": 0,
              "length": 31
            }
          ]
        }
      ]
    },
    "output": { "format": "mp4", "resolution": "1080" }
  }
  ```
  Both `src` values must be mapped as live tokens (via Make's field-click mapping panel), not typed as plain text — and both URLs must be genuinely public/direct-fetchable links, not view-only pages.

### 9. Sleep (Delay)

Add **Tools → Sleep**, set to ~40 seconds. Shotstack renders are asynchronous — checking status immediately after submission will always show "fetching" since the render hasn't had time to actually run yet.

### 10. Shotstack — Get Render Status

- **Render ID:** mapped from the previous module's `response.id` output.
- Once `Status` = `done`, use the **`Private URL`** field (valid 24 hours) as the finished video's download link.

### 11. HTTP — Download the Finished Video

- **URL:** mapped from the `Private URL` field above.
- **Method:** GET
- **Parse response:** No (this is binary video data, not JSON).

### 12. YouTube — Upload a Video

1. Add **YouTube → Upload a Video**, connect using Make's built-in connection (usually auto-available, no manual OAuth client needed).
2. **Title / Description:** hardcode for testing, or map from Gemini's output.
3. **File Name:** e.g. `final_video.mp4`
4. **Data:** mapped from the HTTP download module's output.
5. **Video Category:** e.g. Film & Animation, or whatever fits your content.
6. **Privacy Status:** set to **Private** or **Unlisted** while testing — switch to Public only once you're confident in the output.

---

## Running the Pipeline

- **Test a single run:** click **"Run once"** on the main canvas (not inside an individual module — that only tests one step in isolation and will show empty/broken mappings for anything that depends on earlier steps).
- **Automatic runs:** set the scenario's schedule (clock icon, bottom-left of canvas) to your desired frequency (e.g. 3x/day, weekdays only).
- To temporarily skip a module during testing (e.g. YouTube while other steps aren't ready), right-click it and choose **"Turn off module."**

## Known Issues & Fixes Log

| Issue | Cause | Fix |
|---|---|---|
| ElevenLabs quota exceeded quickly | Free tier only 10,000 characters/month — too low for 3x/day | Switched to Google Cloud TTS (1M free chars/month) |
| JSON parse error in TTS request | Script text contained raw line breaks, breaking manually-typed JSON | Switched Body input method to "Data structure" (auto-escapes) |
| Audio file wouldn't play | `toBinary()` used without specifying source encoding | Added `; base64` parameter: `toBinary(audioContent; base64)` |
| Shotstack stuck "fetching" (audio) | Audio URL pointed to Drive's view-only link, not a direct download link | Used `webContentLink` field instead of `Share Link` |
| Shotstack stuck "fetching" (video) | Missing array index in Pexels mapping (`video_files[]` instead of `video_files[5]`) | Mapped to a specific array index |
| Shotstack showed "fetching" even after finishing | Make checked render status immediately after submission, before the ~31s render actually completed | Added a 40-second Sleep module before the status check |
| TTS reading "Visual:"/"Audio:" labels aloud | Gemini's script included screenplay-style formatting, and the full raw output was fed into TTS | Prompt instructed to output plain spoken narration only, no labels/stage directions |

## Future Improvements (Not Yet Built)

- **Dynamic topics:** replace the hardcoded script topic with a rotating list or Google Sheet, so content varies daily instead of repeating similar results.
- **Smarter visual matching:** use a separate short Gemini call to generate a targeted search keyword for Pexels based on the actual script topic, rather than a fixed query term.
- **Multi-platform publishing:** extend the same rendered video to TikTok (Content Posting API — requires app approval) and Facebook/Instagram Reels (Graph API — requires Business account + app review).
- **Regenerate the Google TTS API key** used during setup, since it was shared in plaintext during development.
