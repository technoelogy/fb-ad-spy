# FB Ad Spy

Automated competitor ad intelligence for Consumer Health, OTC, and DTC brands.

An n8n workflow that watches every ad running on Facebook/Instagram for a given search phrase, uses AI to describe what happens in each one (frame-by-frame for videos), and logs results to a Google Sheet.

Built as a weekly intel pipeline. 32 ads tracked in the first production run for the "protein shake" category in India.

---

## Who this is for

Marketers, strategists, agency planners, and brand founders working in categories where creative-level competitive intel actually moves decisions. If you've ever built the "competitive section" of a brief by scrolling the FB Ad Library and taking screenshots, this replaces that work.

## What you get in the output sheet

Each row = one ad. Columns:

| Column | What it is |
|---|---|
| `ad_archive_id` | Facebook's unique ad ID |
| `page_id` | Facebook's page ID for the advertising brand |
| `type` | video / image / text |
| `date_added` | Timestamp when scraped |
| `page_name` | Brand name |
| `page_url` | Link to brand's FB page |
| `summary` | AI-written strategic analysis of what the ad is doing |
| `rewritten_ad_copy` | Alternate draft of the ad copy in a neutral voice |
| `image_prompt` | Text description of what's in the image (for image ads) |
| `video_prompt` | Frame-by-frame description with timestamps (for video ads) |

The `video_prompt` column turns a 60-second ad into text you can scan in 10 seconds. That's the main unlock.

---

## Architecture

```
FB Ad Library (via Apify)
        ↓
Filter by page likes (>100)
        ↓
Switch node routes by format
        ↓
┌───────┬───────┬───────┐
│ Video │ Image │ Text  │
└───┬───┴───┬───┴───┬───┘
    │       │       │
Download  Fetch   (direct)
    ↓       ↓       ↓
Gemini   OpenAI  OpenAI
Video    Vision  Text
    ↓       ↓       ↓
OpenAI   OpenAI  OpenAI
GPT-4.1  GPT-4.1 GPT-4.1
    ↓       ↓       ↓
    └───→ Google Sheet ←───┘
```

---

## Setup

### 1. Import workflow
Import `fb_ad_spy_workflow.json` into your n8n instance.

### 2. Create Google Sheet
Create a new Google Sheet. Add these headers in row 1 (A1 through J1):

```
ad_archive_id | page_id | type | date_added | page_name | page_url | summary | rewritten_ad_copy | image_prompt | video_prompt
```

Rename default tab to `Ads`. Copy the sheet ID from the URL.

### 3. Wire 5 credentials in n8n

| Credential | Type | Where used |
|---|---|---|
| Apify Bearer | Header Auth, value: `Bearer YOUR_APIFY_TOKEN` | Run Ad Library Scraper |
| Gemini API | Query param, direct key paste | 3 Gemini HTTP nodes |
| Google Drive | OAuth2 | Upload Video to Drive, Redownload Video |
| Google Sheets | OAuth2 | 3 "Add as Type" nodes |
| OpenAI | API Key | 4 OpenAI nodes |

### 4. Paste sheet ID
In each of the 3 "Add as Type" nodes (Video, Image, Text), replace `YOUR_SHEET_ID` with your actual Google Sheet ID.

### 5. Localise the search
In "Run Ad Library Scraper" node's jsonBody, change the URL params:
- `country=IN` → your target country (US, UK, AU etc.)
- `q=%22protein%20shake%22` → your search phrase (URL-encoded)
- `count=10` → adjust batch size
- `period=last7d` → adjust lookback window

### 6. Image branch note
Facebook's CDN blocks OpenAI's server from fetching image URLs directly. The workflow uses an HTTP Request node between Loop Over Image Ads and Analyze Image to download the image with a browser User-Agent spoof, then passes it as Binary File to OpenAI Vision.

If you add new image-handling logic, keep this pattern.

### 7. Trigger
Default is manual trigger. For production, add a Schedule Trigger node (weekly Monday 9am recommended) and wire it to Run Ad Library Scraper.

---

## Cost estimate (per 10-ad run)

| Service | Cost |
|---|---|
| Apify FB Ad Library actor | ~$0.10 |
| Gemini video analysis | Free tier covers 20/day; paid is ~$0.02/video |
| OpenAI GPT-4o + GPT-4.1 | ~$0.30 |
| Google Drive/Sheets | Free |
| **Total per run** | **~$0.50** |

---

## Known limitations

- Facebook CDN image URLs are signed and expire. The HTTP Request download step handles this, but if the User-Agent header isn't properly set, images will 403.
- Gemini free tier is 20 requests/day. For weekly runs with 10+ videos, upgrade to paid tier or swap video analysis to OpenAI GPT-4o Vision.
- `rewritten_ad_copy` column is useful as a creative starter but not a finished asset. Review before using in production creative.
- No automatic deduplication across runs. Same ad will create a new row each week. Easy filter in Google Sheets.

---

## Credits

Built on top of a community n8n template structure. Significant adaptations:
- Localized to Indian market (country, search phrases, likes threshold)
- Added HTTP Request + User-Agent spoof for FB CDN image bypass
- Updated Gemini model to current version
- Added error isolation (On Error: Continue) so one branch failure doesn't kill parallel branches
- Credential rewiring and OAuth management

---

## License

MIT — see LICENSE file.

---

## Author

Noel Gomes (Tron) — Digital marketer and AI builder, focused on Consumer Health, OTC, DTC, and pharma marketing in India.

- LinkedIn: [gomesnoel](https://www.linkedin.com/in/gomesnoel/)
- GitHub: [technoelogy](https://github.com/technoelogy)
- Site: [technoelogy.com](https://technoelogy.com)

Open to collaboration on AI automation builds for Consumer Health, OTC, or healthcare brands.
