# TalentBridge — Intelligent Recruiter Agent

> An AI-powered recruiting agent that matches candidates to job descriptions, generates personalized "Why this person?" pitches, and drafts LinkedIn outreach — all in one deployable web app.

[![Built with Claude](https://img.shields.io/badge/AI-Claude%20Sonnet%204-8B5CF6?style=flat-square)](https://anthropic.com)
[![Runs on Replit](https://img.shields.io/badge/Deploy-Replit-F26207?style=flat-square)](https://replit.com)
[![Node.js](https://img.shields.io/badge/Node.js-20+-339933?style=flat-square&logo=node.js)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

---

## What it does

Traditional job boards rank candidates by keywords. TalentBridge goes further — it understands context.

Paste a job description and a pool of candidate profiles. The agent embeds both into semantic vectors, runs cosine similarity matching to surface the top 5 candidates, then calls Claude to write a three-paragraph "Why this person?" pitch for each one. It also generates personalized LinkedIn connection notes and InMail messages at your chosen tone.

**The full pipeline:**

```
Job description + Candidate pool + LinkedIn context
        ↓
  NLP extraction  ·  Profile normalisation  ·  Context enrichment
        ↓
  MiniLM-L6-v2 embeddings  →  Cosine similarity ranking
        ↓
  Claude Sonnet 4 pitch agent  (streamed live to UI)
        ↓
  "Why this person?" cards  ·  Shortlist export  ·  Outreach messages
```

---

## Features

- **Semantic vector matching** using `@xenova/transformers` (MiniLM-L6-v2) — no paid vector database required
- **AI pitch generation** streamed live, covering technical fit, unique value, and one interview question to probe
- **LinkedIn outreach generator** — connection note (≤300 chars) and full InMail per candidate, with tone selection (Professional / Casual / Enthusiastic)
- **Match score rings** — circular SVG progress indicators colour-coded green/yellow/red per candidate
- **Shortlist persistence** via Replit Database — survives page refreshes and new sessions
- **CSV export** of your shortlist, generated client-side with no server round-trip
- **5 sample candidates pre-loaded** so the demo works immediately, no setup needed
- **Graceful fallbacks** — template-based pitches if Claude API is unavailable; TF-IDF keyword scoring if embeddings fail to load

---

## Quick start

### 1. Fork on Replit

Click **Use Template** on the Replit project page. Everything is pre-configured.

### 2. Add your API key

Open **Secrets** (the lock icon in the sidebar) and add:

```
Key:   ANTHROPIC_API_KEY
Value: sk-ant-...
```

This is the only required secret. The app will not start without it.

### 3. Run

Click the **Run** button. The server starts on port 3000 and Replit opens a preview automatically.

---

## Local development

```bash
# Clone
git clone https://github.com/your-org/talentbridge.git
cd talentbridge

# Install dependencies
npm install

# Set environment variable
export ANTHROPIC_API_KEY=sk-ant-...

# Start the server
node server/index.js
```

Open `http://localhost:3000` in your browser.

---

## Project structure

```
talentbridge/
├── client/
│   ├── index.html              # Single-page app shell
│   ├── App.jsx                 # Root React component
│   ├── components/
│   │   ├── JobDescriptionInput.jsx   # JD textarea + tag extraction
│   │   ├── CandidateUploader.jsx     # Manual entry + JSON upload
│   │   ├── MatchResults.jsx          # Top-5 ranked candidate list
│   │   ├── CandidateCard.jsx         # Score ring + pitch + actions
│   │   └── OutreachModal.jsx         # LinkedIn message preview
│   └── styles.css
├── server/
│   ├── index.js                # Express entry point (port 3000)
│   ├── routes/
│   │   ├── match.js            # POST /api/match
│   │   ├── pitch.js            # POST /api/pitch  (streamed)
│   │   └── outreach.js         # POST /api/outreach
│   └── lib/
│       ├── embeddings.js       # Vector generation + cosine similarity
│       ├── claude.js           # Anthropic API wrapper
│       └── linkedin.js         # Outreach message generator
├── .env.example
├── .replit                     # run = "node server/index.js"
├── package.json
└── README.md
```

---

## API reference

### `POST /api/match`

Embeds the job description and all candidate profiles, returns the top 5 ranked by cosine similarity.

**Request**
```json
{
  "jobDescription": "Senior React engineer, 5+ years, team lead experience...",
  "candidates": [
    {
      "id": "c1",
      "name": "Priya Menon",
      "title": "Senior Full-Stack Engineer",
      "summary": "8 years building scalable web apps...",
      "skills": ["React", "Node.js", "TypeScript"],
      "experience": 8
    }
  ]
}
```

**Response**
```json
{
  "matches": [
    {
      "candidate": { "id": "c1", "name": "Priya Menon", "..." : "..." },
      "score": 0.87,
      "rank": 1,
      "matchedSkills": ["React", "TypeScript", "Team Leadership"]
    }
  ]
}
```

---

### `POST /api/pitch`

Calls Claude to generate a three-paragraph pitch. Response is chunked (streamed).

**Request**
```json
{
  "jobDescription": "...",
  "candidate": { "..." : "..." },
  "score": 0.87
}
```

**Response** — `Transfer-Encoding: chunked` text stream. The client appends each chunk to the pitch card as it arrives.

---

### `POST /api/outreach`

Generates a LinkedIn connection note and InMail for a given candidate and tone.

**Request**
```json
{
  "candidate": { "name": "Priya Menon", "title": "Senior Full-Stack Engineer", "..." : "..." },
  "jobTitle": "Engineering Lead",
  "tone": "professional"
}
```

**Response**
```json
{
  "connectionNote": "Hi Priya — your work on distributed React systems caught my eye...",
  "inMail": "Hi Priya, I came across your profile while searching for..."
}
```

---

## Candidate data format

You can upload a `.json` file containing an array of candidates. Each object supports these fields:

| Field        | Type     | Required | Description                              |
|--------------|----------|----------|------------------------------------------|
| `id`         | string   | yes      | Unique identifier                        |
| `name`       | string   | yes      | Full name                                |
| `title`      | string   | yes      | Current or most recent job title         |
| `summary`    | string   | yes      | Bio or profile summary (used for vector) |
| `skills`     | string[] | yes      | List of skills (used for highlights)     |
| `experience` | number   | yes      | Years of experience                      |
| `linkedin`   | string   | no       | LinkedIn profile URL                     |

A sample file with 5 pre-loaded candidates is included at `client/data/sample-candidates.json`.

---

## Configuration

| Environment variable  | Required | Default               | Description                  |
|-----------------------|----------|-----------------------|------------------------------|
| `ANTHROPIC_API_KEY`   | yes      | —                     | Your Anthropic API key       |
| `PORT`                | no       | `3000`                | HTTP server port             |
| `TOP_N_RESULTS`       | no       | `5`                   | Number of top matches to return |
| `PITCH_MAX_TOKENS`    | no       | `1000`                | Max tokens per pitch         |

---

## Deployment on Replit

1. Fork the Repl.
2. Add `ANTHROPIC_API_KEY` to Secrets.
3. Click **Deploy → Autoscale** (free tier).
4. Replit provisions a public URL — share it directly with your team.

The `/health` endpoint returns `{ "status": "ok" }` and is used by Replit's deployment health checks.

---

## How the matching works

Each candidate's profile is compiled into a single document — title + summary + skills joined as plain text. The job description is treated the same way. Both are passed through the `Xenova/all-MiniLM-L6-v2` sentence transformer model, which produces 384-dimensional vectors. Cosine similarity between the job vector and each candidate vector produces a score between 0 and 1, shown in the UI as a percentage.

If the embedding model fails to load (cold start, memory limit), the server falls back to a pure-JS TF-IDF implementation that scores candidates by weighted keyword overlap. Results are less nuanced but the app stays usable.

---

## Pitch structure

Each Claude-generated pitch covers three areas:

1. **Technical fit** — specific skills and experience from the candidate's profile that directly match the job requirements
2. **Unique value** — what this person brings that other candidates likely don't
3. **Interview probe** — one honest concern or open question, with a suggested interview question to address it

This gives recruiters something real to work with, not generic praise.

---

## Tech stack

| Layer          | Technology                                   |
|----------------|----------------------------------------------|
| Frontend       | React 18, Tailwind CSS                       |
| Backend        | Node.js 20, Express 4                        |
| Embeddings     | @xenova/transformers (MiniLM-L6-v2)          |
| AI             | Anthropic Claude Sonnet 4 (`claude-sonnet-4-20250514`) |
| Persistence    | Replit Database (`@replit/database`)         |
| Deployment     | Replit Autoscale (free tier)                 |

---

## Contributing

Pull requests are welcome. For significant changes, open an issue first to discuss what you'd like to change.

```bash
# Run tests
npm test

# Lint
npm run lint
```

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

*Built for Hackathon Problem Statement 2: The Intelligent Recruiter*
