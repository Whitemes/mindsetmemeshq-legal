# MindsetMemesHQ Publisher

Automation to turn **voice commands** into **memes + motivational posts** and publish them to **X (Twitter)** and **Instagram** using **n8n**.

> Stack: n8n • Telegram Bot • OpenAI (Whisper + GPT‑5 + gpt‑image‑1) • Instagram Graph API • X API v2

---

## 1) Features

- 🎙️ **Voice → Text** via Telegram + Whisper (`whisper-1`)
- 🧠 **Idea → JSON** via GPT‑5 (short post + meme description)
- 🖼️ **Image Generation** with `gpt-image-1` (3:2, mobile‑friendly)
- 🚀 **Auto‑Publish** to **X** and/or **Instagram Business**
- 🔁 Optional: web research tool for trending topics
- 🔒 Credentials encrypted (n8n `N8N_ENCRYPTION_KEY`)
- 💾 One‑click local backup (exports + DB folder)

---

## 2) Architecture (high level)

```
Voice (Telegram) 
  → Telegram Trigger (n8n)
    → Download voice (.oga)
      → Transcribe (OpenAI Whisper)
        → Chat (GPT‑5) → { name, title, text, image }
          → Images (gpt‑image‑1) → PNG
            → Storage (Drive/S3/public URL)
              → Publish (X API v2 / Instagram Graph API)
                → (Optional) Send result back to Telegram
```

---

## 3) Prerequisites

### System
- Windows 10/11 64‑bit
- Node.js LTS ≥ 18 (includes npm)
- (optional) Git for local versioning

### Accounts & API keys
- **Telegram Bot token** (via `@BotFather`)
- **OpenAI API key** (Whisper + Images)
- **X Developer** project/app (OAuth 2.0; scopes `tweet.write`, `media.write`, optional `users.read`)
- **Instagram Business** account linked to a **Facebook Page**
- **Meta App** with **Instagram Graph API** enabled
  - Public URLs:
    - Privacy Policy
    - Data Deletion Instructions
    - (optional) Terms of Service

> For tests in Development mode: add your IG account as **Instagram Tester** in the app roles and **accept** the invite from the Instagram mobile app.

---

## 4) Install n8n on Windows

```powershell
# 1) Install
npm install -g n8n

# 2) Persistent user folder
mkdir C:\n8n-data
setx N8N_USER_FOLDER "C:\n8n-data"

# 3) (Recommended) fixed encryption key
# generates a 32-byte random key (Base64) – copy output then:
powershell -NoProfile -Command "$b = New-Object byte[] 32; (New-Object System.Security.Cryptography.RNGCryptoServiceProvider).GetBytes($b); [Console]::WriteLine([Convert]::ToBase64String($b))"
setx N8N_ENCRYPTION_KEY "<PASTE_YOUR_BASE64_KEY>"

# 4) Run
n8n
# UI: http://localhost:5678
```

To expose webhooks in local dev (Telegram trigger), run with tunnel:
```powershell
n8n start --tunnel
```

---

## 5) Credentials in n8n

- **Telegram API** → paste `Access Token` from `@BotFather`  
  Test token from PowerShell:
  ```powershell
  $token="<BOT_TOKEN>"
  Invoke-RestMethod -Uri "https://api.telegram.org/bot$token/getMe"
  ```

- **OpenAI** → API key in `Credentials > OpenAI API`

- **Instagram** → use `HTTP Request` node with **query param** `access_token=<LONG_LIVED_USER_TOKEN>`

- **X (Twitter)** → either:
  - use an aggregator (simpler), **or**
  - call X v2: media upload → create tweet (requires OAuth 2.0 app + tokens)

> Keep all secrets inside **Credentials** (never hard-code in nodes).

---

## 6) Instagram Graph API (publish flow)

### 6.1 Prepare
- IG account is **Professional (Business)** and **linked** to a **Facebook Page**
- Get **Page ID**: `GET /me/accounts`  
- Get **IG Business User ID**: `GET /{page-id}?fields=instagram_business_account` → use `id`

### 6.2 Create media then publish

**Create Media (container)**
```
POST https://graph.facebook.com/v18.0/{ig-user-id}/media
?image_url={PUBLIC_PNG_URL}
&caption={YOUR_CAPTION}
&access_token={LONG_LIVED_TOKEN}
```

