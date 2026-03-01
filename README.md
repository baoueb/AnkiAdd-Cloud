# AnkiAdd-Cloud

Add Japanese vocabulary to **Anki Desktop** from your **iPhone/iPad** — even when you’re not on the same Wi‑Fi.

This repo contains the **Cloud Sync method** used by AnkiAdd: your phone uploads a card to your own cloud endpoint, and a small helper running on your computer adds it to Anki via **AnkiConnect**.

---

## Table of contents

- [What this is (in one minute)](#what-this-is-in-one-minute)
- [What you’ll get](#what-youll-get)
- [Who this is for](#who-this-is-for)
- [Who should NOT use this](#who-should-not-use-this)
- [Privacy & data-sharing](#privacy--data-sharing)
- [Security model](#security-model)
- [What’s in this repo](#whats-in-this-repo)
- [Quickstart (10 minutes)](#quickstart-10-minutes)
  - [Step 1 — Install & verify AnkiConnect (Desktop)](#step-1--install--verify-ankiconnect-desktop)
  - [Step 2 — Create your API token (keep it secret)](#step-2--create-your-api-token-keep-it-secret)
  - [Step 3 — Create AWS backend (minimal)](#step-3--create-aws-backend-minimal)
  - [Step 4 — Verify your cloud API (copy/paste)](#step-4--verify-your-cloud-api-copypaste)
  - [Step 5 — Run the Desktop Puller (foreground test)](#step-5--run-the-desktop-puller-foreground-test)
  - [Step 6 — Configure AnkiAdd (iOS)](#step-6--configure-ankiadd-ios)
  - [Step 7 — End-to-end test](#step-7--end-to-end-test)
  - [Common quick fixes](#common-quick-fixes)
- [How it works (architecture + privacy)](#how-it-works-architecture--privacy)
- [AWS setup (step-by-step)](#aws-setup-step-by-step)
- [Backend code (Lambda) + API contract](#backend-code-lambda--api-contract)
- [Desktop Puller (script + LaunchAgent auto-run)](#desktop-puller-script--launchagent-auto-run)
- [Configure AnkiAdd + troubleshooting + FAQ](#configure-ankiadd--troubleshooting--faq)
- [License](#license)

---

## What this is (in one minute)

Normally, **AnkiConnect works only on your computer** (Anki Desktop). Your iPhone can’t talk to it reliably when you’re outside your home network.

**AnkiAdd-Cloud** solves that by using a simple “inbox” in the cloud:

1. **iPhone/iPad** → uploads the card to your private cloud API  
2. **Desktop Puller** → downloads new cards and adds them to Anki Desktop  
3. **Desktop Puller** → reports back **Added / Duplicate / Failed**  
4. **iPhone/iPad** → shows the result automatically

---

## What you’ll get

- ✅ Add cards from iOS to **Anki Desktop**, anywhere  
- ✅ A background **Desktop Puller** (macOS) that can run automatically  
- ✅ Status sync back to your phone:
  - **Added ✓**
  - **Duplicate**
  - **Failed**
- ✅ Works with decks that have custom fields (Expression, Reading, Definition, Tags, Audio, etc.)
- ✅ You control the server (your AWS account)

---

## Who this is for

- Japanese learners who want a **fast, reliable** workflow for adding vocab to Anki
- People comfortable following a step-by-step setup guide (copy/paste + clicking around AWS)

You do **not** need to be a developer, but you should be okay with:

- Creating a few AWS resources (**Lambda + API Gateway + DynamoDB**)
- Running a small script on your computer

---

## Who should NOT use this

- If you don’t want any card data stored in the cloud (even temporarily)
- If you don’t want to set up AWS
- If you want a “one-click, no configuration” solution (this is a more powerful DIY setup)

---

## Privacy & data-sharing

When Cloud Sync is enabled, your phone sends card content to **your** cloud endpoint.

### Typical data sent per card

- The word (**Expression**)
- **Reading / Furigana** (if used)
- **Definitions** (and sometimes example sentences)
- **Tags**
- **Deck name** + **note type name**
- A unique ID so the phone can later show **Added / Duplicate / Failed**

### Not sent

- Your full Anki collection
- Your Anki reviews/history
- Anything from your phone other than what you choose to add

### Where it is stored

- In AWS services (**API Gateway + Lambda + DynamoDB**) under **your** AWS account

If you are not comfortable with this, do not use this method.

---

## Security model

- Requests use **HTTPS**
- Access is protected by a **Bearer token** (a secret string)
- On iOS, the token should be stored in **Keychain**
- Anyone who gets your token can submit notes to your endpoint → treat it like a password

---

## What’s in this repo

- `lambda_function.py` — AWS Lambda backend (API endpoints)
- `ankiadd-puller.py` — Desktop puller script (polls cloud → adds to Anki)
- `LaunchAgent` example — run the puller automatically on macOS
- Setup guide — click-by-click AWS + app configuration

---

## Quickstart (10 minutes)

This Quickstart gets you to a working end-to-end setup:

✅ iPhone uploads → ✅ Desktop Puller adds to Anki Desktop → ✅ iPhone shows **Added / Duplicate / Failed**

> **You control the server.** This method uploads card content (word/definition/tags/etc.) to a cloud endpoint you own.  
> If you don’t want any card data stored in the cloud (even temporarily), do not use Cloud Sync.

---

### What you need

#### On your computer (macOS)

- **Anki Desktop** installed
- **AnkiConnect** add-on installed
- **Python 3.12+** recommended

#### On AWS

- An AWS account
- Region: **ap-northeast-1 (Tokyo)** recommended

#### In AnkiAdd (iOS)

- Sync method **Cloud** available in Settings
- Your deck + field mapping already set up in the app

---

### Step 1 — Install & verify AnkiConnect (Desktop)

1. Open **Anki Desktop**
2. Install **AnkiConnect** (Tools → Add-ons → Get Add-ons…)
3. Restart Anki Desktop

Verify AnkiConnect is reachable:

```bash
curl -sS -X POST "http://127.0.0.1:8765"   -H "Content-Type: application/json"   -d '{"action":"version","version":6}'
```

Expected output:

```json
{"result":6,"error":null}
```

---

### Step 2 — Create your API token (keep it secret)

You need one long random token. Treat it like a password.

You will paste this token into:

- AWS Lambda environment variable: `API_TOKEN`
- Desktop Puller env var: `ANKIADD_CLOUD_TOKEN`
- AnkiAdd → Settings → Cloud → **API Token** (stored in Keychain)

Example token format:

```text
Bw_gWFTgkIUnwVEoPFx5PqNo5jdnBZZeFWN9mjAUW_-v2Pj_ejAVQFKu_uOsWMIo
```

---

### Step 3 — Create AWS backend (minimal)

You need these endpoints:

- `POST /v1/notes/batch` (phone uploads)
- `GET /v1/notes/pending` (desktop puller reads)
- `POST /v1/notes/{id}/result` (desktop puller posts results)
- `GET /v1/notes/status?since=<epoch>` (phone polls for results)

#### 3.1 Create DynamoDB table

DynamoDB → **Create table**

- Table name: `AnkiAddNotes`
- Partition key: `userId` (String)
- Sort key: `noteId` (String)

#### 3.2 Create Lambda function

Lambda → **Create function**

- Name: `AnkiAddApi`
- Runtime: Python (latest available)

Lambda → Configuration → Environment variables

- `API_TOKEN` = your token
- `TABLE_NAME` = `AnkiAddNotes`
- `USER_ID` = `yoyo` (or any identifier you want)

Lambda → Permissions (execution role)

- allow DynamoDB: `PutItem`, `UpdateItem`, `GetItem`, `Query`

> The full Lambda code is provided later in this README. For Quickstart, you just need the API running.

#### 3.3 Create HTTP API Gateway (Tokyo)

API Gateway → **Create API** → **HTTP API**

- Integration: your Lambda (`AnkiAddApi`)

Routes to add (all to the same Lambda integration):

- `POST /v1/notes/batch`
- `GET /v1/notes/pending`
- `POST /v1/notes/{id}/result`
- `GET /v1/notes/status`

Copy the **Invoke URL** (Cloud Base URL). It looks like:

```text
https://xxxx.execute-api.ap-northeast-1.amazonaws.com
```

---

### Step 4 — Verify your cloud API (copy/paste)

In Terminal, set:

```bash
INVOKE_URL="https://xxxx.execute-api.ap-northeast-1.amazonaws.com"
TOKEN="YOUR_TOKEN_HERE"
```

Test **pending**:

```bash
curl -i -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/pending?limit=5"
```

Expected (example):

```json
{"notes":[]}
```

Test **status**:

```bash
curl -i -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/status?since=0"
```

Expected (example):

```json
{"updates":[],"serverTime":1772340585}
```

If you get:

- **401** → token mismatch
- **404** → route missing / wrong API
- **500** → check CloudWatch logs (Lambda error)

---

### Step 5 — Run the Desktop Puller (foreground test)

From the repo folder:

```bash
python3 -m pip install --user requests

export ANKIADD_CLOUD_BASE_URL="$INVOKE_URL"
export ANKIADD_CLOUD_TOKEN="$TOKEN"
export ANKICONNECT_URL="http://127.0.0.1:8765"

# tuning (recommended)
export ANKIADD_POLL_SECONDS="8"
export ANKIADD_BATCH_LIMIT="25"
export ANKIADD_HTTP_TIMEOUT="25"

./ankiadd-puller.py
```

Expected output (example):

```text
Starting AnkiAdd Puller
Cloud: https://xxxx.execute-api.ap-northeast-1.amazonaws.com
AnkiConnect: http://127.0.0.1:8765
AnkiConnect OK, version: 6
```

Leave it running.

---

### Step 6 — Configure AnkiAdd (iOS)

In AnkiAdd → Settings → Cloud:

- Sync Method: **Cloud**
- Cloud Base URL: your invoke URL  
  Example: `https://xxxx.execute-api.ap-northeast-1.amazonaws.com`
- API Token: paste token (Keychain)
- Auto-fetch cloud results: **ON**
- Poll interval: **10–20 seconds**

---

### Step 7 — End-to-end test

1. Make sure **Anki Desktop** is open
2. Make sure **Desktop Puller** is running
3. On iPhone, add a word and tap **Sync**

Expected:

- Phone shows: *Uploaded • waiting for desktop*
- Puller prints: `ADDED: <cloudId> -> <ankiNoteId>`
- Phone later shows: **Added ✓** (or **Duplicate** / **Failed**)

---

### Common quick fixes

**“Uploaded but never added”**

- Desktop Puller not running
- Anki Desktop not open
- AnkiConnect not reachable on `127.0.0.1:8765`

**“Puller keeps timing out”**  
Increase timeout:

```bash
export ANKIADD_HTTP_TIMEOUT="25"
```

**“Pending returns 404”**  
You added routes on a different API than the invoke URL you’re calling.

---

## How it works (architecture + privacy)

This section explains **what happens behind the scenes**, what data is uploaded, and what you should understand before using Cloud Sync.

### High-level flow

1. **AnkiAdd (iPhone/iPad)** uploads your card data to your cloud API.
2. The **Desktop Puller** polls the cloud for pending notes.
3. The puller calls **AnkiConnect** (localhost) to add the note into **Anki Desktop**.
4. The puller posts the result back (**Added / Duplicate / Failed**).
5. **AnkiAdd** polls the cloud for updates and shows the final status.

### Architecture diagram

```text
┌──────────────────┐         HTTPS          ┌──────────────────────────────┐
│   AnkiAdd (iOS)   │ ─────────────────────► │ API Gateway (HTTP API)       │
│  Upload + Polling │                        │  /v1/notes/* endpoints       │
└─────────┬────────┘                        └──────────────┬───────────────┘
          │                                                 │
          │                                                 ▼
          │                                      ┌──────────────────────────┐
          │                                      │ Lambda (AnkiAddApi)      │
          │                                      │ Auth + routing + logic   │
          │                                      └──────────────┬───────────┘
          │                                                     │
          │                                                     ▼
          │                                      ┌──────────────────────────┐
          │                                      │ DynamoDB (AnkiAddNotes)  │
          │                                      │ pending + results        │
          │                                      └──────────────┬───────────┘
          │                                                     ▲
          │                                                     │
          │                                   HTTPS              │
          │                         ┌────────────────────────────┘
          │                         │
          ▼                         │
┌──────────────────┐               │
│ Desktop Puller    │──────────────┘
│ (macOS script)    │   polls /pending, posts /result
└─────────┬────────┘
          │ localhost
          ▼
┌──────────────────┐
│ AnkiConnect       │  addNote / findNotes / guiBrowse
│ (Anki Desktop)    │
└──────────────────┘
```

---

### What data is uploaded to the cloud?

When you tap Sync with Cloud enabled, AnkiAdd uploads a note payload. The payload is intentionally minimal: it only contains what is needed to create an Anki note.

**Always included (typical):**

- Deck name (e.g., `Default`)
- Note type / model name (e.g., `Basic`)
- Fields dictionary (the actual Anki fields you configured), e.g.:
  - `Expression`
  - `ExpressionReading`
  - `ExpressionFurigana`
  - `MainDefinition` (definitions + examples, formatted HTML)
  - `ExpressionAudio` (often includes `[sound:...]` tag if used)
  - `PitchPosition` (if you use it)
- Tags list (Anki tags like `AnkiAdd`, plus any JMdict tags you map)
- `clientNoteId` (UUID generated by the app so it can track status later)
- timestamps (`createdAt`, `updatedAt`)

**Not included:**

- Your entire Anki collection
- Your Anki review history
- Other notes from your decks
- Your browsing history / unrelated clipboard data

> If you enabled “Watch clipboard”, AnkiAdd may prefill searches from clipboard text — but it still only uploads something when you explicitly add/sync a note.

---

### What gets stored in DynamoDB?

DynamoDB stores each uploaded note and its processing status.

Typical stored fields:

- `userId`
- `noteId` (cloudNoteId)
- `clientNoteId`
- `deck`, `model`
- `fields` (dictionary)
- `tags` (list)
- `status`: `PENDING` → `PROCESSING` → `ADDED | DUPLICATE | FAILED`
- `result` object:
  - `ankiNoteId` (if added)
  - `duplicateNoteIds` (if duplicate)
  - `error` (if failed)
- `updatedAt` (epoch seconds)

**Important:** by default, this means the definitions/examples you add are stored in DynamoDB under your AWS account.

---

### How authentication works (Bearer token)

Every API call requires:

```http
Authorization: Bearer <YOUR_TOKEN>
```

- The token is checked in Lambda (`API_TOKEN` env var).
- Your iOS app stores the token in Keychain (recommended).
- Your Desktop Puller reads the token from environment variables.

**Security note:** anyone with this token can submit notes to your cloud endpoint. Treat it like a password.

---

### Duplicate handling (what “Duplicate” means)

Duplicates are detected on the desktop side (AnkiConnect):

- The puller calls `addNote` with `allowDuplicate = false`
- If AnkiConnect rejects it as a duplicate:
  - The puller may call `findNotes` to locate existing notes
  - The puller reports back `DUPLICATE` with `duplicateNoteIds` (when available)

Your phone then shows:

- Duplicate (already in deck)
- (Optional UX) “Open in Anki” / “Open in AnkiMobile” actions

---

### Status polling (why your phone updates later)

Your phone does **not** talk to Anki Desktop directly. Instead:

- It uploads, then polls: `GET /v1/notes/status?since=<epoch>`
- When the desktop finishes, it posts results to the cloud
- The phone sees those results during the next polling cycle

---

### Privacy checklist (recommended)

Before enabling Cloud Sync:

- ✅ You understand that definitions/examples are stored in your AWS account
- ✅ You use a long random token and keep it private
- ✅ You set rate limits in API Gateway if you plan to share this setup
- ✅ You are okay with AWS costs (usually small, but not zero)

---

### Cost notes (rough)

Most users will pay very little, but costs depend on usage:

- API Gateway requests
- Lambda invocations
- DynamoDB reads/writes/storage

For personal use, this is typically low (a few dollars/month or less), but it depends on how often you sync and poll.

---

## AWS setup (step-by-step)

This section is a click-by-click guide to create the AWS backend in **Tokyo (ap-northeast-1)**:

- API Gateway (HTTP API)
- Lambda (Python)
- DynamoDB table
- (Recommended) DynamoDB GSI for fast `/pending`

> **Privacy note:** your Anki card content (word/definition/tags/etc.) will be stored in DynamoDB under your AWS account.

### Overview (what you are building)

You will deploy a private API with these endpoints:

- `POST /v1/notes/batch` — upload notes from iPhone
- `GET /v1/notes/pending?limit=...` — desktop puller fetches notes to add
- `POST /v1/notes/{id}/result` — desktop puller posts status back
- `GET /v1/notes/status?since=<epoch>` — iPhone polls results

All endpoints require:

```http
Authorization: Bearer <YOUR_TOKEN>
```

---

### Step 0 — Choose region + prepare values

Use Tokyo region:

- Region: `ap-northeast-1`

Prepare these values (you’ll paste them later):

- `API_TOKEN` — your long random token
- `USER_ID` — a simple identifier (example: `yoyo`)
- `TABLE_NAME` — use `AnkiAddNotes`

---

### Step 1 — Create DynamoDB table

1. AWS Console → DynamoDB
2. Left menu → Tables
3. Click **Create table**

Fill:

- Table name: `AnkiAddNotes`
- Partition key: `userId` (String)
- Sort key: `noteId` (String)

Leave other settings default (on-demand is fine for personal use).

✅ Wait until table status becomes **Active**.

---

### Step 2 — Add the GSI (recommended for performance)

Without an index, `/pending` becomes slower as your table grows. With this index, `/pending` stays fast.

1. DynamoDB → Tables → `AnkiAddNotes`
2. Tab: **Indexes**
3. Click **Create index**

Create a Global Secondary Index:

- Index name: `gsi_status_updatedAt`
- Partition key: `status` (String)
- Sort key: `updatedAt` (Number)
- Projection: `ALL`

✅ Wait for index status: **Active**.

---

### Step 3 — Create Lambda function (Python)

1. AWS Console → Lambda
2. Click **Create function**
3. Choose **Author from scratch**

Fill:

- Function name: `AnkiAddApi`
- Runtime: Python (latest available)
- Region: `ap-northeast-1` (Tokyo)

---

### Step 4 — Configure Lambda environment variables

Lambda → Configuration → Environment variables → **Edit**:

Add:

- `API_TOKEN` = your token
- `TABLE_NAME` = `AnkiAddNotes`
- `USER_ID` = `yoyo` (or your chosen ID)

---

### Step 5 — Attach IAM permissions (Lambda execution role)

Your Lambda must read/write DynamoDB (and query the GSI).

1. Lambda → Configuration → Permissions  
2. Under Execution role, click the role name (opens IAM)
3. IAM role page → Add permissions → Create inline policy
4. Go to JSON tab and paste (replace `ACCOUNT_ID`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AnkiAddNotesAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:Query",
        "dynamodb:UpdateItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem"
      ],
      "Resource": [
        "arn:aws:dynamodb:ap-northeast-1:ACCOUNT_ID:table/AnkiAddNotes",
        "arn:aws:dynamodb:ap-northeast-1:ACCOUNT_ID:table/AnkiAddNotes/index/*"
      ]
    }
  ]
}
```

5. Click Next
6. Policy name: `AnkiAddNotesIndexAccess`
7. Click **Create policy**

To find your AWS account id: AWS Console top-right → click your account → copy **Account ID**.

---

### Step 6 — Paste Lambda code

Lambda → Code tab → open `lambda_function.py`  
Replace the entire file with the code provided later in this README, then click **Deploy**.

---

### Step 7 — Create API Gateway (HTTP API)

1. AWS Console → API Gateway
2. Click **Create API**
3. Choose **HTTP API** → Build
4. Integrations → Add integration → **Lambda** → select `AnkiAddApi`

**Routes** (attach each route to the Lambda integration):

- `POST /v1/notes/batch`
- `GET /v1/notes/pending`
- `POST /v1/notes/{id}/result`
- `GET /v1/notes/status`

**Deploy**  
HTTP APIs often auto-deploy to `$default`. If auto-deploy is off, click **Deploy**.

Copy the **Invoke URL**:

```text
https://xxxx.execute-api.ap-northeast-1.amazonaws.com
```

---

### Step 8 — Test endpoints with curl

Set these:

```bash
INVOKE_URL="https://xxxx.execute-api.ap-northeast-1.amazonaws.com"
TOKEN="YOUR_TOKEN_HERE"
```

Test pending:

```bash
curl -i -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/pending?limit=5"
```

Expected:

```json
{"notes":[]}
```

Test status:

```bash
curl -i -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/status?since=0"
```

Expected:

```json
{"updates":[],"serverTime":1772340585}
```

---

### Troubleshooting AWS

- **401 Unauthorized**
  - Token in request does not match Lambda env var `API_TOKEN`
- **404 Not Found**
  - Route not created, or created on a different API than the invoke URL
- **500 Internal Server Error**
  - Check Lambda logs:
    - Lambda → Monitor → View logs in CloudWatch
  - Common causes:
    - Missing DynamoDB permissions
    - Syntax error in Lambda file
    - DynamoDB index not active / wrong name (for `/pending`)

---

## Backend code (Lambda) + API contract

This section provides:

- A complete `lambda_function.py` you can paste into AWS Lambda
- The HTTP API contract (requests + responses)
- Example `curl` calls for every endpoint
- Notes about security, duplicates, and performance

> **Important:** this backend stores note content (word/definition/tags/etc.) in DynamoDB under your AWS account.

### Required AWS resources

- **DynamoDB table:** `AnkiAddNotes`
  - Partition key: `userId` (String)
  - Sort key: `noteId` (String)
- **(Recommended) GSI:** `gsi_status_updatedAt`
  - PK: `status` (String)
  - SK: `updatedAt` (Number)
- **Lambda env vars:**
  - `API_TOKEN` = your Bearer token
  - `TABLE_NAME` = `AnkiAddNotes`
  - `USER_ID` = `yoyo` (or your identifier)

---

### Full Lambda code (`lambda_function.py`)

Paste this entire file into AWS Lambda → Code → `lambda_function.py`, then click **Deploy**.

```python
import os, json, time, uuid, boto3
from decimal import Decimal
from boto3.dynamodb.conditions import Key

API_TOKEN = os.environ.get("API_TOKEN", "")
TABLE_NAME = os.environ.get("TABLE_NAME", "")
USER_ID = os.environ.get("USER_ID", "yoyo")

ddb = boto3.resource("dynamodb")
table = ddb.Table(TABLE_NAME)

def _json_default(o):
    if isinstance(o, Decimal):
        return int(o) if o % 1 == 0 else float(o)
    raise TypeError(f"Object of type {o.__class__.__name__} is not JSON serializable")

def _resp(code, obj):
    return {
        "statusCode": code,
        "headers": {"content-type": "application/json"},
        "body": json.dumps(obj, default=_json_default),
    }

def _unauthorized():
    return _resp(401, {"error": "unauthorized"})

def _bad(code, msg):
    return _resp(code, {"error": msg})

def _ok(obj):
    return _resp(200, obj)

def _auth_ok(event):
    h = event.get("headers") or {}
    auth = h.get("authorization") or h.get("Authorization")
    if not auth or not auth.startswith("Bearer "):
        return False
    return auth.split(" ", 1)[1].strip() == API_TOKEN

def lambda_handler(event, context):
    if not _auth_ok(event):
        return _unauthorized()

    rc = event.get("requestContext", {}).get("http", {})
    method = rc.get("method", "")
    path = rc.get("path", "")
    qs = event.get("queryStringParameters") or {}

    # -------------------------
    # POST /v1/notes/batch
    # -------------------------
    if method == "POST" and path.endswith("/v1/notes/batch"):
        try:
            payload = json.loads(event.get("body") or "{}")
        except Exception:
            payload = {}

        notes = payload.get("notes") or []
        accepted, rejected = [], []
        now = int(time.time())

        for n in notes:
            client_id = (n.get("clientNoteId") or "").strip()
            if not client_id:
                rejected.append({"clientNoteId": "", "error": "missing clientNoteId"})
                continue

            cloud_id = str(uuid.uuid4())

            item = {
                "userId": USER_ID,
                "noteId": cloud_id,
                "clientNoteId": client_id,
                "createdAt": n.get("createdAt") or "",
                "deck": n.get("deck") or "Default",
                "model": n.get("model") or "Basic",
                "fields": n.get("fields") or {},
                "tags": n.get("tags") or ["AnkiAdd"],
                "status": "PENDING",
                "updatedAt": now,
                "result": {},
            }

            table.put_item(Item=item)
            accepted.append({
                "clientNoteId": client_id,
                "cloudNoteId": cloud_id,
                "status": "PENDING",
            })

        return _ok({"accepted": accepted, "rejected": rejected})

    # -------------------------
    # GET /v1/notes/pending?limit=25
    # Uses GSI when available; falls back to per-user query.
    # -------------------------
    if method == "GET" and path.endswith("/v1/notes/pending"):
        try:
            limit = int(qs.get("limit") or "25")
            limit = max(1, min(limit, 100))
            now = int(time.time())

            # Try GSI first (fast)
            try:
                resp = table.query(
                    IndexName="gsi_status_updatedAt",
                    KeyConditionExpression=Key("status").eq("PENDING"),
                    ScanIndexForward=True,
                    Limit=limit,
                )
                items = resp.get("Items", [])
                debug_used = "gsi"
            except Exception:
                # Fallback (still works without GSI)
                resp = table.query(
                    KeyConditionExpression=Key("userId").eq(USER_ID),
                    Limit=500,
                )
                items_all = resp.get("Items", [])
                items = [x for x in items_all if x.get("status") in ("PENDING", "PROCESSING")]
                items.sort(key=lambda x: x.get("updatedAt", 0))
                items = items[:limit]
                debug_used = "fallback"

            notes_out = []

            for x in items:
                if x.get("userId") != USER_ID:
                    continue

                notes_out.append({
                    "cloudNoteId": x["noteId"],
                    "clientNoteId": x.get("clientNoteId"),
                    "deck": x.get("deck"),
                    "model": x.get("model"),
                    "fields": x.get("fields"),
                    "tags": x.get("tags"),
                    "status": x.get("status"),
                })

                # Claim note: PENDING -> PROCESSING (avoid double-processing)
                if x.get("status") == "PENDING":
                    try:
                        table.update_item(
                            Key={"userId": x["userId"], "noteId": x["noteId"]},
                            UpdateExpression="SET #s=:s, updatedAt=:u",
                            ConditionExpression="#s = :pending",
                            ExpressionAttributeNames={"#s": "status"},
                            ExpressionAttributeValues={
                                ":s": "PROCESSING",
                                ":pending": "PENDING",
                                ":u": now,
                            },
                        )
                    except Exception:
                        pass

            return _ok({"notes": notes_out, "debug": debug_used})

        except Exception as e:
            return _resp(500, {"error": "pending crashed", "details": str(e)})

    # -------------------------
    # POST /v1/notes/{id}/result
    # -------------------------
    if method == "POST" and "/v1/notes/" in path and path.endswith("/result"):
        cloud_id = path.split("/v1/notes/")[1].split("/result")[0].strip("/")

        try:
            payload = json.loads(event.get("body") or "{}")
        except Exception:
            payload = {}

        status = (payload.get("status") or "").upper()
        if status not in ("ADDED", "DUPLICATE", "FAILED"):
            return _bad(400, "invalid status")

        result = {
            "ankiNoteId": payload.get("ankiNoteId"),
            "duplicateNoteIds": payload.get("duplicateNoteIds") or [],
            "error": payload.get("error") or "",
        }

        table.update_item(
            Key={"userId": USER_ID, "noteId": cloud_id},
            UpdateExpression="SET #s=:s, #r=:r, updatedAt=:u",
            ExpressionAttributeNames={"#s": "status", "#r": "result"},
            ExpressionAttributeValues={
                ":s": status,
                ":r": result,
                ":u": int(time.time()),
            },
        )

        return _ok({"ok": True})

    # -------------------------
    # GET /v1/notes/status?since=<epochSeconds>
    # Returns updates with updatedAt > since
    # -------------------------
    if method == "GET" and path.endswith("/v1/notes/status"):
        since = int(qs.get("since") or "0")

        resp = table.query(
            KeyConditionExpression=Key("userId").eq(USER_ID),
            Limit=500,
        )
        items = resp.get("Items", [])

        updates = []
        for x in items:
            if int(x.get("updatedAt", 0)) <= since:
                continue

            st = x.get("status")
            if st in ("PENDING", "PROCESSING"):
                continue

            r = x.get("result") or {}

            updates.append({
                "cloudNoteId": x["noteId"],
                "clientNoteId": x.get("clientNoteId"),
                "status": st,
                "updatedAt": int(x.get("updatedAt", 0)),
                "ankiNoteId": r.get("ankiNoteId"),
                "duplicateNoteIds": r.get("duplicateNoteIds") or [],
                "error": r.get("error") or "",
            })

        updates.sort(key=lambda u: int(u["updatedAt"]))
        return _ok({"updates": updates, "serverTime": int(time.time())})

    return _resp(404, {"message": "Not Found"})
```

---

### API Contract

All requests require:

- `Authorization: Bearer <YOUR_TOKEN>`
- `Content-Type: application/json`

#### 1) Upload notes (batch)

`POST /v1/notes/batch`

Body:

```json
{
  "notes": [
    {
      "clientNoteId": "5A779C6B-CF9A-44AE-AE13-4D5C048E20EE",
      "createdAt": "2026-03-01T00:00:00Z",
      "deck": "Default",
      "model": "Basic",
      "fields": {
        "Expression": "階段",
        "ExpressionReading": "かいだん",
        "MainDefinition": "<ol>...</ol>"
      },
      "tags": ["AnkiAdd", "news1k"]
    }
  ]
}
```

Response:

```json
{
  "accepted": [
    {
      "clientNoteId": "5A779C6B-CF9A-44AE-AE13-4D5C048E20EE",
      "cloudNoteId": "3273a1c3-9ad5-4b0d-b592-f8abade34696",
      "status": "PENDING"
    }
  ],
  "rejected": []
}
```

#### 2) Desktop puller fetches pending notes

`GET /v1/notes/pending?limit=25`

Response:

```json
{
  "notes": [
    {
      "cloudNoteId": "3273a1c3-9ad5-4b0d-b592-f8abade34696",
      "clientNoteId": "5A779C6B-CF9A-44AE-AE13-4D5C048E20EE",
      "deck": "Default",
      "model": "Basic",
      "fields": { "Expression": "階段", "MainDefinition": "<ol>...</ol>" },
      "tags": ["AnkiAdd"],
      "status": "PENDING"
    }
  ],
  "debug": "gsi"
}
```

> The `debug` field is for development. You can remove it later.

#### 3) Desktop puller posts result

`POST /v1/notes/{id}/result`

- Added:

```json
{ "status": "ADDED", "ankiNoteId": 1772337031243 }
```

- Duplicate:

```json
{ "status": "DUPLICATE", "duplicateNoteIds": [12345, 67890] }
```

- Failed:

```json
{ "status": "FAILED", "error": "AnkiConnect error message..." }
```

Response:

```json
{ "ok": true }
```

#### 4) Phone polls status updates

`GET /v1/notes/status?since=<epochSeconds>`

Response:

```json
{
  "updates": [
    {
      "cloudNoteId": "3273a1c3-9ad5-4b0d-b592-f8abade34696",
      "clientNoteId": "5A779C6B-CF9A-44AE-AE13-4D5C048E20EE",
      "status": "ADDED",
      "updatedAt": 1772337031,
      "ankiNoteId": 1772337031243,
      "duplicateNoteIds": [],
      "error": ""
    }
  ],
  "serverTime": 1772340585
}
```

The app should store the largest `updatedAt` and use it as the next sync.

---

### Example curl calls

Set these first:

```bash
INVOKE_URL="https://xxxx.execute-api.ap-northeast-1.amazonaws.com"
TOKEN="YOUR_TOKEN_HERE"
```

Upload:

```bash
curl -sS -X POST   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json"   -d '{"notes":[{"clientNoteId":"11111111-1111-1111-1111-111111111111","createdAt":"2026-03-01T00:00:00Z","deck":"Default","model":"Basic","fields":{"Expression":"階段","MainDefinition":"test"},"tags":["AnkiAdd"]}]}'   "$INVOKE_URL/v1/notes/batch"
```

Pending:

```bash
curl -sS -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/pending?limit=5"
```

Status:

```bash
curl -sS -H "Authorization: Bearer $TOKEN"   "$INVOKE_URL/v1/notes/status?since=0"
```

---

## Desktop Puller (script + LaunchAgent auto-run)

The Desktop Puller is the bridge between your cloud inbox and Anki Desktop:

- Polls your cloud for pending notes
- Adds them to Anki via **AnkiConnect** (localhost)
- Posts results back (**ADDED / DUPLICATE / FAILED**)

> The puller must run on the same computer where **Anki Desktop** is running.

### What you need (desktop)

- **Anki Desktop** installed
- **AnkiConnect** installed and working
- **Python 3.12+** recommended
- Internet connection (to reach your cloud API)

---

### Step 1 — Verify AnkiConnect is working

Make sure Anki Desktop is open, then run:

```bash
curl -sS -X POST "http://127.0.0.1:8765"   -H "Content-Type: application/json"   -d '{"action":"version","version":6}'
```

Expected:

```json
{"result":6,"error":null}
```

If this fails:

- Open Anki Desktop
- Reinstall AnkiConnect
- Restart Anki

---

### Step 2 — Install Python dependency

```bash
python3 -m pip install --user requests
```

---

### Step 3 — Run the puller manually (foreground)

From your puller folder (example: `~/ankiadd-puller`):

```bash
export ANKIADD_CLOUD_BASE_URL="https://xxxx.execute-api.ap-northeast-1.amazonaws.com"
export ANKIADD_CLOUD_TOKEN="YOUR_TOKEN_HERE"
export ANKICONNECT_URL="http://127.0.0.1:8765"

# tuning (recommended)
export ANKIADD_POLL_SECONDS="8"
export ANKIADD_BATCH_LIMIT="25"
export ANKIADD_HTTP_TIMEOUT="25"

./ankiadd-puller.py
```

Expected output:

```text
Starting AnkiAdd Puller
Cloud: https://xxxx.execute-api.ap-northeast-1.amazonaws.com
AnkiConnect: http://127.0.0.1:8765
AnkiConnect OK, version: 6
```

When it processes a note, you’ll see one of:

```text
ADDED: <cloudNoteId> -> <ankiNoteId>
DUPLICATE: <cloudNoteId> dup_ids: [...]
FAILED: <cloudNoteId> <error message>
```

---

### Step 4 — Make environment variables permanent (zsh)

If you don’t want to paste exports every time, add them to your `~/.zshrc`:

```bash
nano ~/.zshrc
```

Add (edit values):

```bash
# --- AnkiAdd Puller ---
export ANKIADD_CLOUD_BASE_URL="https://xxxx.execute-api.ap-northeast-1.amazonaws.com"
export ANKIADD_CLOUD_TOKEN="YOUR_TOKEN_HERE"
export ANKICONNECT_URL="http://127.0.0.1:8765"

export ANKIADD_POLL_SECONDS="8"
export ANKIADD_BATCH_LIMIT="25"
export ANKIADD_HTTP_TIMEOUT="25"
```

Reload:

```bash
source ~/.zshrc
```

---

### Step 5 — Run automatically in background (macOS LaunchAgent)

This starts the puller at login and restarts it if it crashes.

#### 5.1 Create the LaunchAgent plist

Create:

```bash
nano ~/Library/LaunchAgents/com.ankiadd.puller.plist
```

Paste the plist below.

> **IMPORTANT:** replace `/Users/YOUR_USERNAME` with your real path, and set the correct python path.

To find python:

```bash
which python3
python3 --version
```

Use the output path from `which python3` inside the plist.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>com.ankiadd.puller</string>

    <key>ProgramArguments</key>
    <array>
      <!-- Replace this with: which python3 -->
      <string>/Library/Frameworks/Python.framework/Versions/3.12/bin/python3</string>
      <!-- Replace this with your actual script path -->
      <string>/Users/YOUR_USERNAME/ankiadd-puller/ankiadd-puller.py</string>
    </array>

    <key>EnvironmentVariables</key>
    <dict>
      <key>ANKIADD_CLOUD_BASE_URL</key>
      <string>https://xxxx.execute-api.ap-northeast-1.amazonaws.com</string>

      <key>ANKIADD_CLOUD_TOKEN</key>
      <string>YOUR_TOKEN_HERE</string>

      <key>ANKICONNECT_URL</key>
      <string>http://127.0.0.1:8765</string>

      <key>ANKIADD_POLL_SECONDS</key>
      <string>8</string>

      <key>ANKIADD_BATCH_LIMIT</key>
      <string>25</string>

      <key>ANKIADD_HTTP_TIMEOUT</key>
      <string>25</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/ankiadd-puller.out.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/ankiadd-puller.err.log</string>
  </dict>
</plist>
```

#### 5.2 Load the LaunchAgent

```bash
launchctl load ~/Library/LaunchAgents/com.ankiadd.puller.plist
```

To restart after changes:

```bash
launchctl unload ~/Library/LaunchAgents/com.ankiadd.puller.plist
launchctl load ~/Library/LaunchAgents/com.ankiadd.puller.plist
```

#### 5.3 View logs

```bash
tail -f /tmp/ankiadd-puller.out.log
tail -f /tmp/ankiadd-puller.err.log
```

#### 5.4 Stop / uninstall

```bash
launchctl unload ~/Library/LaunchAgents/com.ankiadd.puller.plist
rm ~/Library/LaunchAgents/com.ankiadd.puller.plist
```

---

## Configure AnkiAdd + troubleshooting + FAQ

Open **AnkiAdd → Settings → Cloud** and set:

### 1) Sync Method

Set **Sync Method** to **Cloud**.

### 2) Cloud Base URL

Paste your API Gateway **Invoke URL**, for example:

```text
https://xxxx.execute-api.ap-northeast-1.amazonaws.com
```

Notes:

- Do not include `/v1/...`
- Prefer no trailing slash
- Must start with `https://`

### 3) API Token (Bearer token)

Paste your token and save it.

- Stored securely using Keychain (recommended)
- Treat it like a password

### 4) Auto-fetch cloud results

Turn this **ON** to automatically show:

- **Added ✓**
- **Duplicate**
- **Failed**

This works by polling:

`GET /v1/notes/status?since=<epoch>`

### 5) Poll interval (recommended)

If you can choose a polling interval:

- **10–20 seconds**

Faster polling costs more requests; slower polling delays status updates.

---

### Understanding the Sync panel statuses

**“Uploaded • waiting for desktop”**  
Your phone successfully uploaded the note to the cloud.

What must happen next:

- Desktop Puller must be running
- Anki Desktop must be open
- The puller will fetch `/pending` and add via AnkiConnect

**“Added ✓”**  
The Desktop Puller successfully added your note to Anki Desktop.

**“Duplicate”**  
AnkiConnect rejected the add because the note already exists (or you configured duplicates to be blocked).

**“Failed”**  
The puller couldn’t add the note.

Common causes:

- Anki Desktop not open
- AnkiConnect error
- Invalid deck/model/field mapping
- Audio upload / field formatting errors (if enabled)

---

### Common troubleshooting

**Problem: “Uploaded” but it never becomes “Added”**

Checklist (most common cause: desktop is not running):

- Desktop Puller running?
- Anki Desktop open?
- AnkiConnect working?

Test AnkiConnect:

```bash
curl -sS -X POST "http://127.0.0.1:8765"   -H "Content-Type: application/json"   -d '{"action":"version","version":6}'
```

**Problem: Puller prints timeouts**

- Increase puller HTTP timeout:

```bash
export ANKIADD_HTTP_TIMEOUT="25"
```

- In AWS Lambda, consider:
  - Timeout: **10–15 seconds**
  - Memory: **512 MB**

**Problem: API returns 401 Unauthorized**

- Token mismatch:
  - Lambda env var `API_TOKEN` must match
  - Puller env var `ANKIADD_CLOUD_TOKEN` must match
  - AnkiAdd Settings token must match

**Problem: API returns 404 Not Found**

- Routes were created in a different API than the Invoke URL you are calling
- Confirm API Gateway routes exist:
  - `POST /v1/notes/batch`
  - `GET /v1/notes/pending`
  - `POST /v1/notes/{id}/result`
  - `GET /v1/notes/status`

**Problem: API returns 500 Internal Server Error**

- Check CloudWatch logs:
  - Lambda → Monitor → View logs in CloudWatch

Common causes:

- Syntax/indentation error in Lambda code
- Missing DynamoDB permissions
- Decimal JSON serialization issue (handled in this repo)
- GSI not active / wrong name (if using GSI path)

**Problem: Notes stuck in PROCESSING**

This usually means:

- The puller claimed the note but crashed before posting result  
  **or**
- The puller is running multiple copies and conflicts

Fix:

- Restart the puller
- Ensure only one puller instance is running
- (Hardening idea) add a “processing timeout” rule in Lambda

---

### FAQ

**Does this work if my phone is not on the same Wi‑Fi as my computer?**  
Yes — that is the whole point of Cloud Sync.

**Is this safe?**  
It can be safe if you keep your token private and use HTTPS (default).  
But remember: your card content is stored in your AWS account, and anyone with the token can submit notes.

**How much does it cost?**  
Typically low for personal use, but depends on usage. AWS charges for API Gateway requests, Lambda invocations, and DynamoDB reads/writes/storage.

**Can I use this on Windows?**  
Yes, but the LaunchAgent is macOS-specific. You would run the puller manually or via Windows Task Scheduler.

**Can I share this setup with friends?**  
Technically yes, but you should add per-user auth (not a single shared token), rate limits, and admin tooling.

---

## License

MIT
