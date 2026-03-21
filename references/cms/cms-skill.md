---
name: nodejs-cms-builder
description: >-
  Build CMS systems on Node.js following the SmartEat architecture: monolithic
  server.js with all routes inline, EJS views with express-ejs-layouts, Firebase
  or Supabase as the backend (Firestore/PostgreSQL accessed directly, no ORM),
  session-based auth with bcrypt, Bulma CSS + Font Awesome, dark mode sidebar
  layout, file uploads via busboy to cloud storage.
  Use when the user asks to build a CMS, content management system, admin panel,
  dashboard, blog platform, or any Node.js project that manages structured content
  with Firebase or Supabase integration.
---

# Node.js CMS Builder

Build CMS systems following the SmartEat monolithic architecture pattern.

## Architecture Overview

```
project-root/
├── server.js                    # Single entry — ALL routes, middleware, logic inline
├── package.json
├── .env / .encrypted.env        # Secrets (dotenv)
├── views/                       # EJS templates
│   ├── layout.ejs               # Shared layout (sidebar + header + <%- body %>)
│   ├── login.ejs                # Auth (layout: false)
│   ├── dashboard.ejs            # Admin dashboard
│   ├── [entity].ejs             # List page per entity
│   ├── add[Entity].ejs          # Create form
│   ├── edit[Entity].ejs         # Edit form
│   └── 404.ejs                  # Error page
├── public/                      # Static assets
│   ├── style.css                # Global styles
│   ├── script.js                # Global JS
│   └── images/                  # Logos, icons, animations
├── uploads/                     # Temp multer/busboy dir
└── [service-account].json       # Firebase credentials (gitignored)
```

**No `routes/`, no `models/`, no `controllers/` directories.** Everything lives in `server.js`. Database is accessed directly via Firestore SDK or Supabase client.

## Backend Selection

**Firebase** — realtime sync, document-oriented data, serverless Cloud Functions, Google Cloud ecosystem. See [firebase.md](firebase.md).

**Supabase** — relational data (PostgreSQL), complex queries/joins, row-level security, SQL migrations, self-hostable. See [supabase.md](supabase.md).

## server.js Structure

The single `server.js` follows this top-to-bottom order:

```javascript
// 1. Require dependencies
const express = require('express');
const admin = require('firebase-admin');      // OR: const { createClient } = require('@supabase/supabase-js');
const session = require('express-session');
const bcrypt = require('bcrypt');
const expressLayouts = require('express-ejs-layouts');
const path = require('path');
const helmet = require('helmet');
const multer = require('multer');
const morgan = require('morgan');
const busboy = require('busboy');
const os = require('os');
const fs = require('fs');
require('dotenv').config();

const app = express();

// 2. Body parsing + static + logging
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static('public'));
app.use(morgan('dev'));

// 3. Busboy file upload middleware (multipart/form-data)
const filesUpload = (req, res, next) => { /* see reference */ };
app.use(filesUpload);

// 4. Initialize backend (Firebase Admin or Supabase)
admin.initializeApp({
  credential: admin.credential.cert(require('./service-account.json')),
  storageBucket: '<project-id>.appspot.com'
});
const db = admin.firestore();
const bucket = admin.storage().bucket();

// 5. Session middleware (Firestore-backed or connect-pg-simple for Supabase)
app.use(session({ /* config */ }));

// 6. View locals middleware
app.use((req, res, next) => {
  res.locals.loggedIn = !!req.session.user;
  res.locals.userName = req.session.user ? req.session.user.email : null;
  res.locals.admin = req.session?.user || null;
  next();
});

// 7. Auth middleware functions
function isAuthenticated(req, res, next) { /* check session */ }
function redirectIfAuthenticated(req, res, next) { /* redirect if logged in */ }

// 8. EJS + layouts
app.set('view engine', 'ejs');
app.use(expressLayouts);
app.set('views', './views');

// 9. Helper functions (uploadFileToFirebase, etc.)

// 10. ALL route handlers (grouped by entity, inline)
//     - Auth routes (/login, /logout)
//     - Dashboard (/dashboard)
//     - Entity CRUD (/items, /items/add, /items/edit/:id, /items/delete/:id)
//     - Media routes
//     - Settings routes
//     - Public pages

// 11. 404 handler
app.use((req, res) => res.status(404).render('404', { layout: false }));

// 12. Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

module.exports = { app };
```

## Core Dependencies

