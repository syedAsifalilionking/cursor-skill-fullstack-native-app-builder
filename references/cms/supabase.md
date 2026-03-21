# Supabase Integration Reference

Patterns for Supabase backend following the monolithic server.js architecture.

## Setup in server.js

```javascript
const { createClient } = require('@supabase/supabase-js');

const supabase = createClient(
    process.env.SUPABASE_URL,
    process.env.SUPABASE_SERVICE_KEY
);
```

## Dependencies

```json
{
  "@supabase/supabase-js": "^2.45",
  "connect-pg-simple": "^10.0",
  "pg": "^8.13"
}
```

## Session Store (PostgreSQL)

```javascript
const pgSession = require('connect-pg-simple')(session);

app.use(session({
    store: new pgSession({
        conString: process.env.DATABASE_URL,
        tableName: 'sessions'
    }),
    secret: process.env.SESSION_SECRET || 'cms-secret-key',
    name: 'cms.sid',
    resave: false,
    saveUninitialized: false,
    rolling: true,
    cookie: {
        maxAge: 7 * 24 * 60 * 60 * 1000,
        httpOnly: true,
        sameSite: 'lax',
        secure: process.env.NODE_ENV === 'production'
    }
}));
```

Create the sessions table:

```sql
CREATE TABLE "sessions" (
  "sid" varchar NOT NULL COLLATE "default",
  "sess" json NOT NULL,
  "expire" timestamp(6) NOT NULL,
  CONSTRAINT "sessions_pkey" PRIMARY KEY ("sid")
);
CREATE INDEX "IDX_sessions_expire" ON "sessions" ("expire");
```

