# Database Restore: AWS Backup → Neon Dev Branch

How to pull the latest production backup from the AWS server and restore it into the Neon **development** branch for local work.

---

## Prerequisites (one-time setup)

- SSH key at `~/paperspace_key`
- `psql` installed locally
- Local folder exists: `/home/rocky/code/iqra-wave/quran-wave/db-backup/`
- Neon `development` branch exists in the project

---

## Step 1 — Download the latest backup from server

Run **locally** (no need to SSH in first):

```bash
set LATEST (ssh -i ~/paperspace_key iqra_rocky@44.196.233.7 'ls -t ~/QuranWave.Api/backups/*.sql | head -1')
scp -i ~/paperspace_key "iqra_rocky@44.196.233.7:$LATEST" /home/rocky/code/iqra-wave/quran-wave/db-backup/
```

You'll be prompted for your server password (twice — once per command).

### Want a specific file instead of the latest?

List them first:

```bash
ssh -i ~/paperspace_key iqra_rocky@44.196.233.7 'ls -lh ~/QuranWave.Api/backups/'
```

Then copy by name:

```bash
scp -i ~/paperspace_key iqra_rocky@44.196.233.7:~/QuranWave.Api/backups/pre-deploy-YYYYMMDD-HHMMSS.sql /home/rocky/code/iqra-wave/quran-wave/db-backup/
```

---

## Step 2 — Get the Neon **development** branch connection string

In the Neon console:

1. Project → **Branches** → click **development**
2. In the **Computes** section, click **Connect** on the Primary row
3. In the dialog:
   - **Database:** your db (likely `neondb`)
   - **Role:** `neondb_owner`
   - **Pooled connection:** **OFF** (you need the direct URL for restore)
4. Copy the URL — should look like:
   ```
   postgresql://neondb_owner:PASSWORD@ep-wild-unit-adza266s.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require
   ```
   (Note: **no `-pooler`** in the host)

Save it to a fish variable so you don't paste it three times:

```bash
set NEON_DEV "postgresql://neondb_owner:PASSWORD@ep-wild-unit-adza266s.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require"
```

---

## Step 3 — Test the connection (also wakes the suspended compute)

```bash
psql "$NEON_DEV" -c "SELECT version();"
```

- Returns a Postgres version → ready to go.
- Auth error → wrong password or URL malformed. Re-copy from Neon.

---

## Step 4 — Wipe the dev branch's schema

```bash
psql "$NEON_DEV" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
```

This deletes everything in the dev branch. **Make sure you're on `development`, not `production`** — double-check the host in `$NEON_DEV` (production has a different `ep-...` ID).

---

## Step 5 — Restore the dump

```bash
psql "$NEON_DEV" -f /home/rocky/code/iqra-wave/quran-wave/db-backup/pre-deploy-YYYYMMDD-HHMMSS.sql
```

(Replace the filename with whichever you downloaded.)

Expect 1–3 minutes for ~60 MB. Hundreds of lines will fly past.

| Output | Meaning |
|---|---|
| `CREATE TABLE`, `CREATE INDEX`, `COPY <N>` | Working |
| `ERROR: role "postgres" does not exist` | Ignore — harmless |
| `ERROR: extension "..." not available` | Only matters if your app uses it |
| `ERROR` on `COPY` or `CREATE TABLE` | Real problem — investigate |

---

## Step 6 — Verify

```bash
psql "$NEON_DEV" -c "\dt"
```

You should see all 28 tables: `admins`, `users`, `corpus_quran_words`, `quranic_ayahs`, etc. Press `q` to exit the pager.

Spot-check a row count:

```bash
psql "$NEON_DEV" -c "SELECT count(*) FROM users;"
```

---

## Step 7 — Update local `.env`

Switch to the **pooled** dev URL for your app (the one **with** `-pooler` in the host):

```env
DATABASE_URL="postgresql://neondb_owner:PASSWORD@ep-wild-unit-adza266s-pooler.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require&schema=public"
```

If your `schema.prisma` uses `directUrl`, also set:

```env
DIRECT_URL="postgresql://neondb_owner:PASSWORD@ep-wild-unit-adza266s.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require&channel_binding=require&schema=public"
```

**Always wrap URLs in double quotes.** The `&` and `?` will break unquoted values.

---

## Step 8 — Run the app

```bash
npm run dev
```

- Prisma error `"(not available)" credentials are not valid` → URL malformed. Re-copy, check quotes, check the password for special chars.
- **Don't run `prisma migrate` after a fresh restore** — you'd overwrite the data.

---

## Common pitfalls

1. **Don't use the `-pooler` URL for restore.** Pooler chokes on long-running ops. Use direct (no `-pooler`).
2. **Always quote URLs** in `.env` and on the command line — `&` and `?` are shell/URL metacharacters.
3. **Don't paste passwords in chat, screenshots, or git.** If you do, rotate immediately in Neon → Roles & Databases.
4. **The `role "postgres" does not exist` errors are expected** — the AWS dump references `postgres` ownership; Neon uses `neondb_owner`. Tables are created fine, just with a different owner.
5. **Restore overwrites everything.** Always restore into `development`, never `production`, unless that's literally what you intend.

---

## Quick reference (copy-paste block)

```bash
# 1. Download latest backup
set LATEST (ssh -i ~/paperspace_key iqra_rocky@44.196.233.7 'ls -t ~/QuranWave.Api/backups/*.sql | head -1')
scp -i ~/paperspace_key "iqra_rocky@44.196.233.7:$LATEST" /home/rocky/code/iqra-wave/quran-wave/db-backup/

# 2. Set Neon dev URL (DIRECT, no -pooler)
set NEON_DEV "postgresql://neondb_owner:PASSWORD@ep-xxxxxxxx.c-2.us-east-1.aws.neon.tech/neondb?sslmode=require"

# 3. Test
psql "$NEON_DEV" -c "SELECT version();"

# 4. Wipe + restore
psql "$NEON_DEV" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
psql "$NEON_DEV" -f /home/rocky/code/iqra-wave/quran-wave/db-backup/pre-deploy-YYYYMMDD-HHMMSS.sql

# 5. Verify
psql "$NEON_DEV" -c "\dt"
```
