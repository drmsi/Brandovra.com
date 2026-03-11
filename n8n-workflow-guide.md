# n8n Workflow Guide — Push Products to Brandovra

## Overview

An n8n workflow pushes product JSON files to the `/products/` folder in the GitHub repo.
The homepage fetches all files via the GitHub raw API, sorts them newest-first by filename (Unix timestamp ms), and renders them automatically.

---

## Workflow Structure

```
Manual Trigger ──┐
                 ├──▶ Product Data (Set) ──▶ Build Payload (Code) ──▶ Push to GitHub (HTTP)
Schedule Trigger─┘
```

---

## Node 1 — Triggers

Use **both** triggers wired into a **Merge** node:

- **Manual Trigger** — run ad-hoc when adding a single product
- **Schedule Trigger** — e.g. every hour to auto-pull from a Google Sheet

---

## Node 2 — Product Data (Set node)

Set these fields manually or map them from a Google Sheet / Airtable row:

| Field | Type | Required | Notes |
|---|---|---|---|
| `title` | string | ✅ | Product name |
| `brand` | string | ✅ | Brand name |
| `price` | string | ✅ | Include currency symbol e.g. `KWD 49` / `$89` / `AED 199` |
| `category` | string | ✅ | Any value — auto-appears as filter button on site |
| `country` | string | ⬜ | 2-letter ISO code (see table below). **Leave empty to show on all country pages** |
| `media_type` | string | ✅ | `image` or `video` |
| `drive_file_id` | string | ⬜ | Google Drive file ID (used if no `image_url`) |
| `image_url` | string | ⬜ | Direct image URL — overrides `drive_file_id` if set |
| `affiliate_url` | string | ✅ | Product link. Amazon links get `?tag=brandovra-20` added automatically by the site |
| `description` | string | ⬜ | Short product description |

---

## Node 3 — Build Payload (Code node)

Generates the filename (`{timestamp}.json`) and base64-encodes the content for the GitHub API:

```javascript
const p = $input.item.json;
const now = Date.now();

const product = {
  title:         (p.title         || '').trim(),
  brand:         (p.brand         || '').trim(),
  price:         (p.price         || '').trim(),
  category:      (p.category      || '').toLowerCase().trim(),
  country:       (p.country       || '').toLowerCase().trim() || null,
  media_type:    (p.media_type    || 'image').trim(),
  drive_file_id: (p.drive_file_id || '').trim(),
  image_url:     (p.image_url     || '').trim(),
  affiliate_url: (p.affiliate_url || '').trim(),
  description:   (p.description   || '').trim()
};

// Drop empty optional fields
if (!product.country)       delete product.country;
if (!product.drive_file_id) delete product.drive_file_id;
if (!product.image_url)     delete product.image_url;

const jsonStr = JSON.stringify(product, null, 2);
const content = Buffer.from(jsonStr).toString('base64');

return [{
  json: {
    filename: `${now}.json`,
    message:  `Add product: ${product.title}`,
    content
  }
}];
```

---

## Node 4 — Push to GitHub (HTTP Request node)

| Setting | Value |
|---|---|
| **Method** | `PUT` |
| **URL** | `https://api.github.com/repos/drmsi/Brandovra.com/contents/products/{{ $json.filename }}` |
| **Auth header** | `Authorization: token {GITHUB_PAT}` |
| **Accept header** | `application/vnd.github+json` |
| **Body (JSON)** | `{ "message": "{{$json.message}}", "content": "{{$json.content}}", "branch": "main" }` |

GitHub Personal Access Token needs `repo` scope (Contents: write).

---

## Product Schema Reference

```json
{
  "title": "Silk Evening Gown",
  "brand": "Maison Luxe",
  "price": "KWD 149",
  "category": "fashion",
  "country": "kw",
  "media_type": "image",
  "drive_file_id": "1abc...xyz",
  "image_url": "https://images.unsplash.com/photo-xxx?w=400&h=711&fit=crop",
  "affiliate_url": "https://www.amazon.com/dp/B0XXXXXXXXX",
  "description": "Pure silk, hand-stitched evening gown."
}
```

### `country` — Country Routing

Products with a `country` code appear **only on that country page**.
Products with **no `country` field** appear on **all pages including the homepage**.

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
| `fr` | France | brandovra.com/fr |
| `de` | Germany | brandovra.com/de |
| `jp` | Japan | brandovra.com/jp |
| `au` | Australia | brandovra.com/au |
| `ca` | Canada | brandovra.com/ca |

### `category` — Fully Dynamic

Category filter buttons on the site are **auto-generated** from whatever values exist in loaded products — no code change needed. Use any string:

```
fashion  jewelry  bags  footwear  beauty  watches
electronics  home  skincare  fragrance  accessories  ...
```

### `media_type`

| Value | Behaviour |
|---|---|
| `image` | Renders `<img>` from `image_url` or `drive_file_id` |
| `video` | Renders Google Drive `<iframe>` preview from `drive_file_id` |

### `image_url` vs `drive_file_id`

- `image_url` takes priority if set
- If only `drive_file_id` is set, image URL becomes `https://lh3.googleusercontent.com/d/{ID}`
- For video, uses `https://drive.google.com/file/d/{ID}/preview`
- Make Drive files public: **Share → Anyone with the link → Viewer**

### `affiliate_url` — Amazon Auto-Tagging

The site automatically appends `?tag=brandovra-20` to any Amazon URL. Just use the plain product URL:

```
https://www.amazon.com/dp/B0XXXXXXXXX
https://amzn.to/shortlink
https://www.amazon.co.uk/dp/B0XXXXXXXXX
```

The "As an Amazon Associate I earn from qualifying purchases." disclosure appears automatically in the modal for Amazon products.

---

## How the Site Fetches Products

1. Fetches file list: `GET https://api.github.com/repos/drmsi/Brandovra.com/contents/products`
2. Filters `.json` files only, sorts by filename (timestamp) **descending** → newest first
3. Fetches each file from: `https://raw.githubusercontent.com/drmsi/Brandovra.com/main/products/{filename}`
4. Caches in `localStorage` for 10 minutes to avoid GitHub API rate limits
5. Auto-refreshes every 10 minutes (bypasses cache)

---

## Testing

1. Trigger the workflow manually in n8n
2. Check GitHub repo → `/products/` folder → new `{timestamp}.json` file should appear
3. Visit brandovra.com — new product appears at top of grid (newest-first)
4. If not appearing: clear browser localStorage or wait 10 minutes for cache to expire

---

## Google Sheet Source (Optional)

To pull products from a Google Sheet instead of a Set node:

1. Add **Google Sheets** node → **Get Row(s)** operation
2. Map columns to the fields above in the Set node
3. Add an **If** node to skip rows where `title` is empty
4. Connect to Build Payload → Push to GitHub

Each sheet row = one product push per workflow run.
