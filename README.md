# Automated Multi-Platform Posting API

A small FastAPI service for sending one short message to several configured platforms from a single request. The API accepts only the message; the destinations are selected by the service configuration. Each platform is handled independently, so one failed integration does not prevent the others from being attempted.

**Platforms:** Bluesky, Mastodon, Discord, Matrix and Stoat  
**Endpoint:** `POST http://localhost:4000/posting`  
**API docs:** `http://localhost:4000/docs`

## Run locally

```bash
git clone <repository-url>
cd automated-posting
python -m venv .venv
```

Activate the environment, then install the dependencies:

```bash
# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate

python -m pip install -r requirements-dev.txt
```

Copy `.env.example` to `.env` and fill in credentials for the platforms you plan to use. A platform with missing or invalid configuration is reported as failed; it does not prevent the API from starting.

```bash
# Windows PowerShell
Copy-Item .env.example .env

# macOS / Linux
cp .env.example .env
```

Start the API on the port specified in the practical-test brief:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 4000
```

Open `http://localhost:4000/docs` to try the endpoint.

## API contract

### `POST /posting`

Request:

```json
{
  "message": "Check out our latest business platform and grow your online visibility. https://example.com"
}
```

Only `message` is accepted. The client cannot choose destinations in the request. By default, the service attempts all five configured platforms. An operator can select a subset with the server-side `POSTING_PLATFORMS` environment variable, for example:

```env
POSTING_PLATFORMS=bluesky,mastodon,discord
```

Invalid extra fields are rejected with HTTP `422`.

### Validation

- The message must contain 1–200 characters.
- Exactly one HTTP or HTTPS CTA URL must be present.
- The URL must have a valid public hostname and port.
- URL credentials, localhost, private and link-local addresses are rejected.
- A bare `www.` address is not accepted; include `https://`.

Validation errors use FastAPI's standard `422` response.

### Response shape

```json
{
  "success": true,
  "results": [
    {
      "platform": "mastodon",
      "status": "success",
      "posted_url": "https://mastodon.social/@demo/113001",
      "public_access": true,
      "date": "2026-11-19",
      "time": "10:30:00",
      "error": null
    },
    {
      "platform": "matrix",
      "status": "failed",
      "posted_url": null,
      "public_access": false,
      "date": null,
      "time": null,
      "error": "Platform not configured: set MATRIX_HOMESERVER_URL, MATRIX_ROOM_ID, MATRIX_ACCESS_TOKEN"
    }
  ]
}
```

The response above is an example of the shape, not evidence of a live post. The full examples are in `docs/examples/`.

A valid request returns a result for each selected platform. The top-level `success` value is `true` when at least one platform accepts the post. A post can have `status: "success"` and `public_access: false`: publishing and public readability are separate outcomes. `date` and `time` are UTC. They use the timestamp returned by the platform when one is available; otherwise the adapter records the local UTC submission time.

## Platform behavior

| Platform | Publishing method | How `public_access` is determined |
|---|---|---|
| Bluesky | AT Protocol `createRecord` | Anonymous read from the public AppView; the returned text is checked against the submitted message. |
| Mastodon | `POST /api/v1/statuses` with public visibility | Anonymous `GET` of the created status; its visibility must be `public`. |
| Discord | Incoming webhook with `wait=true` | Set to `false`; the adapter does not treat a channel link as anonymous public access. |
| Matrix | `PUT .../send/m.room.message/{txnId}` | The room's history visibility is checked. `world_readable` means non-members can read events; the join rule is a separate setting. This is a room-policy check, not a separate guest-account read test. |
| Stoat | Bot-authenticated channel message | Set to `false` conservatively; this adapter does not perform a separate public-read check. The returned URL points to the specific message in the channel. |

The public-access value is intentionally conservative. A message being accepted by an API does not, by itself, prove that another person can read it.

## Implementation decisions

