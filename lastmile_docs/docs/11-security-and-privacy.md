# Security and Privacy

## Authentication
- Google OAuth selected.
- Flutter sends Google ID token to NestJS.
- Verify signature, issuer, audience, expiry and `sub`.
- Use Google `sub` as stable external identity key.
- NestJS issues app access/refresh tokens.
- Store app tokens securely (`flutter_secure_storage`).
- Rotate and revoke refresh tokens; store protected token metadata.
- Logout revokes session.

## Authorization
- Role selection at app entry is a UX choice, not authorization.
- Backend loads persisted roles and driver approval.
- Passenger sees own bookings/offers/rides.
- Driver acts only on offers/assignments belonging to them.
- Driver must be approved and not suspended.
- Admin endpoints require ADMIN role and resource checks.
- Driver cannot approve self.

## Driver documents
Restrict document references/content to driver and authorized admins. Avoid returning licence/document details to passengers. Record reviewer, decision, timestamp and reason.

## Location privacy
Request location only for pickup validation/active trip. Only ride participants and authorized admins can see live location. Persist periodic snapshots, not every high-frequency sample. Define retention before production. Show last-updated timestamp when GPS is stale.

## API/socket security
- DTO validation and server-side authorization.
- Rate-limit sensitive endpoints.
- Admin web CORS allowlist.
- Verify resource ownership.
- Never trust client fare, role, approval or state.
- Authenticate Socket.IO; authorize room joins.
- Validate driver assignment before accepting GPS.
- Rate-limit GPS and reject out-of-order points.
- Avoid stack traces/secrets in errors/logs.

## Secret hygiene
Never commit OAuth secrets, JWT secrets, DB/Redis credentials, Apple signing assets, `.env` with secrets, personal driver documents or real location dumps. Commit `.env.example` placeholders only. Rotate leaked secrets.
