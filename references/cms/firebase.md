# Firebase Integration Reference

Patterns for Firebase backend following the monolithic server.js architecture.

## Setup in server.js

```javascript
const admin = require('firebase-admin');
const { Storage } = require('@google-cloud/storage');
const { FirestoreStore } = require('@google-cloud/connect-firestore');

admin.initializeApp({
    credential: admin.credential.cert(require('./service-account.json')),
    storageBucket: '<project-id>.appspot.com'
});

const db = admin.firestore();

const storage = new Storage({
    projectId: '<project-id>',
    keyFilename: './service-account.json'
});
const bucket = storage.bucket('<project-id>.appspot.com');
```

## Dependencies

```json
{
  "firebase-admin": "^13.3",
  "@google-cloud/storage": "^7.14",
  "@google-cloud/connect-firestore": "^3.0",
  "@google-cloud/firestore": "^7.11"
}
```

## Session Store

```javascript
app.use(session({
    store: new FirestoreStore({
        dataset: admin.firestore(),
        kind: 'express-sessions'
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

## Firestore Collections Structure

```
admins/                     # CMS admin accounts
  {adminId}/
    name, email, password (bcrypt hash), role, createdAt

users/                      # App users managed through CMS
  {userId}/
    name, email, status, createdAt, lastLogin

[entity]/                   # One collection per content type
  {docId}/
    ...fields, status, authorId, createdAt, updatedAt

settings/                   # App settings
  general/                  # Site name, maintenance mode, etc.
  security/                 # Max login attempts, lockout duration
  apiKeys/                  # External API keys

media/                      # Media metadata
  {mediaId}/
    url, path, mimetype, size, uploadedBy, createdAt

express-sessions/           # Session store (auto-managed)
```

## CRUD Operations (Direct Firestore)

### List with Pagination

```javascript
app.get('/items', isAuthenticated, async (req, res) => {
    const page = parseInt(req.query.page) || 1;
    const perPage = 25;

    let query = db.collection('items').orderBy('createdAt', 'desc');

    if (req.query.status) {
        query = query.where('status', '==', req.query.status);
    }

    const countSnapshot = await query.count().get();
    const total = countSnapshot.data().count;

    const snapshot = await query
        .offset((page - 1) * perPage)
        .limit(perPage)
        .get();

    const items = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        createdAt: doc.data().createdAt?.toDate().toLocaleDateString() || 'N/A'
    }));

    res.render('items', {
        items,
        currentPage: page,
        totalPages: Math.ceil(total / perPage),
        total
    });
});
```

### Create with File Upload

```javascript
app.post('/items/add', isAuthenticated, async (req, res) => {
    try {
        const { title, description, status } = req.body;
        let imageUrl = '';

        if (req.files && req.files.length > 0) {
            const file = req.files[0];
            imageUrl = await uploadFileToFirebase(
                file.buffer, file.originalname, file.mimetype
            );
        }

        await db.collection('items').add({
            title,
            description,
            status: status || 'draft',
            imageUrl,
            authorId: req.session.user.id,
            createdAt: admin.firestore.FieldValue.serverTimestamp(),
            updatedAt: admin.firestore.FieldValue.serverTimestamp()
        });

        res.redirect('/items');
    } catch (err) {
        console.error('Error creating item:', err);
        res.redirect('/items/add');
    }
});
```

### Update

```javascript
app.post('/items/edit/:id', isAuthenticated, async (req, res) => {
    try {
        const { title, description, status } = req.body;
        const updates = {
            title,
            description,
            status,
            updatedAt: admin.firestore.FieldValue.serverTimestamp()
        };

        if (req.files && req.files.length > 0) {
            const file = req.files[0];
            updates.imageUrl = await uploadFileToFirebase(
                file.buffer, file.originalname, file.mimetype
            );
        }

        await db.collection('items').doc(req.params.id).update(updates);
        res.redirect('/items');
    } catch (err) {
        console.error('Error updating item:', err);
        res.redirect(`/items/edit/${req.params.id}`);
    }
});
```

### Soft Delete

```javascript
app.post('/items/delete/:id', isAuthenticated, async (req, res) => {
    await db.collection('items').doc(req.params.id).update({
        status: 'deleted',
        deletedAt: admin.firestore.FieldValue.serverTimestamp()
    });
    res.redirect('/items');
});
```

## File Upload to Firebase Storage

```javascript
async function uploadFileToFirebase(fileBuffer, fileName, mimeType, folder = 'media') {
    const file = bucket.file(`${folder}/${fileName}`);
    await file.save(fileBuffer, {
        metadata: { contentType: mimeType },
        public: true
    });
    const [url] = await file.getSignedUrl({
        action: 'read',
        expires: '03-01-2500'
    });
    return url;
}
```

## Busboy File Upload Middleware

Required for Firebase Cloud Functions (multer doesn't work there):

```javascript
const filesUpload = (req, res, next) => {
    const contentType = req.headers['content-type'];
    if (req.method !== 'POST' || !contentType || !contentType.startsWith('multipart/form-data')) {
        return next();
    }

    const bb = busboy({ headers: req.headers, limits: { fileSize: 10 * 1024 * 1024 } });
    const fields = {};
    const files = [];
    const fileWrites = [];
    const tmpdir = os.tmpdir();

    bb.on('field', (key, value) => { fields[key] = value; });

    bb.on('file', (fieldname, file, info) => {
        const { filename, encoding, mimeType } = info;
        const filepath = path.join(tmpdir, filename || `file_${Date.now()}`);
        const writeStream = fs.createWriteStream(filepath);
        file.pipe(writeStream);

        fileWrites.push(new Promise((resolve, reject) => {
            file.on('end', () => writeStream.end());
            writeStream.on('finish', () => {
                fs.readFile(filepath, (err, buffer) => {
                    if (err) return reject(err);
                    files.push({
                        fieldname,
                        originalname: filename,
                        encoding,
                        mimetype: mimeType,
                        buffer,
                        size: Buffer.byteLength(buffer)
                    });
                    try { fs.unlinkSync(filepath); } catch (_) {}
                    resolve();
                });
            });
            writeStream.on('error', reject);
        }));
    });

    bb.on('finish', () => {
        Promise.all(fileWrites).then(() => {
            req.body = fields;
            req.files = files;
            next();
        }).catch(next);
    });

    if (req.rawBody) { bb.end(req.rawBody); } else { req.pipe(bb); }
};