```json
{
  "dependencies": {
    "express": "^4.21",
    "ejs": "^3.1",
    "express-ejs-layouts": "^2.5",
    "express-session": "^1.18",
    "bcrypt": "^5.1",
    "dotenv": "^16.4",
    "helmet": "^8.0",
    "morgan": "^1.10",
    "multer": "^1.4",
    "busboy": "^1.6",
    "cors": "^2.8",
    "sharp": "^0.33",
    "slugify": "^1.6",
    "express-rate-limit": "^8.3"
  }
}
```

Add `firebase-admin`, `@google-cloud/storage`, `@google-cloud/connect-firestore` for Firebase.
Add `@supabase/supabase-js`, `connect-pg-simple`, `pg` for Supabase.

## EJS Views

### layout.ejs Pattern

`layout.ejs` wraps all authenticated pages with:
- Collapsible dark sidebar (background `#1f2937`)
- Top header with avatar dropdown
- `<%- body %>` content area
- Delete/confirm modals
- Theme toggle (dark/light stored in localStorage)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><%= typeof title !== 'undefined' ? title : 'CMS' %></title>
    <link rel="stylesheet" href="/style.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.9.3/css/bulma.min.css">
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">
</head>
<body>
    <!-- Sidebar -->
    <aside class="custom-sidebar" id="sidebar">
        <div class="sidebar-brand">
            <div class="sidebar-logo-wrapper">
                <img src="/images/logo.png" alt="Logo">
            </div>
            <span class="sidebar-title">CMS</span>
        </div>
        <nav class="menu">
            <ul class="menu-list">
                <li><a href="/dashboard"><i class="fas fa-tachometer-alt"></i><span>Dashboard</span></a></li>
                <!-- Add menu items per entity -->
            </ul>
        </nav>
        <button class="toggle-sidebar-btn" onclick="toggleSidebar()">
            <i class="fas fa-bars"></i>
        </button>
    </aside>

    <!-- Header -->
    <header class="header">
        <div class="header-right">
            <span><%= admin ? admin.name : '' %></span>
        </div>
    </header>

    <!-- Main content -->
    <main class="main-content">
        <%- body %>
    </main>

    <script src="/script.js"></script>
</body>
</html>
```

Pages that don't use the layout (login, public pages): `res.render('login', { layout: false })`.

### View Naming Convention

| Purpose | Filename |
|---------|----------|
| List entities | `articles.ejs`, `users.ejs`, `products.ejs` |
| Add entity | `addArticle.ejs`, `addUser.ejs`, `addProduct.ejs` |
| Edit entity | `editArticle.ejs`, `editUser.ejs`, `editProduct.ejs` |
| Public page | `home.ejs`, `contact.ejs`, `terms.ejs` |
| Localized | `home_en.ejs`, `home_ru.ejs`, `terms_en.ejs` |

### View Structure

Each list page includes: search bar, filters, data table with actions, pagination. Each form page includes: input fields matching the entity schema, file upload where needed, save/cancel buttons.

```html
<!-- Example: articles.ejs (list) -->
<div class="container">
    <div class="level">
        <h1 class="title">Articles</h1>
        <a href="/articles/add" class="button is-primary">
            <i class="fas fa-plus"></i> Add Article
        </a>
    </div>
    <table class="table is-fullwidth is-hoverable">
        <thead>
            <tr>
                <th>Title</th>
                <th>Status</th>
                <th>Created</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            <% articles.forEach(article => { %>
            <tr>
                <td><%= article.title %></td>
                <td><span class="tag is-<%= article.status === 'published' ? 'success' : 'warning' %>"><%= article.status %></span></td>
                <td><%= article.createdAt %></td>
                <td>
                    <a href="/articles/edit/<%= article.id %>" class="button is-small is-info"><i class="fas fa-edit"></i></a>
                    <button class="button is-small is-danger" onclick="confirmDelete('<%= article.id %>')"><i class="fas fa-trash"></i></button>
                </td>
            </tr>
            <% }) %>
        </tbody>
    </table>
</div>
```

## Route Patterns (Inline in server.js)

### Auth Routes

```javascript
app.get('/login', redirectIfAuthenticated, (req, res) => {
    res.render('login', { layout: false, error: null });
});

app.post('/login', async (req, res) => {
    const { email, password } = req.body;
    const snapshot = await db.collection('admins').where('email', '==', email).limit(1).get();
    if (snapshot.empty) return res.render('login', { layout: false, error: 'Invalid credentials' });

    const adminDoc = snapshot.docs[0];
    const adminData = adminDoc.data();
    const match = await bcrypt.compare(password, adminData.password);
    if (!match) return res.render('login', { layout: false, error: 'Invalid credentials' });

    req.session.user = { id: adminDoc.id, name: adminData.name, email: adminData.email, role: adminData.role };
    res.redirect('/dashboard');
});

