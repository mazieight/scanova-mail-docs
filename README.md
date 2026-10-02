# Scanova Mail

Independent local Next.js / TypeScript / PostgreSQL mail workspace. No VPS, DNS, or Receiving changes are made.

## Run locally

1. `npm install`
2. `node scripts/setup.cjs` (generates `.env` and a random local admin password)
3. `docker compose up -d`
4. `npm run migrate`
5. `npm run dev`

Open http://localhost:3105. Sign in with `ADMIN_EMAIL` and `ADMIN_PASSWORD` from `.env`. The password is hashed when the administrator is first initialized; editing the environment password later does not reset the existing user.

Set `RESEND_API_KEY` in `.env` and restart Next.js to enable sending. Never use a NEXT_PUBLIC variable for credentials. Set `REAL_TEST_TO` to an inbox you control and run `npm run test:real` once to test delivery. Missing credentials never trigger a simulated successful send.

## Verification

`npm run lint`, `npm run typecheck`, `npm run migrate`, `npm test`, `npm run test:browser`, `npm run build`.

Automated QA uses a separate local `scanova_mail_test` database and, for browser tests, a separate loopback app at port 3106. Each run generates test authentication and webhook credentials and removes live Resend credentials. Tests reset only that test database; fixtures never populate the normal Inbox. Mock body and outbound transports require `SCANOVA_TEST_MODE=true`, a loopback database name ending `_test`, and non-production mode. The normal workspace never accepts mock email bodies. `npm run cleanup:fixtures` removes only identified old automated Inbox/contact fixtures and preserves all outbound records.

Production signature checking uses the raw request body before JSON parsing. Inbound metadata triggers Resend body and attachment retrieval. Files are streamed from URLs obtained through the authenticated Resend attachment API, and bytes are saved with the message in one PostgreSQL transaction. Expiring download URLs are not used for later user downloads. Users download durable stored files through authenticated endpoints. Attachment failures roll back ingestion, record a durable failure count, and return a retryable 503. Duplicate event IDs, provider IDs and Message-IDs are protected, including concurrent duplicate events.

HTML is sanitized on ingestion and rendered in a sandboxed iframe with a restrictive CSP; remote images, scripts and trackers are blocked. Outgoing attachments are stored in PostgreSQL and sent through Resend (five files, 5 MB total). Inbound storage allows up to 100 files and 50 MB total per message. Downloads use attachment disposition and no-store caching. Recipient validation supports comma and semicolon separated addresses with optional CC/BCC, readable inline errors, and server validation. Raw database, provider and validation error objects are never rendered in the interface.

The workspace is a single administrator shared mailbox. Sends retain their request ID across retries, validate the payload fingerprint, and use provider idempotency keys. Failures are persisted and duplicate failures never claim success. Ambiguous network outcomes remain `sending`; inspect Resend before creating another send. Recovery with the original request ID is allowed inside the provider's idempotency window. Sent Message-IDs are retrieved from Resend and enriched by signed status events. Incoming replies match case-insensitive In-Reply-To and References headers; late parent arrival repairs the conversation. Status events received before the send response are reconciled from durable webhook storage.

Before later deployment: use production secrets and HTTPS, leave test mode disabled, configure the real webhook signing secret and Receiving intentionally, and QA actual inbound mail (Receiving remains disabled during local work). Rate limits are persisted in PostgreSQL. The app binds only to loopback locally. No VPS or DNS configuration is included.