app.use(filesUpload);
```

## Authentication with Login Lockout

```javascript
app.post('/login', async (req, res) => {
    const { email, password } = req.body;
    const snapshot = await db.collection('admins').where('email', '==', email).limit(1).get();
    if (snapshot.empty) return res.render('login', { layout: false, error: 'Invalid credentials' });

    const adminDoc = snapshot.docs[0];
    const adminData = adminDoc.data();

    // Check lockout
    const securityDoc = await db.collection('settings').doc('security').get();
    const maxAttempts = securityDoc.exists ? securityDoc.data().maxLoginAttempts || 5 : 5;
    const lockoutMinutes = securityDoc.exists ? securityDoc.data().lockoutDuration || 15 : 15;

    if (adminData.failedAttempts >= maxAttempts) {
        const lockedUntil = adminData.lastFailedAt?.toDate();
        if (lockedUntil && (Date.now() - lockedUntil.getTime()) < lockoutMinutes * 60 * 1000) {
            return res.render('login', { layout: false, error: 'Account locked. Try again later.' });
        }
        await adminDoc.ref.update({ failedAttempts: 0 });
    }

    const match = await bcrypt.compare(password, adminData.password);
    if (!match) {
        await adminDoc.ref.update({
            failedAttempts: (adminData.failedAttempts || 0) + 1,
            lastFailedAt: admin.firestore.FieldValue.serverTimestamp()
        });
        return res.render('login', { layout: false, error: 'Invalid credentials' });
    }

    await adminDoc.ref.update({ failedAttempts: 0, lastLogin: admin.firestore.FieldValue.serverTimestamp() });
    req.session.user = { id: adminDoc.id, name: adminData.name, email: adminData.email, role: adminData.role };
    res.redirect('/dashboard');
});
```

## Maintenance Mode Middleware

```javascript
let maintenanceMode = false;
async function refreshMaintenance() {
    const doc = await db.collection('settings').doc('general').get();
    maintenanceMode = doc.exists ? doc.data().maintenance_mode === true : false;
}
refreshMaintenance();
setInterval(refreshMaintenance, 30000);

function maintenanceGuard(req, res, next) {
    if (maintenanceMode && !req.session?.user && req.query.preview !== '1') {
        return res.render('maintenance', { layout: false });
    }
    next();
}
```

## Cloud Functions Deployment

```javascript
// At bottom of server.js — conditional start
if (require.main === module) {
    const PORT = process.env.PORT || 3000;
    app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
}
module.exports = { app };

// functions/index.js
const { onRequest } = require('firebase-functions/v2/https');
const { onSchedule } = require('firebase-functions/v2/scheduler');
const { app } = require('./server');

exports.app = onRequest({ timeoutSeconds: 300, memory: '1GiB' }, app);
```

## Firestore Indexes

Create `firestore.indexes.json` for composite queries:

```json
{
  "indexes": [
    {
      "collectionGroup": "items",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "items",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "authorId", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

## Firestore Security Rules

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isAdmin() {
      return request.auth != null &&
        get(/databases/$(database)/documents/admins/$(request.auth.uid)).data.role == 'admin';
    }
    match /{collection}/{document=**} {
      allow read, write: if isAdmin();
    }
  }
}
```