**Publish**
```
POST https://graph.facebook.com/v18.0/{ig-user-id}/media_publish
?creation_id={CREATION_ID}
&access_token={LONG_LIVED_TOKEN}
```

Tips:
- Wait ~2–3s (or poll) between create & publish.
- Image URL must be **public**.
- Common error codes: 10/200 → missing permission or not Business/linked.

---

## 7) X (Twitter) API v2 (outline)

**Upload media** → get `media_id` → **Create Tweet** with `media.media_ids`.

- Scopes: `tweet.write`, `media.write` (optional `users.read` to fetch own user id)
- Respect rate limits of the Free tier (write‑only).

> If you prefer, use a posting aggregator and call:
> 1) `/media` (ingest) → returns a hosted media URL/ID  
> 2) `/posts` with `targetType="twitter"` and `mediaUrls=[...]`

---

## 8) The core n8n workflow (nodes)

1. **Telegram Trigger** (events: `message`, `voice`)
2. **Telegram → Get File** (input: `{{$json.message.voice.file_id}}`)
3. **OpenAI → Audio → Transcribe** (`whisper-1`) → output `text`
4. **OpenAI → Chat** (`gpt-5`)  
   **System**: 
   ```
   You are a veteran tech-meme writer for AI/Twitter.
   Output ONLY valid JSON: {name,title,text,image}.
   Max 3 sentences in "text", each separated by two newlines.
   ```
   **User**:
   ```
   Voice idea: "{{ $json.text }}".
   Return: { "name": "", "title": "", "text": "", "image": "" }
   ```
5. **HTTP Request → OpenAI Images (generations)**  
   `POST https://api.openai.com/v1/images/generations`  
   ```json
   {
     "model": "gpt-image-1",
     "prompt": "Create a meme-ready 3:2 image for Twitter/Instagram. Background: black with dark grainy film. Colors: black/white + accent salad green #cff150. Minimal composition; crisp text. Content: {{ $('OpenAI Chat').item.json.image }}",
     "size": "1536x1024"
   }
   ```
   → get `data[0].b64_json`
6. **Convert to File** (base64 → PNG)
7. **Uploader** (Drive or S3) → get **public URL**
8. **Branch**:
   - **Instagram**: 6.2 flow above
   - **X**: media upload v2 → create tweet
9. **Telegram → Send Photo** (optional confirmation)

---

## 9) Backups (local only)

```powershell
# Export each workflow & credentials to versionable files
mkdir C:\n8n-backups\workflows
mkdir C:\n8n-backups\credentials

n8n export:workflow --all --separate --pretty --output "C:\n8n-backups\workflows"
n8n export:credentials --all --pretty --output "C:\n8n-backups\credentials"

# Zip user folder (DB + encryption key)
Compress-Archive -Path "C:\n8n-data\*" -DestinationPath "C:\n8n-backups\backup-$(Get-Date -Format yyyyMMdd-HHmmss).zip"
```

> Never back up **decrypted** credentials. Keep `N8N_ENCRYPTION_KEY` safe.

---

## 10) Troubleshooting

- **Telegram: Authorization failed**  
  Token typo or network. Test with PowerShell: `getMe`. In n8n, remove leading/trailing spaces; base URL stays `https://api.telegram.org`.

- **Images API 400 / quality**  
  Omit `quality` on the `edits` endpoint; re‑try. For simple generations, use `/images/generations`.

- **Instagram: code 10/200**  
  Check: Business account, linked Page, correct permissions, token not expired, tester accepted, image URL is public.

- **X: write succeeded but media missing**  
  Ensure media upload v2 succeeded; pass returned `media_id` to `/2/tweets`.

---

## 11) Security Notes

- Store secrets only in **Credentials**.  
- Restrict who can access the n8n UI.  
- Rotate tokens regularly.  
- Avoid generating real persons’ faces/logos without rights.

---

## 12) Roadmap / TODO

- Add retries + dead‑letter queue for posting
- Add Perplexity/Custom Web Search tool
- Add analytics (posted URL, engagement id) to a sheet/DB
- One‑click “cross‑post” switch (X + IG)

---

## 13) License & Contact

Private/internal automation for @mindsetmemeshq.  
Contact: **vocal.meme@outlook.com**
