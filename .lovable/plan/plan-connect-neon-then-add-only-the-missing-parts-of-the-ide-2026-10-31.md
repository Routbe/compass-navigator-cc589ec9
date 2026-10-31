# Plan: Connect Neon, then add only the missing parts of the identity broker

## Already built (will be extended, not rebuilt)
- Per-app email choice: `app_email_assignments` (db/0136), email forwarding (0137), private relay addresses (0138), and `email-broker.ts`/`.server.ts`, including how userinfo picks the email.
- Per-app sybil nullifier (`oidc-identifiers.ts`) and the per-app subject (`oauth/subject.ts`, `ROUT_PROVIDER_PAIRWISE_SECRET`).
- Wallet passes (`src/lib/wallet/*`, `WalletDialog`, a wallet option in `ShareMenu`).
- Step-up verification (`auth/step-up*`, `oauth/step-up.ts`).
- Footer pages already exist: about, contact, sovereignty, manifesto, explore, press, privacy, terms, data-deletion.
- The newest migration is 0138. The numbers 0135 and 0136 in the brief are already taken, so new files start at **0139**.

## Step 0: Connect Neon
- Link the Neon connector, which adds the NEON_API_KEY used by `src/lib/neon.ts`.
- Use `db:migrate:gateway` to apply any pending migrations, then refresh the snapshot and types.

## Step 1: Granular scopes and the disclosure log (`db/0139_granular_identity_scopes.sql`)
- Add `allowed_scopes text[]`, `is_trusted_kyc_partner bool`, `backchannel_webhook_url` and `backchannel_webhook_secret_sealed` to `oauth_clients`. The webhook secret is sealed the same way the signing keys are, never stored in plain text. This replaces the separate `oidc_client_webhooks` table.
- Add a new `identity_disclosure_logs` table: append-only, with RLS through a single `rout_scope` policy and grants for `rout_app`.
- Pure `scope-rules.ts` defines `openid`, `profile:read`, `sybil:verify`, `identity:legal_name` and `identity:national_id`, and decides which claims are allowed. `national_id` is refused with a 403 unless the app is a trusted partner, the user is tier 2, and a fresh step-up has passed.
- `oidc-identifiers.server.ts` provides `getPairwiseSubject`, `getAppSybilNullifier` and `generateUserInfoResponse`. These reuse `subject.ts` and `appSybilNullifier`; `ROUT_SYBIL_PEPPER` is new.
- Wire this into the existing userinfo route, and write a log entry for every sensitive disclosure.

## Step 2: Consent screen (`/oauth/authorize`)
- Show the app's trust status, a switch for each optional claim, a lock on the national ID, and the existing email picker.

## Step 3: Studio account page
- Add a "Gekoppelde Apps & E-mail Beheer" section that shows each app's email and scopes, using the existing connected-apps functions.
- Add a bulk email-update modal with "Selecteer alles" that opens when the primary email changes.
- Add an account merge wizard, protected by step-up. The merge moves profile data, links and credentials into one account and keeps the immutable subject.

## Step 4: Back-channel sync (`email-sync-webhook.server.ts`)
- Send `user.email_updated` to each app's webhook, with the body `{event, sub (pairwise), previous_email, new_email, updated_at}`.
- Sign it with HMAC-SHA256 and run it in the background, so it never blocks the user.
- The webhook URL must pass `safe-redirect`-style checks: https only, no private IP addresses.

## Step 5: Last.fm relay
- `src/lib/integrations/lastfm.server.ts` fetches from Last.fm with a 30-second cache. Visitors read it only through `/api/public/lastfm/$handle`, with opt-in and a public profile required.
- Add a CSS equalizer "Luistert nu" card, and move Last.fm from "soon" to available. `LASTFM_API_KEY` is new.

## Step 6: Footer pages
- Review and update the copy (AGPLv3, European hosting, manifesto) on the existing pages and in the README.
- Check that contact uses sanitizing and rate limits, and that data deletion uses the existing account-deletion flow.
- No new routes.

## Verification
- `tsgo --noEmit`, plus the full Vitest suite.
- New tests:
  - pairwise subjects differ between apps
  - the nullifier stays the same within one app and differs between apps
  - the national_id 403 rules
  - userinfo returns the email assigned to that app
  - the bulk update changes every selected record
  - the webhook signature is correct
- Record the new rules in AGENTS.md.

## Needs from you during the build
- Connect Neon (approval card).
- Secrets: `ROUT_SYBIL_PEPPER` (I can generate it) and `LASTFM_API_KEY` (from last.fm/api).