- **Keep destination selection on the server.** The request body stays small and predictable: it contains only the message. The operator controls enabled destinations through environment configuration.
- **Run platform calls concurrently, but keep their outcomes separate.** A slow or unavailable destination should not block the other integrations. Each adapter has a timeout and its own error boundary.
- **Separate posting success from public readability.** Some services accept a message even when an unauthenticated or non-member user cannot read it. The response reports those facts separately rather than calling every accepted post public.
- **Do not automatically retry write requests.** A timeout can happen after the remote service has accepted a message. Retrying without knowing the outcome can create duplicates. Mastodon receives an idempotency key, but this service does not claim exactly-once delivery across client retries.
- **Keep API-specific code inside adapters.** Authentication, request formats and response parsing belong to the platform adapter. `PostingService` handles fan-out, timeouts, result formatting and safe error messages.
- **Do not follow HTTP redirects on authenticated platform calls.** This avoids forwarding credentials or webhook secrets to a different host after a redirect.

## Project layout

```text
automated-posting/
├── app/
│   ├── main.py                 # FastAPI routes and lazy service dependency
│   ├── schemas.py              # Request and response models
│   ├── validation.py           # Message and CTA URL validation
│   ├── config.py               # Server-side platform selection and timeout
│   ├── errors.py               # Safe platform error type
│   ├── platforms/              # Shared adapter base and five integrations
│   └── services/
│       ├── factory.py          # Builds configured adapters
│       └── posting_service.py  # Concurrent fan-out and result aggregation
├── tests/                      # Mocked API, validation and adapter tests
├── scripts/live_smoke.py       # Optional real-post smoke test
├── docs/
│   ├── examples/               # Sample request and response
│   └── EVIDENCE.md             # Live evidence checklist
├── .github/workflows/tests.yml # CI test workflow
├── .env.example
├── requirements.txt
└── requirements-dev.txt
```

## Tests and CI

Run the automated suite locally:

```bash
python -m pytest -q
```

The adapter tests use `httpx.MockTransport`; they do not send messages to real accounts. The suite checks request formats, validation, token/session handling, anonymous verification behavior, partial failures, timeout handling and error sanitization.

A GitHub Actions workflow is included for Python 3.10, 3.11 and 3.12. It runs the same test command on pushes and pull requests. The workflow result should be checked in the repository after the code is pushed; adding the workflow file here does not mean a remote run has already passed.

For a real publishing check, configure `.env` and run:

```bash
python -m scripts.live_smoke --yes
```

This command publishes an actual message to each enabled, configured platform. Use a test channel or account. It requires the explicit `--yes` flag to avoid accidental posts.

## Configuration and security

- Keep tokens, app passwords and webhook URLs in environment variables or the ignored `.env` file.
- Commit `.env.example`, never `.env`.
- Discord webhook URLs are credentials; do not paste them into logs, tickets or screenshots.
- Platform HTTP clients do not follow redirects.
- If a credential has been committed or shared, revoke it and issue a replacement.

The reasoning behind the request contract, partial-failure behavior and visibility checks is recorded in [`docs/IMPLEMENTATION_NOTES.md`](docs/IMPLEMENTATION_NOTES.md).

## API references

The integrations were implemented against the platforms' published API documentation:

- [Bluesky: create a session](https://docs.bsky.app/docs/api/com-atproto-server-create-session) and [create a record](https://docs.bsky.app/docs/api/com-atproto-repo-create-record)
- [Mastodon: create a status](https://docs.joinmastodon.org/methods/statuses/#create)
- [Discord: execute a webhook](https://discord.com/developers/docs/resources/webhook#execute-webhook)
- [Matrix: Client-Server API specification](https://spec.matrix.org/latest/client-server-api/), including room events and history visibility
- [Stoat: developer API documentation](https://developers.stoat.chat/category/api/) and [authentication](https://developers.stoat.chat/developers/api/authentication.html). The message URL follows the [`Message.path`](https://github.com/stoatchat/javascript-client-sdk/blob/main/src/classes/Message.ts) and [`Channel.path`](https://github.com/stoatchat/javascript-client-sdk/blob/main/src/classes/Channel.ts) definitions in the official client SDK.

## Known limitations

- There is no queue or persistent delivery record. Sending the same API request twice results in two independent posting attempts.
- The service does not guarantee exactly-once delivery across a network timeout.
- Bluesky and Mastodon have anonymous-read checks. Matrix reports the configured room's current history-visibility policy. Discord and Stoat are reported conservatively as not publicly verified by this adapter.
- Live publishing and external visibility must still be checked with real platform accounts before submission.
