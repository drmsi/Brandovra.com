# n8n Workflow Guide — Push Products to Brandovra

## Overview

An n8n cron workflow fetches product data (from a Google Sheet, Airtable, or manual trigger) and pushes a JSON file to the `/products/` folder in this repo. The homepage reads new files automatically via the GitHub Contents API.

---

## Workflow Nodes

### 1. Schedule Trigger
- Type: **Cron**
- Interval: e.g. every hour, or trigger manually

### 2. (Optional) Data Source
- **Google Sheets** node — read a row of product data, or
- **Set** node — hardcode a product for testing

### 3. GitHub Node — Create/Update File

| Field | Value |
|---|---|
| **Operation** | Create or Update |
| **Repository Owner** | `drmsi` |
| **Repository Name** | `Brandovra.com` |
| **File Path** | `products/{{ $now.toMillis() }}.json` |
| **File Content** | *(see below)* |
| **Commit Message** | `Add product: {{ $json.title }}` |
| **Branch** | `main` |

#### File Content (JSON, base64 encoded by n8n automatically)

```json
{
  "title": "{{ $json.title }}",
  "brand": "{{ $json.brand }}",
  "price": "{{ $json.price }}",
  "category": "{{ $json.category }}",
  "country": "{{ $json.country }}",
  "media_type": "{{ $json.media_type }}",
  "drive_file_id": "{{ $json.drive_file_id }}",
  "affiliate_url": "{{ $json.affiliate_url }}",
  "description": "{{ $json.description }}"
}
```

> **Note:** In the GitHub node, set the content field to the raw JSON string above (n8n will base64-encode it). Use the **Expression** toggle on the Content field.

---

## Product Schema

```json
{
  "title": "Silk Evening Dress",
  "brand": "Maison Luxe",
  "price": "KWD 149",
  "category": "fashion",
  "country": "kw",
  "media_type": "image",
  "drive_file_id": "1abc...xyz",
  "affiliate_url": "https://amzn.to/...",
  "description": "Pure silk, hand-stitched evening gown."
}
```

**`country`**: ISO 3166-1 alpha-2 country code (lowercase). Controls which country page the product appears on.

Supported codes and their URLs:

| Code | Country | URL |
|---|---|---|
| `kw` | Kuwait | brandovra.com/kw |
| `ae` | UAE | brandovra.com/ae |
| `sa` | Saudi Arabia | brandovra.com/sa |
| `bh` | Bahrain | brandovra.com/bh |
| `qa` | Qatar | brandovra.com/qa |
| `om` | Oman | brandovra.com/om |
| `jo` | Jordan | brandovra.com/jo |
| `eg` | Egypt | brandovra.com/eg |
| `us` | United States | brandovra.com/us |
| `gb` | United Kingdom | brandovra.com/gb |

**`media_type`**: `"image"` or `"video"`

**`drive_file_id`**: The ID portion of a Google Drive share URL:
- `https://drive.google.com/file/d/`**`1abc...xyz`**`/view`

**Category values** (used for filter buttons on homepage):
`fashion` | `jewelry` | `bags` | `footwear` | `beauty` | `watches`

---

## Google Drive Media URLs

| Type | URL pattern |
|---|---|
| Image (direct) | `https://lh3.googleusercontent.com/d/{DRIVE_FILE_ID}` |
| Video (embed) | `https://drive.google.com/file/d/{DRIVE_FILE_ID}/preview` |

Make sure the Drive file is shared as **"Anyone with the link — Viewer"**.

---

## Testing

1. Run the workflow once manually in n8n
2. Check the repo — a new file like `1741786400789.json` should appear in `/products/`
3. Visit brandovra.com — the new product should appear at the top of the grid (newest-first)

---

## GitHub Credentials in n8n

1. In n8n, go to **Credentials → New → GitHub**
2. Use a **Personal Access Token** with `repo` scope (Contents write)
3. Reference that credential in the GitHub node
