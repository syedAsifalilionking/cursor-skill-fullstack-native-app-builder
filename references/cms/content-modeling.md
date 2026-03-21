# Content Modeling Reference

Patterns for structuring CMS content in the monolithic server.js architecture. Data is stored directly in Firestore collections or Supabase tables — no ORM, no models directory.

## Common Entities

### Blog / Articles

```
Collection: articles (Firestore) / Table: articles (Supabase)

Fields:
  title        — text, max 200 chars
  content      — richtext (HTML or Markdown)
  excerpt      — text, max 500 chars
  slug         — auto-generated from title, unique
  coverImage   — URL string (uploaded to Storage)
  category     — string or reference
  tags         — array of strings
  status       — draft | published | archived | deleted
  authorId     — reference to admins
  seoTitle     — text, max 60 chars
  seoDescription — text, max 160 chars
  publishedAt  — timestamp (set on publish)
  deletedAt    — timestamp (set on soft delete)
  createdAt    — server timestamp
  updatedAt    — server timestamp
```

### Users (Managed via CMS)

```
Collection: users / Table: users

Fields:
  name         — text
  email        — text, unique
  phone        — text
  avatar       — URL string
  status       — active | inactive | banned
  role         — user role in the app
  lastLogin    — timestamp
  createdAt    — server timestamp
  updatedAt    — server timestamp
```

### Products (E-commerce CMS)

```
Collection: products / Table: products

Fields:
  name         — text
  description  — richtext
  price        — number
  sku          — text, unique
  images       — array of URL strings
  category     — string or reference
  inStock      — boolean
  status       — draft | published | archived
  createdAt    — server timestamp
  updatedAt    — server timestamp
```

### Pages (Static content)

```
Collection: pages / Table: pages

Fields:
  title        — text
  slug         — unique
  body         — richtext (HTML)
  template     — default | landing | sidebar
  parentId     — reference to parent page (for hierarchy)
  order        — number (sort order)
  status       — draft | published
  createdAt    — server timestamp
  updatedAt    — server timestamp
```

### Notifications (Push)

```
Collection: notifications / Table: notifications

Fields:
  title        — text
  body         — text
  imageUrl     — URL string
  targetAudience — all | segment | individual
  targetUsers  — array of user IDs (if individual)
  sentAt       — timestamp
  status       — draft | sent | scheduled
  scheduledFor — timestamp
  createdAt    — server timestamp
```

### Settings

```
Collection: settings / Table: settings

Documents/Rows:
  general      — { siteName, logo, maintenanceMode, ... }
  security     — { maxLoginAttempts, lockoutDuration, passwordMinLength, ... }
  apiKeys      — { openaiKey, analyticsKey, ... }
  media        — { maxUploadSize, allowedTypes, cdnUrl, ... }
```

## Content Lifecycle

```
draft → published → archived
  ↑         |           |
  └─────────┴───────────┘
        (can revert)
```

### Status Transitions

| From | Allowed To |
|------|-----------|
| draft | published, deleted |
| published | draft, archived, deleted |
| archived | draft, deleted |
| deleted | permanent delete (admin only) |

### In Route Handlers

```javascript
// Publish
app.post('/articles/publish/:id', isAuthenticated, async (req, res) => {
    await db.collection('articles').doc(req.params.id).update({
        status: 'published',
        publishedAt: admin.firestore.FieldValue.serverTimestamp()
    });
    res.redirect('/articles');
});

// Unpublish
app.post('/articles/unpublish/:id', isAuthenticated, async (req, res) => {
    await db.collection('articles').doc(req.params.id).update({
        status: 'draft',
        publishedAt: null
    });
    res.redirect('/articles');
});
```

## Slug Generation

Generate in the route handler, not in a model:

```javascript
const slugify = require('slugify');

function generateSlug(title) {
    return slugify(title, { lower: true, strict: true });
}

// In route: check uniqueness
async function uniqueSlug(collection, title) {
    let slug = generateSlug(title);
    let counter = 1;
    const base = slug;
    while (true) {
        const existing = await db.collection(collection)
            .where('slug', '==', slug).limit(1).get();
        if (existing.empty) break;
        slug = `${base}-${counter++}`;
    }
    return slug;
}
```

## Localization Pattern

For multi-language content, use separate view files per language:

```
views/
  home.ejs          # Hebrew (default)
  home_en.ejs       # English
  home_ru.ejs       # Russian
  terms.ejs
  terms_en.ejs
  terms_ru.ejs
```

With routes:

```javascript
app.get('/',    (req, res) => res.render('home',    { layout: false }));
app.get('/en',  (req, res) => res.render('home_en', { layout: false }));
app.get('/ru',  (req, res) => res.render('home_ru', { layout: false }));
```

For CMS-managed translations, store localized fields as nested objects:

```javascript
// Firestore document
{
  title: { he: "כותרת", en: "Title", ru: "Заголовок" },
  content: { he: "...", en: "...", ru: "..." },
  status: "published",
  createdAt: Timestamp
}
```

## CSV Export Pattern

```javascript
const { Parser } = require('json2csv');

app.get('/export-users', isAuthenticated, async (req, res) => {
    const snapshot = await db.collection('users').get();
    const users = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

    const parser = new Parser({ fields: ['id', 'name', 'email', 'status'] });
    const csv = parser.parse(users);

    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', 'attachment; filename=users.csv');
    res.send(csv);
});
```

## Image Processing

```javascript
const sharp = require('sharp');

async function processImage(buffer) {
    return sharp(buffer)
        .resize(1920, 1080, { fit: 'inside', withoutEnlargement: true })
        .webp({ quality: 80 })
        .toBuffer();
}

// In route handler before upload:
if (file.mimetype.startsWith('image/')) {
    const processed = await processImage(file.buffer);
    imageUrl = await uploadFileToFirebase(processed, file.originalname.replace(/\.[^.]+$/, '.webp'), 'image/webp');
}
```

## Email Notifications

```javascript
const nodemailer = require('nodemailer');

const transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: { user: process.env.EMAIL_USER, pass: process.env.EMAIL_PASS }
});

async function sendNotification(to, subject, html) {
    await transporter.sendMail({
        from: `"CMS" <${process.env.EMAIL_USER}>`,
        to, subject, html
    });
}
```
