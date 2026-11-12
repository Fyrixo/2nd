# Good Things Club

The site runs in a local preview by default. To make accounts, posts, reactions, and AI helpers work across visitors, connect it to a Supabase project and deploy the Edge Functions.

## Set up the shared community

1. Create a Supabase project and make sure Email sign-in is enabled.
2. From this folder, install the Supabase CLI if needed, then link and apply the database migration:

   ```powershell
   supabase login
   supabase link --project-ref YOUR_PROJECT_REF
   supabase db push
   ```

3. In Supabase **Project Settings → API**, copy the project URL and the public `anon`/publishable key into `supabaseConfig` near the bottom of `index.html`. These values are intended to be public; never put the `service_role` key in the website.
4. In **Edge Functions → Secrets**, set `OPENAI_API_KEY`, `OPENAI_MODEL` (for example `gpt-4o-mini`), and a long random `AI_CRON_SECRET`. Supabase supplies the project URL and service-role key to Edge Functions.
5. Deploy the AI functions:

   ```powershell
   supabase functions deploy community-ai
   supabase functions deploy quiet-room-host --no-verify-jwt
   ```

6. In the Supabase SQL Editor, enable the scheduler extensions and store the same random cron secret in Vault. Replace the example project reference and secret locally before running this SQL; do not commit real secrets.

   ```sql
   create extension if not exists pg_cron with schema pg_catalog;
   create extension if not exists pg_net with schema extensions;
   select vault.create_secret('USE_THE_SAME_LONG_RANDOM_VALUE_AS_AI_CRON_SECRET', 'good-things-ai-cron');

   select cron.schedule(
     'good-things-quiet-room-host',
     '*/30 * * * *',
     $$
       select net.http_post(
         url := 'https://YOUR_PROJECT_REF.supabase.co/functions/v1/quiet-room-host',
         headers := jsonb_build_object(
           'Content-Type', 'application/json',
           'x-cron-secret', (
             select decrypted_secret
             from vault.decrypted_secrets
             where name = 'good-things-ai-cron'
           )
         ),
         body := '{}'::jsonb
       );
     $$
   );
   ```

7. Host `index.html` over HTTPS (for example, with a static-site host) and add that domain to the Supabase **Authentication → URL Configuration** site and redirect URL settings. Email confirmation may need an email provider configured in Supabase.

The quiet-room host checks every 30 minutes. It creates one shared AI kindness prompt per day and posts only after a room has had no recent activity, at most once per room every four hours. AI-generated posts are labeled as AI. Members can ask Sunny AI questions and get AI-powered mini-game ideas; interactive AI requests are limited to five per member per minute.

## Privacy and safety

The AI helper sends the member's prompt to OpenAI using the server-side API key. The key is never sent to the browser. Do not enter private or sensitive information into the AI chat. Supabase row-level security keeps account actions tied to the signed-in member; community posts and reactions are readable by everyone.

The sample conversations in local preview are examples, not real members. Real shared activity appears only after the Supabase project and functions are configured.
