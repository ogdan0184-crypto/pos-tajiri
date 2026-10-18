# Supabase account setup

Tajiri POS uses Supabase Auth for email/password sign-in and the Postgres
`public.users` and `public.system` tables for profiles, roles, and the
first-admin bootstrap state. Passwords are managed by Supabase Auth and are not
stored in POS profile records or browser storage.

The browser uses `supabase-shim.js` to provide the compatibility interface
expected by the existing POS authentication code. The `create-staff` Edge
Function is required for admin-created accounts; it verifies the administrator,
creates the Auth account, and writes the profile through a restricted database
function.

## Configure a Supabase project

1. Create a Supabase project at [supabase.com](https://supabase.com).
2. In the Supabase SQL editor, run [`supabase-schema.sql`](./supabase-schema.sql)
   first, then run [`supabase-saas-migration.sql`](./supabase-saas-migration.sql).
   The second script assigns existing profiles to a legacy shop, creates
   tenant-isolated shop memberships and shared POS storage, and replaces the
   one-time administrator bootstrap with verified shop signup. Review the
   migration and take a database backup before applying it to a production
   project.
3. In **Authentication → Providers**, enable **Email** and keep email
   confirmation enabled. Set the allowed redirect URLs in
   **Authentication → URL Configuration** to include the POS deployment URL
   and the local development URL you use.
4. Copy the **Project URL** and **anon/public key** from **Settings → API**.
   Put these public client values in the Supabase configuration near the top of
   `index.html`:

   ```js
   window.TAJIRI_SUPABASE_CONFIG = {
     url: "https://YOUR-PROJECT.supabase.co",
     anonKey: "YOUR-ANON-PUBLIC-KEY",
   };
   ```

   Do not put the `service_role` key in `index.html` or any client-side file.
5. Install and authenticate the Supabase CLI, then link it to this project and
   deploy the staff function from the project directory after both SQL scripts
   have completed:

   ```text
   supabase login
   supabase link --project-ref YOUR_PROJECT_REF
   supabase functions deploy create-staff
   ```

   Supabase provides the function's `SUPABASE_URL`, `SUPABASE_ANON_KEY`, and
   `SUPABASE_SERVICE_ROLE_KEY` server-side environment variables. Never expose
   or copy the service-role key into the browser.
6. Open the Admin page and choose **Start free trial**. The owner verifies
   their email and signs in to create a shop workspace. The database assigns
   the shop owner a 14-day trial and a KES 100 monthly plan record. Creating
   that record does not collect payment or enforce subscription expiry.
7. Use **Staff access** in the Admin workspace to create cashier and
   administrator accounts. The Edge Function creates the Supabase Auth login
   and profile inside the administrator's shop. Row-level security limits
   profile reads and POS state to the signed-in user's shop.

If the Supabase values are not configured, the app can still use its existing
device-local account mode; that mode does not authenticate against Supabase or
synchronize accounts between devices.

## Existing local accounts and POS data

Accounts created before cloud authentication exist only in that browser's
local storage. Their password hashes cannot be converted to Supabase passwords.
Create a verified cloud shop account, then recreate staff accounts from the
Admin workspace and assign each a password.

Cloud POS data is stored in a revision-checked JSON document per shop. A
stale-device write is rejected rather than overwriting a newer cloud revision;
the POS reports when a change remains only on the current device. This is an
initial cloud-sync layer, not transactional multi-register inventory
accounting. Do not use it for concurrent real-money tills until transactional
sales and stock updates are implemented and validated.

The current M-Pesa configuration is sandbox only. The trial and price records
are not a billing service, and subscriptions are not charged or gated. Live
launch requires a production-approved Daraja account, a secure server-side
subscription/payment workflow, subscription enforcement, and a public HTTPS
hosting environment. Never expose the local `server.js` payment API directly
to the internet.
