# Board Ingest Webhook — Hopscotch Test Payloads

**Webhook URL:** `https://workflows.engineeryourbusiness.com/webhook/ingestboardrecording`  
**Method:** POST  
**Content-Type:** application/json

---

## Path 1: Text Transcript URL

Use this when you have a plain-text transcript file hosted at a URL
(e.g. in Supabase Storage, Google Drive direct-download, S3, etc.).

```json
{
  "type": "transcript",
  "title": "Board Meeting – May 2025 Test",
  "category": "Board Meeting",
  "recorded_date": "2025-05-07",
  "filename": "board_may_2025_test.txt",
  "transcript_url": "https://YOUR_SUPABASE_PROJECT.supabase.co/storage/v1/object/public/transcripts/board_may_2025_test.txt"
}
```

**What happens:**
1. Parse Input validates `transcript_url` is present
2. Route by Type → FALSE branch (not a recording)
3. Fetch Transcript from URL → GET request to `transcript_url`
4. Merge Transcript Text → combines fetched text with metadata
5. Create Recording in Supabase → returns `recording_id`
6. Create Transcript in Supabase → stores `full_text`, returns `transcript_id`
7. Execute AI Processor → generates KCS articles

**Quick test with a public text file:**
```json
{
  "type": "transcript",
  "title": "Hopscotch Text URL Test",
  "category": "Test",
  "recorded_date": "2025-05-18",
  "filename": "hopscotch_test_transcript.txt",
  "transcript_url": "https://raw.githubusercontent.com/pparikh73/zendesk-metrics-sync/main/README.md"
}
```
*(Uses the README as a dummy transcript — won't produce great articles but confirms the path works end-to-end)*

---

## Path 2: Video/Audio URL

Use this when you have a direct URL to an MP4, M4A, or other audio/video file
that AssemblyAI can download (must be publicly accessible or a signed URL).

```json
{
  "type": "recording",
  "title": "Board Meeting – May 2025 Video Test",
  "category": "Board Meeting",
  "recorded_date": "2025-05-07",
  "filename": "board_may_2025_video.mp4",
  "file_url": "https://YOUR_STORAGE_BUCKET.supabase.co/storage/v1/object/public/recordings/board_may_2025_video.mp4"
}
```

**What happens:**
1. Parse Input validates `file_url` is present
2. Route by Type → TRUE branch (is a recording)
3. Create Recording in Supabase → returns `recording_id`
4. Submit to AssemblyAI → POSTs `{ audio_url: file_url, ... }` with webhook callback
5. Wait for AssemblyAI → execution pauses until AssemblyAI POSTs back (minutes to hours)
6. Fetch AssemblyAI Result → GETs completed transcript
7. Create Transcript in Supabase
8. Execute AI Processor → generates KCS articles

**Note:** For the video path, Hopscotch will get a quick 200 response from the Webhook node
while n8n continues processing asynchronously via the Wait node.

---

## Error Cases to Verify

**Missing file_url for recording type:**
```json
{
  "type": "recording",
  "title": "Bad Recording Test",
  "category": "Test",
  "recorded_date": "2025-05-18",
  "filename": "bad_test.mp4"
}
```
Expected: Parse Input throws `type=recording requires file_url`

**Missing transcript_url for transcript type:**
```json
{
  "type": "transcript",
  "title": "Bad Transcript Test",
  "category": "Test",
  "recorded_date": "2025-05-18",
  "filename": "bad_test.txt"
}
```
Expected: Parse Input throws `type=transcript requires transcript_url`

---

## Supabase Storage — Getting a Public URL for Testing

If you upload a transcript `.txt` file to Supabase Storage:
1. Go to Storage → your bucket → upload the file
2. Click the file → Copy URL
3. If the bucket is public, the URL works directly
4. If private, generate a signed URL (valid for N seconds)

For AssemblyAI video URLs, the file must be downloadable without auth headers.
Use a public bucket or a signed URL with enough time for transcription to complete.
