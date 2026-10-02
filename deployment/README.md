# Production deployment

Deployed at `https://mail.scanova.ma` in `/home/PROJECT_USER/scanova-mail`. Docker Compose runs the app on `127.0.0.1:3115` and PostgreSQL on a private container network. Existing VPS services are unchanged. Caddy serves valid public HTTPS.

Resend Sending and the signed production webhook work. The controlled real email arrived in the Scanova Gmail Inbox with an attachment, and delivery status persisted. Receiving is enabled in the Resend account but awaits the exact DNS record returned by Resend: `MX @ 10 inbound-smtp.eu-west-1.amazonaws.com`, TTL Auto/default. Existing sending DNS records remain verified. Real inbound and reply QA remain pending that MX record.

Production credentials live only in the server's `deployment/production.env` with permission 0600; never copy that file into source control. The temporary admin password is provided separately and must be replaced on first login. `production-qa.json` records non-secret QA evidence. Container restart policies and Docker autostart are enabled; message, contact, and attachment persistence across PostgreSQL/application restart passed.

The recommended target is `/home/PROJECT_USER/scanova-mail`. First run the read-only audit, inspect the active Caddy configuration privately, record existing application health, and select a free loopback port. Back up each existing production file before changing it. The site snippet must be added to the existing Caddy configuration, validated, and gracefully reloaded; it must not replace existing sites.

The Docker production candidate uses a dedicated database container, dedicated non-superuser application role, private internal database network, isolated named volume, and app port published only on `127.0.0.1`. The app has an egress network for Resend HTTPS. No SMTP services are installed. Container restart policies require Docker itself to be enabled at reboot, which must be verified on the actual VPS.

Generate new strong random database, admin, and session credentials on the server. Store them in a file outside Git with mode 0600. Use the existing Resend key only server-side. The `production.env.example` file contains no working production credentials. The app container receives neither the DB bootstrap credential nor the initial admin password. First login forces the temporary admin password to be changed, and password changes revoke all previous sessions.

After the VPS audit, using the protected environment file:

1. Build the production image. Its build stages run lint, TypeScript and Next.js build.
2. Start only the isolated database service.
3. Run the migration profile once to initialize the dedicated database and production account.
4. Run isolated mail and browser tests using disposable test infrastructure, not the production database.
5. Start the app and verify loopback health, authentication, upload limits, and persistence across app restart.
6. Add only the mail site to Caddy after a backup and successful validation. Check existing sites before and after a graceful reload.
7. Verify public DNS, external reachability and the TLS certificate before claiming HTTPS success.
8. Create the Resend webhook for the verified public endpoint; securely save its returned signing secret, restart only the mail app, and verify signatures and duplicate handling.
9. Enable Receiving through the existing Resend domain only after readiness; apply only the exact Receiving records returned by Resend. Preserve DKIM and the existing send.scanova.ma records.
10. Perform the authorized controlled outbound and inbound Gmail/reply tests. Confirm provider delivery status, durable message/attachment storage, and no fake production fixtures.

Do not copy the local `.env`, local database, or local administrator credentials to production. The source archive excludes local secrets and generated builds. Successful local validation does not imply successful VPS deployment or external HTTPS.