app.get('/logout', (req, res) => {
    req.session.destroy(() => res.redirect('/login'));
});
```

### CRUD Routes Pattern (per entity)

```javascript
// List
app.get('/articles', isAuthenticated, async (req, res) => {
    const snapshot = await db.collection('articles').orderBy('createdAt', 'desc').get();
    const articles = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        createdAt: doc.data().createdAt?.toDate().toLocaleDateString() || 'N/A'
    }));
    res.render('articles', { articles });
});

// Add form
app.get('/articles/add', isAuthenticated, (req, res) => {
    res.render('addArticle');
});

// Create
app.post('/articles/add', isAuthenticated, async (req, res) => {
    const { title, content, status } = req.body;
    let imageUrl = '';
    if (req.files && req.files.length > 0) {
        imageUrl = await uploadFileToFirebase(req.files[0].buffer, req.files[0].originalname, req.files[0].mimetype);
    }
    await db.collection('articles').add({
        title, content, status: status || 'draft', imageUrl,
        authorId: req.session.user.id,
        createdAt: admin.firestore.FieldValue.serverTimestamp(),
        updatedAt: admin.firestore.FieldValue.serverTimestamp()
    });
    res.redirect('/articles');
});

// Edit form
app.get('/articles/edit/:id', isAuthenticated, async (req, res) => {
    const doc = await db.collection('articles').doc(req.params.id).get();
    if (!doc.exists) return res.redirect('/articles');
    res.render('editArticle', { article: { id: doc.id, ...doc.data() } });
});

// Update
app.post('/articles/edit/:id', isAuthenticated, async (req, res) => {
    const { title, content, status } = req.body;
    const updates = { title, content, status, updatedAt: admin.firestore.FieldValue.serverTimestamp() };
    if (req.files && req.files.length > 0) {
        updates.imageUrl = await uploadFileToFirebase(req.files[0].buffer, req.files[0].originalname, req.files[0].mimetype);
    }
    await db.collection('articles').doc(req.params.id).update(updates);
    res.redirect('/articles');
});

// Delete
app.post('/articles/delete/:id', isAuthenticated, async (req, res) => {
    await db.collection('articles').doc(req.params.id).delete();
    res.redirect('/articles');
});
```

Repeat this pattern for every entity (users, products, categories, media, settings, etc.).

## Authentication & Session

- **Session store**: Firestore via `@google-cloud/connect-firestore` (or `connect-pg-simple` for Supabase)
- **Passwords**: `bcrypt.hash(password, 10)` / `bcrypt.compare(password, hash)`
- **Cookie**: 7-day maxAge, httpOnly, sameSite `lax`, secure in production
- **Roles**: stored in admin document (`admin`, `editor`, `author`, `viewer`)
- **Login lockout**: track failed attempts in the admin doc, lock after N failures

## File Uploads

Use busboy middleware for multipart parsing (supports Firebase Cloud Functions where multer doesn't work):

```javascript
async function uploadFileToFirebase(fileBuffer, fileName, mimeType, folder = 'media') {
    const file = bucket.file(`${folder}/${fileName}`);
    await file.save(fileBuffer, {
        metadata: { contentType: mimeType },
        public: true
    });
    const [url] = await file.getSignedUrl({ action: 'read', expires: '03-01-2500' });
    return url;
}
```

## Dark Mode & Theming

Use CSS custom properties with `[data-theme="dark"]` selector:

```css
:root {
    --bg-color: #ffffff;
    --text-color: #333;
    --sidebar-bg: #1f2937;
    --card-bg: #ffffff;
    --card-border: #ececec;
}
[data-theme="dark"] {
    --bg-color: #13151a;
    --text-color: #ffffff;
    --card-bg: #1b1e24;
    --card-border: #2a2d35;
}
```

Toggle in `script.js`, persist in `localStorage`.

## Firebase Cloud Functions Deployment

For production, wrap the Express app as a Cloud Function:

```javascript
// functions/index.js
const { onRequest } = require('firebase-functions/v2/https');
const { app } = require('./server');
exports.app = onRequest({ timeoutSeconds: 300, memory: '1GiB' }, app);
```

With `firebase.json` rewriting all traffic to the function:

```json
{
  "hosting": {
    "public": "public",
    "rewrites": [{ "source": "**", "destination": "/app" }]
  }
}
```

## Content Modeling

For detailed content modeling patterns (blog, pages, products, localization, versioning), see [content-modeling.md](content-modeling.md).
