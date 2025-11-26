---
title: LLM Code Deployment System
emoji: 🐋
colorFrom: blue
colorTo: green
sdk: docker
python_version: "3.13"
app_file: main.py
pinned: false
---

# LLM Code Deployment System

An automated application deployment system that receives briefs, generates apps using LLMs, deploys them to GitHub Pages (or Spaces/Vercel), and handles iterative updates.

## Overview

This project implements an automated workflow for building and deploying web applications:

1. **Build Phase**: Receives a brief, generates code using an LLM, creates/updates a GitHub repository, and deploys to GitHub Pages.
2. **Revise Phase**: Accepts update requests, modifies existing code, and redeploys.

## Features

- LLM-powered code generation (Gemini / OpenAI-compatible APIs)
- Automated GitHub repository creation and Pages deployment
- Multi-round support (initial build + revisions)
- Secret verification for request authentication
- Auto-generated professional README and MIT license
- Robust retry / exponential backoff logic for external API calls

## Setup Instructions

### 1. Prerequisites

- Python 3.13+
- uv package manager (https://github.com/astral-sh/uv) or use pip if preferred
- GitHub account with a Personal Access Token
- OpenAI/Gemini API key (or compatible LLM provider key)

### 2. Install Dependencies

All required packages are defined in `pyproject.toml`.

```bash
# Install packages with uv
uv sync
```

(If you prefer pip, use your virtualenv and `pip install -r requirements.txt` if you maintain a requirements file.)

### 3. Configure Environment Variables

Copy the example environment file and fill in your credentials:

```bash
cp .env.example .env
```

Edit `.env` and set the following:

- `GITHUB_TOKEN`: Your GitHub Personal Access Token  
  - Create at: https://github.com/settings/personal-access-tokens/new  
  - Required scopes: `repo`, `workflow`, `admin:repo_hook`
- `GITHUB_USERNAME`: Your GitHub username
- `OPENAI_API_KEY`: Your OpenAI / Gemini API key (primary LLM provider key)
- `SECRET`: Shared secret for request verification
- `PORT`: (Optional) Server port, defaults to 5000
- `AIPIPE_AKI_KEY`: (Optional) AI Pipe API token used as a fallback LLM provider

### 4. GitHub Personal Access Token Setup

1. Go to https://github.com/settings/tokens/new
2. Give it a descriptive name (e.g., "LLM Deployment System")
3. Select scopes:
   - `repo` (Full control of repositories)
   - `workflow` (Update GitHub Action workflows)
   - `admin:repo_hook` (Manage repository webhooks)
4. Click "Generate token" and copy it to your `.env` file

### 5. Verify Configuration

Before running the server, verify your configuration:

```bash
uv run check_config.py
```

This script will validate environment variables and tokens and provide guidance when something is missing.

## Usage

### Running the Server

```bash
uv run main.py
```

The server will start on `http://localhost:5000` (or the port specified in your `.env`).

### Run with Docker

Build the image and run the container (exposes port 5000 by default):

```bash
# Build
docker build -t llm-deploy:latest .

# Run (maps host 5000 -> container 5000)
docker run --env-file .env -p 5000:5000 llm-deploy:latest
```

Override the port by setting `PORT` in your `.env` and mapping it accordingly, e.g., `-p 8080:8080`.

### Deploy to Vercel

You can deploy this application to Vercel for production hosting.

1. Install Vercel CLI:

```bash
npm i -g vercel
```

2. Authenticate:

```bash
vercel
```

3. Test locally:

```bash
vercel dev
```

4. Deploy:

```bash
vercel --prod
```

After deploying, set environment variables in the Vercel dashboard.

### Deploy to Hugging Face Spaces (Docker)

Hugging Face Spaces supports Docker and is a good hosting option for a simple API.

Method 1 — Git-based deployment (recommended):

1. Generate a Hugging Face token: https://huggingface.co/settings/tokens (give "Write" permission)
2. Create a new Space and choose SDK: Docker
3. Add secrets in Space settings:
   - `GITHUB_TOKEN`, `GITHUB_USERNAME`, `OPENAI_API_KEY`, `SECRET`, `AIPIPE_AKI_KEY`
4. Push the repo to the Space remote:

```bash
git remote add hf https://huggingface.co/spaces/{username}/{spacename}
git add .
git commit -m "Deploy to Hugging Face Spaces"
git push hf main
```

If authentication fails, set the remote URL with token:
```bash
git remote set-url hf https://{username}:{your_hf_token}@huggingface.co/spaces/{username}/{spacename}
git push hf main
```

Method 2 — Hugging Face CLI:

```bash
pip install huggingface_hub
huggingface-cli login
huggingface-cli repo create {username}/{spacename} --type space --private
huggingface-cli upload {username}/{spacename} . --repo-type=space
```

Verify deployment once the Space build completes (2–5 minutes). The API endpoints will be available under the Space URL.

Note on Space configuration: the YAML frontmatter at the top of this README contains the configuration used for Spaces (sdk: docker, app_file: main.py). Keep that consistent with your repo.

## API Endpoints

### POST /api-endpoint

Main endpoint for build and revise requests.

Request format (example):

```json
{
  "email": "student@example.com",
  "secret": "your_secret_key",
  "task": "captcha-solver-xyz",
  "round": 1,
  "nonce": "ab12-cd34",
  "brief": "Create a captcha solver that handles ?url=https://.../image.png",
  "checks": [
    "Repo has MIT license",
    "README.md is professional",
    "Page displays captcha URL passed at ?url=...",
    "Page displays solved captcha text within 15 seconds"
  ],
  "evaluation_url": "https://example.com/notify",
  "attachments": [
    {
      "name": "sample.png",
      "url": "data:image/png;base64,iVBORw..."
    }
  ]
}
```

Response (example):

```json
{
  "status": "success",
  "message": "Successfully processed round 1",
  "repo_url": "https://github.com/username/captcha-solver-xyz-round-1",
  "pages_url": "https://username.github.io/captcha-solver-xyz-round-1/",
  "commit_sha": "abc123def456"
}
```

### GET /health

Health check endpoint. Example response:

```json
{ "status": "healthy" }
```

## Testing with cURL

```bash
curl http://localhost:5000/api-endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "secret": "your_secret_key",
    "task": "test-app-001",
    "round": 1,
    "nonce": "test-nonce-123",
    "brief": "Create a simple calculator app",
    "checks": [
      "Has MIT license",
      "README is professional",
      "Calculator works correctly"
    ],
    "evaluation_url": "https://httpbin.org/post"
  }'
```

## Architecture

### Core Components

1. Configuration Module (`utils/config.py`)
   - Loads and validates environment variables
   - Initializes API clients (primary LLM, fallback client)
   - Authenticates GitHub client

2. Request Handler (`main.py`)
   - Flask API endpoints
   - Request validation and secret verification
   - Orchestrates the build/revise workflow

3. Validation Module (`utils/validation.py`)
   - Ensures required fields and correct types
   - Validates secret and optional fields

4. File Handler (`utils/file_handler.py`)
   - Supports text, CSV, JSON, images, audio, video, documents
   - Decodes data URIs and base64 payloads robustly
   - Detects and flags files needing conversion

5. Code Generator (`utils/code_generator.py`)
   - Primary LLM: Gemini or OpenAI-compatible provider
   - Fallback: AI Pipe (when configured)
   - Generates application code and README content

6. GitHub Manager (`utils/github_manager.py`)
   - Creates and updates repositories
   - Adds LICENSE and README.md
   - Uploads `index.html` or other generated files
   - Configures GitHub Pages and triggers builds with retry logic

7. API Notifier (`utils/api_notifier.py`)
   - Posts evaluation payloads to `evaluation_url`
   - Implements exponential backoff and timeout handling

### System Workflow (high-level)

- Incoming POST → validate request & secret → process attachments → (if round >1) fetch existing code → generate or modify code via LLM → create/update GitHub repo → enable Pages → update README → notify evaluation API → return response.

(See source files for implementation details.)

## Error Handling Strategy

- Validation errors return HTTP 400 with a specific message.
- Internal errors return HTTP 500 with step information for debugging.
- API failures use automatic retry and fallback (LLM fallback, Pages build retries).
- Evaluation API notifications retry with exponential backoff (1, 2, 4, 8, 16 seconds up to max attempts).

## Round 2 (Revise) Handling

When `round` is 2+:

1. The system locates the existing repository for round 1.
2. It includes prior code in the prompt for the LLM.
3. Generates updated code and commits changes.
4. Regenerates README and notifies evaluation API with the new commit SHA.

## Security Considerations

- Requests are verified using a shared `SECRET`.
- Secrets are not stored in git history (use environment variables).
- Add `.env` to `.gitignore` to avoid accidental commits.

## Dependencies

Core libraries used in this project:

- flask — Web framework for API endpoint
- openai — LLM integration (Gemini/OpenAI-compatible)
- pygithub — GitHub API client
- requests — HTTP client for evaluation API and Pages setup
- python-dotenv — Environment variable management
- gunicorn — Production WSGI server (for Docker/production)

(See `pyproject.toml` or requirements for exact pinned versions.)

## Limitations & Future Improvements

- Currently focused on single-page HTML applications.
- GitHub Pages builds may take 1–2 minutes.
- Rate limits apply for LLM and GitHub APIs.
- Future: support multi-file projects, caching, more robust CI/CD integration.

## License

MIT License — see LICENSE file for details.

## Troubleshooting

- GitHub Token Issues:
  - Ensure required scopes (`repo`, `workflow`, `admin:repo_hook`) are enabled.
- LLM / OpenAI Issues:
  - Verify your API key and account limits.
- GitHub Pages Not Deploying:
  - Wait 1–2 minutes and check repository → Settings → Pages.
  - Ensure `index.html` is present in the main branch.
- Port Already in Use:
  - Change `PORT` in `.env` or kill the process using the port:
    ```bash
    lsof -ti:5000 | xargs kill -9
    ```

 verify:
- `/health` returns `{"status":"healthy"}`
- `/api-endpoint` accepts POST requests with authentication
- Repository visibility and environment variables are configured

## Evidence & Logging

The project contains mechanisms to log requests and responses for auditing and evaluation purposes. Logs include request payloads, responses, timestamps, and status.

## Support

If you run into issues:
1. Check the Troubleshooting section above
2. Review console error messages and logs
3. Verify environment variables and API tokens
4. Reach out (open an issue on this repository) with clear reproduction steps

---

If you'd like, I can:
- Open a pull request that applies this corrected README.md,
- Or directly push the change to a branch and create the PR for you. Tell me which you'd prefer.

