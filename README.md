# 🏨 Hotel Reputation Intelligence via n8n Automation

> An AI-powered self-hosted n8n workflow that monitors hotel reviews across Google Maps, TripAdvisor, Booking.com, and the web that classifies sentiment in 4 languages, drafts multilingual responses, and delivers real-time Slack alerts. Built for $0/month on free tiers.

![n8n](https://img.shields.io/badge/n8n-self--hosted-orange)
![Groq](https://img.shields.io/badge/LLM-Groq%20LLaMA%203.1-blue)
![Sources](https://img.shields.io/badge/Sources-Google%20Maps%20%7C%20TripAdvisor%20%7C%20Booking.com%20%7C%20Web-green)
![Cost](https://img.shields.io/badge/Cost-%240%2Fmonth-brightgreen)
![Languages](https://img.shields.io/badge/Languages-Arabic%20%7C%20French%20%7C%20English%20%7C%20Tunisian-purple)

---

## ✨ Features

- 🌐 **Multi-source monitoring** — Google Maps, TripAdvisor, Booking.com, and web mentions in one workflow
- 🤖 **AI sentiment classification** — positive / neutral / negative with key issue extraction
- ✍️ **Multilingual response drafting** — responds in the reviewer's own language automatically
- 🔔 **Real-time Slack alerts** — only for reviews that need attention
- 🔄 **Smart deduplication** — never processes the same review twice across runs
- 📊 **Weekly digest** — aggregated stats, average rating, and top issues per week
- 🆔 **Auto Place ID lookup** — no manual setup needed for Google Maps or TripAdvisor IDs
- 💰 **$0/month** on free tier APIs

---

## 📌 The Problem It Solves

Hotels receive reviews in multiple languages across multiple platforms with no centralized system to track or respond consistently.

**Slow or missing responses to negative reviews = lost future bookings.**

This workflow answers one question every morning:

> *"What are guests saying about my hotel right now — and what should I say back?"*

---

## 🏗️ Architecture

```
Schedule Trigger (8AM daily)
└── Read Hotels (Google Sheets)
    └── Is Active?
        │
        ├── Branch 1 — Google Maps
        │   ├── Has Place ID? → Lookup Place ID (SerpAPI) → Save Place ID
        │   └── Fetch Google Reviews → Split Google Reviews
        │
        ├── Branch 2 — TripAdvisor
        │   ├── Has TripAdvisor ID? → Lookup TripAdvisor ID (SerpAPI) → Save TripAdvisor ID
        │   └── Fetch TripAdvisor Reviews → Split TripAdvisor Reviews
        │
        ├── Branch 3 — Booking.com + Review Sites (Tavily)
        │   └── Fetch Review Site Mentions → Extract Review Site Mentions
        │
        └── Branch 4 — Web Mentions (Tavily)
            └── Fetch General Mentions → Extract General Mentions
                │
                └── Merge All Sources
                    └── Normalize Reviews
                        └── Read Logged IDs (Google Sheets)
                            └── Filter New Reviews Only
                                └── Has New Reviews?
                                    ├── TRUE → No Operation
                                    └── FALSE → Classify Sentiment (Groq)
                                        └── Extract Sentiment
                                            └── Log Reviews (Google Sheets)
                                                ├── Is Negative?
                                                │   ├── TRUE → Generate Response (Groq)
                                                │   │          └── Update Response in Log
                                                │   │              └── Send Slack Alert
                                                │   └── FALSE → No Operation
                                                ├── Is Web Mention? → Log Web Mentions
                                                └── Aggregate Weekly Stats → Log Weekly Digest
```

---

## 🔧 Tech Stack

| Tool | Purpose | Free Tier |
|------|---------|-----------|
| **n8n** (self-hosted) | Workflow orchestration | Unlimited |
| **SerpAPI** | Google Maps + TripAdvisor reviews | 250 searches/month |
| **Tavily** | Booking.com mentions + web search | 1,000 searches/month |
| **Groq LLaMA 3.1 8B** | Sentiment analysis + response generation | Generous free tier |
| **Google Sheets** | Data storage and logging | Free |
| **Slack Incoming Webhook** | Real-time alerts | Free |

**Total monthly cost: $0** for a single hotel on free tiers.

---

## 🌍 Multilingual Support

| Language | Sentiment Classification | Response Generation |
|----------|--------------------------|-------------------|
| 🇫🇷 French | ✅ | ✅ |
| 🇬🇧 English | ✅ | ✅ |
| 🇸🇦 Arabic | ✅ | ✅ |
| 🇹🇳 Tunisian dialect | ✅ | ✅ |
| 🇩🇪 German / 🇮🇹 Italian / other | ✅ | ✅ |

Both classification and response generation detect the reviewer's language automatically.

---

## 📊 Google Sheets Structure

### `hotels` tab
| Column | Description | Auto-populated |
|--------|-------------|---------------|
| hotel_name | Full hotel name | — |
| city | City | — |
| language | Primary language (fr/en/ar) | — |
| active | TRUE/FALSE | — |
| place_id | Google Maps Place ID | ✅ First run |
| tripadvisor_id | TripAdvisor location ID | ✅ First run |

### `reviews_log` tab
| Column | Description |
|--------|-------------|
| timestamp | When logged |
| hotel_name | Hotel name |
| platform | Google Maps / TripAdvisor / Booking.com / Web |
| reviewer | Reviewer name |
| review_date | Date of review |
| review_text | Full review text |
| rating | Guest rating (normalized to /5) |
| llm_score | AI sentiment score (1-5) |
| sentiment | positive / neutral / negative |
| key_issue | Main complaint in English |
| requires_response | TRUE/FALSE |
| suggested_response | AI-drafted response in reviewer's language |
| review_id | Unique identifier |
| source_url | Original review URL |

### `web_mentions_log` tab
| Column | Description |
|--------|-------------|
| timestamp | When logged |
| hotel_name | Hotel name |
| platform | Source platform |
| source_url | URL of mention |
| title | Page title |
| review_text | Mention content |
| sentiment | positive / neutral / negative |
| key_issue | Main topic |
| llm_score | AI score (1-5) |
| requires_response | TRUE/FALSE |
| review_id | Unique identifier |

### `weekly_digest` tab
| Column | Description |
|--------|-------------|
| week | ISO week label (e.g. 2026-W17) |
| hotel_name | Hotel name |
| total_reviews | Total reviews processed |
| positive | Count of positive reviews |
| neutral | Count of neutral reviews |
| negative | Count of negative reviews |
| avg_rating | Average guest rating (/5) |
| top_issue | Most frequently mentioned issue |
| run_timestamp | When digest was generated |

---

## 🔄 Deduplication — 3 Layers

**Layer 1 — Source filtering**
URL-based hotel validation ensures only the correct hotel's review pages pass through. Wrong hotel pages, listing pages, and promotional content are excluded before any API calls.

**Layer 2 — Batch deduplication**
Content fingerprinting (first 100 characters) removes duplicates that appear across multiple search results in the same run — before hitting Groq.

**Layer 3 — Cross-run deduplication**
`review_id` matching via `appendOrUpdate` in Google Sheets ensures reviews from previous runs are never reprocessed. Filter New Reviews Only compares all incoming IDs against the full sheet on every run.

---

## 📱 Slack Alert Example

```
🚨 Review Alert — Mövenpick Hotel Du Lac Tunis

🌐 Platform: Booking.com
⭐ Rating: 2.0 | 🤖 LLM Score: 2/5
👤 Reviewer: Matthew
📅 Date: January 2026
🔍 Key Issue: cleanliness issues

💬 Review:
"Many thanks, a fine visit" I had no response from the front desk
regarding the shuttle service. I encountered issues with the
cleanliness of the room, particularly the bathroom.

📊 Sentiment: NEGATIVE

✍️ Suggested Response:
Nous sommes désolés d'apprendre que la propreté de votre chambre
n'a pas répondu à vos attentes. N'hésitez pas à nous contacter.

🔗 Source: https://www.booking.com/reviews/...
```

---

## 📈 Sample Weekly Digest

```
Week:           2026-W17
Hotel:          Mövenpick Hotel Du Lac Tunis
Total Reviews:  32
├── Positive:   22  (69%)
├── Neutral:    3   (9%)
└── Negative:   7   (22%)
Avg Rating:     4.3 / 5
Top Issue:      Manque de produits d'hygiène
```

---

## 🚀 Setup Guide

### Prerequisites
- Self-hosted n8n instance (v2.0+)
- Google account with Sheets access
- [SerpAPI](https://serpapi.com) account — free tier
- [Tavily](https://tavily.com) account — free tier
- [Groq](https://console.groq.com) account — free tier
- Slack workspace with incoming webhook

### Step 1 — Clone the repo
```bash
git clone https://github.com/Bechir02/hotel-reputation-intelligence
cd hotel-reputation-intelligence
```

### Step 2 — Import the workflow
In your n8n instance go to **Workflows → Import from file** and select:
```
workflow/Hotel_Reputation_Intelligence.json
```

### Step 3 — Configure credentials
In n8n **Settings → Credentials** add:

| Credential | Type | Where to get it |
|-----------|------|----------------|
| Google Sheets OAuth2 | OAuth2 | Google Cloud Console |
| Groq API | HTTP Bearer Auth | console.groq.com |
| SerpAPI key | Query param in nodes | serpapi.com/manage-api-key |
| Tavily API key | Body param in nodes | app.tavily.com |
| Slack Webhook | URL in Slack Alert node | Slack App settings |

### Step 4 — Create Google Sheet
Create a new Google Sheet named exactly:
```
Hotel Reputation Intelligence
```
Add four tabs: `hotels`, `reviews_log`, `web_mentions_log`, `weekly_digest`

Use the column structure from the **Google Sheets Structure** section above.

### Step 5 — Add your hotel
In the `hotels` tab add one row:
```
hotel_name       | city  | language | active | place_id | tripadvisor_id
Your Hotel Name  | Tunis | fr       | TRUE   |          |
```
Leave `place_id` and `tripadvisor_id` empty — they are auto-populated on first run.

### Step 6 — Replace API keys
In the imported workflow search and replace:
- `YOUR_SERPAPI_KEY` → your SerpAPI key
- `YOUR_TAVILY_KEY` → your Tavily API key
- `YOUR_SLACK_WEBHOOK_URL` → your Slack webhook URL
- `YOUR_GOOGLE_SHEET_ID` → your Google Sheet ID (from the URL)

### Step 7 — Activate
Set the Schedule Trigger to your preferred time (default 8AM daily) and activate.

On first run the workflow auto-discovers and saves Place IDs. From the second run onwards it only processes new reviews.

---

## 🧠 AI Prompt Design

### Sentiment Classification
The Groq LLaMA model uses a strict prompt that:
- Returns only a valid JSON object — no markdown, no explanation
- Classifies any review mentioning ANY complaint as negative regardless of positive parts
- Extracts `key_issue` as a short English phrase describing the main complaint
- Sets `requires_response: true` only when a real complaint exists

### Response Generation
The model is instructed to:
- Write ONLY in the exact same language as the review — no mixing languages
- Never include language labels or prefixes like "In Arabic:" or "French response:"
- Stay under 60 words
- Acknowledge the specific key issue
- Invite the guest to contact the hotel without specifying how
- Write as the hotel directly — no placeholders

---

## ⚠️ Known Limitations

- SerpAPI free tier: 250 searches/month — sufficient for 1 hotel running daily
- TripAdvisor reviews via SerpAPI limited to most recent page only
- Booking.com content comes from Tavily web index — not direct API access
- Review dates from Booking.com via Tavily may be approximate
- Facebook and Instagram not supported (API restrictions)
- Excel serial date format from some Booking.com entries is auto-converted

---

## 🗺️ Roadmap

- [ ] Facebook Reviews integration via Apify
- [ ] Weekly email digest every Friday
- [ ] Week-over-week trend comparison
- [ ] Priority scoring — urgent / normal / low
- [ ] Response approval step before Slack delivery
- [ ] Multi-hotel support with per-hotel channels
- [ ] TripAdvisor pagination for deeper review history

---

## 🤝 Contributing

Pull requests welcome. For major changes please open an issue first to discuss what you would like to change.

---

## 👤 Author

**Becher Zribi** — Data Automation & Quality Analyst  
Building automation workflows for Tunisian businesses and beyond.

[GitHub](https://github.com/Bechir02) · [LinkedIn](https://linkedin.com/in/becher-zribi)

---

## 📄 License

MIT License — free to use, modify, and distribute.