## Database Schema

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE admins (
    id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    name text NOT NULL,
    email text UNIQUE NOT NULL,
    password text NOT NULL,
    role text NOT NULL DEFAULT 'editor' CHECK (role IN ('admin', 'editor', 'author', 'viewer')),
    failed_attempts integer DEFAULT 0,
    last_failed_at timestamptz,
    last_login timestamptz,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

CREATE TABLE users (
    id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    name text NOT NULL,
    email text UNIQUE NOT NULL,
    status text DEFAULT 'active',
    last_login timestamptz,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

-- Repeat for each entity
CREATE TABLE articles (
    id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    title text NOT NULL,
    content text,
    slug text UNIQUE,
    status text DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived', 'deleted')),
    image_url text,
    author_id uuid REFERENCES admins(id),
    published_at timestamptz,
    deleted_at timestamptz,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

CREATE TABLE media (
    id uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    filename text NOT NULL,
    path text NOT NULL,
    url text NOT NULL,
    mimetype text NOT NULL,
    size integer NOT NULL,
    uploaded_by uuid REFERENCES admins(id),
    created_at timestamptz DEFAULT now()
);

CREATE TABLE settings (
    key text PRIMARY KEY,
    value jsonb NOT NULL DEFAULT '{}',
    updated_at timestamptz DEFAULT now()
);

CREATE INDEX idx_articles_status ON articles(status, created_at DESC);
CREATE INDEX idx_articles_author ON articles(author_id);
CREATE INDEX idx_articles_slug ON articles(slug);

CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_admins_updated BEFORE UPDATE ON admins
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_users_updated BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_articles_updated BEFORE UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

## CRUD Operations (Direct Supabase Client)

### List with Pagination

```javascript
app.get('/articles', isAuthenticated, async (req, res) => {
    const page = parseInt(req.query.page) || 1;
    const perPage = 25;

    let query = supabase
        .from('articles')
        .select('*', { count: 'exact' })
        .is('deleted_at', null)
        .order('created_at', { ascending: false })
        .range((page - 1) * perPage, page * perPage - 1);

    if (req.query.status) {
        query = query.eq('status', req.query.status);
    }

    const { data: articles, count, error } = await query;
    if (error) console.error('Error fetching articles:', error);

    const formatted = (articles || []).map(a => ({
        ...a,
        createdAt: a.created_at ? new Date(a.created_at).toLocaleDateString() : 'N/A'
    }));

    res.render('articles', {
        articles: formatted,
        currentPage: page,
        totalPages: Math.ceil((count || 0) / perPage),
        total: count || 0
    });
});
```

### Create with File Upload

```javascript
app.post('/articles/add', isAuthenticated, async (req, res) => {
    try {
        const { title, content, status } = req.body;
        let imageUrl = '';

        if (req.files && req.files.length > 0) {
            imageUrl = await uploadToSupabase(
                req.files[0].buffer, req.files[0].originalname, req.files[0].mimetype
            );
        }

        const { error } = await supabase.from('articles').insert({
            title,
            content,
            status: status || 'draft',
            image_url: imageUrl,
            author_id: req.session.user.id
        });

        if (error) throw error;
        res.redirect('/articles');
    } catch (err) {
        console.error('Error creating article:', err);
        res.redirect('/articles/add');
    }
});
```

### Update

```javascript
app.post('/articles/edit/:id', isAuthenticated, async (req, res) => {
    try {
        const { title, content, status } = req.body;
        const updates = { title, content, status };

        if (req.files && req.files.length > 0) {
            updates.image_url = await uploadToSupabase(
                req.files[0].buffer, req.files[0].originalname, req.files[0].mimetype
            );
        }

        const { error } = await supabase
            .from('articles')
            .update(updates)
            .eq('id', req.params.id);

        if (error) throw error;
        res.redirect('/articles');
    } catch (err) {
        console.error('Error updating article:', err);
        res.redirect(`/articles/edit/${req.params.id}`);
    }
});
```

### Soft Delete

```javascript
app.post('/articles/delete/:id', isAuthenticated, async (req, res) => {
    await supabase
        .from('articles')
        .update({ status: 'deleted', deleted_at: new Date().toISOString() })
        .eq('id', req.params.id);
    res.redirect('/articles');
});
```

## File Upload to Supabase Storage

```javascript
const { v4: uuid } = require('uuid');

async function uploadToSupabase(buffer, originalName, mimetype, folder = 'media') {
    const fileName = `${folder}/${uuid()}-${originalName}`;

    const { error } = await supabase.storage
        .from('media')
        .upload(fileName, buffer, { contentType: mimetype, upsert: false });

    if (error) throw error;

    const { data: { publicUrl } } = supabase.storage
        .from('media')
        .getPublicUrl(fileName);

    return publicUrl;
}
```

Create the storage bucket:

```sql
INSERT INTO storage.buckets (id, name, public) VALUES ('media', 'media', true);
```

## Authentication

```javascript
app.post('/login', async (req, res) => {
    const { email, password } = req.body;

    const { data: admins, error } = await supabase
        .from('admins')
        .select('*')
        .eq('email', email)
        .limit(1);

    if (error || !admins || admins.length === 0) {
        return res.render('login', { layout: false, error: 'Invalid credentials' });
    }

    const adminData = admins[0];

    // Check lockout
    const { data: settings } = await supabase
        .from('settings')
        .select('value')
        .eq('key', 'security')
        .single();

    const maxAttempts = settings?.value?.maxLoginAttempts || 5;
    const lockoutMinutes = settings?.value?.lockoutDuration || 15;

    if (adminData.failed_attempts >= maxAttempts) {
        const lockedUntil = new Date(adminData.last_failed_at);
        if (Date.now() - lockedUntil.getTime() < lockoutMinutes * 60 * 1000) {
            return res.render('login', { layout: false, error: 'Account locked. Try again later.' });
        }
        await supabase.from('admins').update({ failed_attempts: 0 }).eq('id', adminData.id);
    }

    const match = await bcrypt.compare(password, adminData.password);
    if (!match) {
        await supabase.from('admins').update({
            failed_attempts: (adminData.failed_attempts || 0) + 1,
            last_failed_at: new Date().toISOString()
        }).eq('id', adminData.id);
        return res.render('login', { layout: false, error: 'Invalid credentials' });
    }

    await supabase.from('admins').update({
        failed_attempts: 0,
        last_login: new Date().toISOString()
    }).eq('id', adminData.id);

    req.session.user = {
        id: adminData.id,
        name: adminData.name,
        email: adminData.email,
        role: adminData.role
    };
    res.redirect('/dashboard');
});
```

## Row-Level Security (RLS)

```sql
ALTER TABLE articles ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE media ENABLE ROW LEVEL SECURITY;

-- Service key bypasses RLS, so CMS server has full access.
-- These policies apply when using anon/user keys (e.g. public API).

CREATE POLICY "public_read_published" ON articles
    FOR SELECT USING (status = 'published' AND deleted_at IS NULL);

CREATE POLICY "authenticated_full_access" ON articles
    FOR ALL USING (auth.role() = 'service_role');
```

## Maintenance Mode

```javascript
let maintenanceMode = false;
async function refreshMaintenance() {
    const { data } = await supabase
        .from('settings')
        .select('value')
        .eq('key', 'general')
        .single();
    maintenanceMode = data?.value?.maintenance_mode === true;
}
refreshMaintenance();
setInterval(refreshMaintenance, 30000);
```

## Dashboard Aggregation Queries

```javascript
app.get('/dashboard', isAuthenticated, async (req, res) => {
    const { count: totalUsers } = await supabase
        .from('users').select('*', { count: 'exact', head: true });

    const oneDayAgo = new Date(Date.now() - 86400000).toISOString();
    const { count: activeUsers } = await supabase
        .from('users').select('*', { count: 'exact', head: true })
        .gte('last_login', oneDayAgo);

    const { count: totalArticles } = await supabase
        .from('articles').select('*', { count: 'exact', head: true })
        .is('deleted_at', null);

    res.render('dashboard', {
        totalUsers: totalUsers || 0,
        activeUsers: activeUsers || 0,
        totalArticles: totalArticles || 0
    });
});
```

## Environment Variables

```
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_KEY=eyJhbGc...
DATABASE_URL=postgresql://postgres:password@db.xxxx.supabase.co:5432/postgres
SESSION_SECRET=your-secret-here
NODE_ENV=development
PORT=3000
```
