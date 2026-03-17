# LinkShort — Railway Backend Deployment

## What is in this folder

| File | Purpose |
|------|---------|
| `index.cjs` | Complete bundled server (no install needed) |
| `package.json` | Tells Railway to run `node index.cjs` |
| `railway.toml` | Railway auto-reads this for deploy config |

---

## Environment Variables (set these in Railway)

| Variable | Value | Required |
|----------|-------|----------|
| `SESSION_SECRET` | Any random text, e.g. `abc123xyz-secret` | YES |
| `NODE_ENV` | `production` | YES |
| `DATABASE_URL` | Auto-provided by Railway PostgreSQL add-on | AUTO |

---

## Step-by-Step Deployment

### 1. Create a GitHub Repository

- Go to github.com → New repository → name it `linkshort-backend`
- Upload these files into it (all are required):
  - `index.cjs`
  - `package.json` ← must include the `engines` field or Railway won't detect Node.js
  - `railway.toml` ← tells Railway how to build and start
  - `README.md` (optional)

### 2. Deploy to Railway

1. Go to [railway.app](https://railway.app) and sign in
2. Click **New Project** → **Deploy from GitHub repo**
3. Select your `linkshort-backend` repo
4. Railway will detect it automatically

### 3. Add PostgreSQL Database

- In your Railway project, click **+ New** → **Database** → **Add PostgreSQL**
- Railway will automatically set `DATABASE_URL` for your server

### 4. Set Environment Variables

- Go to your server service → **Variables** tab → click **New Variable**
- Add: `SESSION_SECRET` = `any-random-secret-text-here`
- Add: `NODE_ENV` = `production`

### 5. Run the Database SQL

- In Railway, click your **PostgreSQL** service → go to the **Data** tab
- Open the **Query** panel and paste this entire SQL block, then run it:

```sql
-- Enum types
CREATE TYPE user_role AS ENUM ('admin', 'user');
CREATE TYPE user_status AS ENUM ('pending', 'approved', 'banned');
CREATE TYPE ad_slot AS ENUM ('left1', 'left2', 'right1', 'right2');
CREATE TYPE ad_type AS ENUM ('html', 'image', 'script');

-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  full_name TEXT NOT NULL,
  telegram_username TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  role user_role NOT NULL DEFAULT 'user',
  status user_status NOT NULL DEFAULT 'pending',
  restricted BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Short links table
CREATE TABLE short_links (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  short_code TEXT NOT NULL UNIQUE,
  links TEXT NOT NULL,
  timer_seconds INTEGER NOT NULL DEFAULT 10,
  reload_interval_minutes REAL NOT NULL DEFAULT 1,
  total_clicks INTEGER NOT NULL DEFAULT 0,
  total_views INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Ads table
CREATE TABLE ads (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  slot ad_slot NOT NULL,
  content TEXT NOT NULL,
  ad_type ad_type NOT NULL DEFAULT 'html',
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Seed admin account (password: admin123)
INSERT INTO users (full_name, telegram_username, password_hash, role, status, restricted)
VALUES (
  'Administrator',
  'admin',
  '$2b$10$rQFIr55kDuW7dY1ujyDn3OehmTBB4j6591KdXYW38G12cQVtVKhlW',
  'admin',
  'approved',
  false
)
ON CONFLICT (telegram_username) DO NOTHING;
```

### 6. Get Your Railway URL

- Go to your server service → **Settings** → **Networking** → **Generate Domain**
- Copy the URL (e.g. `https://linkshort-backend-production.railway.app`)
- You will need this for the Vercel frontend setup

---

## Admin Login

- **Username**: `admin`
- **Password**: `admin123`
- Change the password immediately after login via Admin Panel → Settings

---

## Health Check

After deployment, open this URL in your browser to verify the server is running:
```
https://your-railway-url.railway.app/api/health
```
It should return: `{"status":"ok"}`
