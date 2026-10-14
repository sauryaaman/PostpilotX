# PostPilot X

PostPilot X is a client-oriented social publishing app. An admin can create client records, connect a client's LinkedIn account with OAuth, publish text/image/video posts immediately, and schedule posts for later. The repository contains a React/Vite frontend (`fronted/`) and an Express/MongoDB backend (`backend/`).

> **Implementation status:** LinkedIn publishing and scheduling are the implemented publishing integrations. Facebook, Instagram, and other platforms appear in parts of the UI/data model, but their OAuth and publishing APIs are not implemented. Some dashboard sample data is mock data. See [Known limitations](#known-limitations) before treating the project as production-ready.

## Contents

- [Architecture](#architecture)
- [Requirements](#requirements)
- [Configuration](#configuration)
- [Run locally](#run-locally)
- [How the application works](#how-the-application-works)
- [Frontend pages](#frontend-pages)
- [HTTP API](#http-api)
- [Data model](#data-model)
- [Known limitations](#known-limitations)
- [Troubleshooting](#troubleshooting)

## Architecture

```text
Browser (React 19, Vite, port 5173)
	├── Admin signup/login; JWT held in localStorage
	├── Client list and profile management
	├── LinkedIn authorization-link flow
	└── Create, schedule, list, and delete LinkedIn posts
						 │ HTTP + Bearer JWT (API URL is currently hardcoded)
						 ▼
Express 5 API (port 5000)
	├── JWT middleware and controllers
	├── Mongoose models ───────────── MongoDB
	├── Client/admin profile images ─ Cloudinary
	├── LinkedIn OAuth + REST API ── LinkedIn
	└── node-cron scheduler (every minute)
```

Important folders:

| Path | Responsibility |
| --- | --- |
| `backend/app.js` | Loads environment, middleware, API routes, callback redirect, scheduler, and starts the server after MongoDB connects. |
| `backend/routes/` | Express route definitions for auth, admins, clients, LinkedIn OAuth, and posts. |
| `backend/controllers/` | Request handlers and business logic. |
| `backend/models/` | Mongoose schemas for admins, clients, and LinkedIn posts. |
| `backend/services/linkedinService.js` | Registers/uploads LinkedIn media and creates/deletes LinkedIn posts. |
| `backend/services/schedulePostlinkedin.js` | Finds due scheduled posts every minute and publishes them. |
| `fronted/src/App.jsx` | React page routes and dashboard route protection. |
| `fronted/src/context/AuthContext.jsx` | Reads the stored JWT and fetches the current admin profile. |
| `fronted/src/components/` | Dashboard, client, OAuth, and post UI. |

## Requirements

- Node.js 18 or newer and npm.
- A MongoDB database (local or hosted MongoDB Atlas).
- A Cloudinary account. The backend currently requires Cloudinary credentials during startup.
- A LinkedIn developer application with OAuth configured and access to the required LinkedIn products/scopes for profile identity and member social posting.

## Configuration

Create `backend/.env` (do not commit real credentials):

```dotenv
# Required database and application secrets
MONGO_URI=mongodb://127.0.0.1:27017/postpilotx
JWT_SECRET=replace-with-a-long-random-secret

# Required by backend/config/cloudinary.js
CLOUDINARY_CLOUD_NAME=your-cloudinary-cloud-name
CLOUDINARY_API_KEY=your-cloudinary-api-key
CLOUDINARY_API_SECRET=your-cloudinary-api-secret

# LinkedIn OAuth application credentials
LINKEDIN_CLIENT_ID=your-linkedin-client-id
LINKEDIN_CLIENT_SECRET=your-linkedin-client-secret
LINKEDIN_REDIRECT_URI=http://localhost:5000/oauth/linkedin/callback

# Frontend destination after OAuth (defaults to http://localhost:5173)
FRONTEND_URL=http://localhost:5173

# Optional; defaults to 5000
PORT=5000
```

In the LinkedIn developer application's authorized redirect URLs, register the exact value of `LINKEDIN_REDIRECT_URI`. For local development that is normally `http://localhost:5000/oauth/linkedin/callback`. For a deployed app, use the publicly reachable HTTPS backend callback URL and set `FRONTEND_URL` to the deployed frontend origin.

The frontend currently calls `http://localhost:5000` directly. There is no configured Vite proxy or frontend API-base environment variable, so changing only `PORT` does **not** change the frontend API target. Update the frontend API URLs (or add a shared configurable API base) when using a different backend host/port.

There is no root npm workspace script and no checked-in `.env.example`; install and run each app from its own directory as shown below.

## Run locally

1. Start MongoDB and create the backend environment file described above.
2. Install and start the API from the backend directory:

	 ```bash
	 cd backend
	 npm install
	 npm run aaa
	 ```

	 `npm run aaa` runs `node app.js`. The declared `dev` script invokes `nodemon`, but `nodemon` is not declared in this package's dependencies; use `npm run aaa` unless you have installed nodemon separately.

3. In another terminal, install and start the frontend:

	 ```bash
	 cd fronted
	 npm install
	 npm run dev
	 ```

4. Open the Vite URL (normally `http://localhost:5173`), create an admin account, add a client, and connect that client's LinkedIn account.

The backend exits if MongoDB cannot connect or required Cloudinary settings are absent. Before relying on these commands, check the case-sensitive import issues in [Known limitations](#known-limitations); they currently prevent a clean startup/build on Linux.

## How the application works

### 1. Admin authentication

The signup form sends `name`, `phone`, `email`, and `password` to the backend. The Admin model hashes passwords with bcrypt before saving. Signup and login return a JWT that expires after seven days. The frontend stores it as `localStorage.token`; protected API requests send `Authorization: Bearer <token>`. `protect` verifies the JWT, loads the Admin record, and puts it on `req.admin`. On page load, `AuthContext` uses the token to fetch the admin profile; invalid tokens are removed from local storage.

### 2. Client management

The dashboard's client list fetches clients belonging to the authenticated admin. Creating a client sends multipart form data, including an optional `profileImage`; the configured Cloudinary multer storage uploads that image. Client profile updates support a replacement image and profile edits. A client record contains per-platform connection fields, including LinkedIn's connection flag, token, profile ID, and profile details.

### 3. LinkedIn authorization

1. From a selected client dashboard, the UI requests `GET /api/oauth/linkedin/connect/:clientId` with the admin JWT.
2. The API generates a LinkedIn authorization URL. Its `state` contains the client ID plus a timestamp/random suffix, and the URL requests `openid`, `profile`, `email`, and `w_member_social` scopes.
3. The client opens the URL, signs in to LinkedIn, and approves access. LinkedIn redirects to `LINKEDIN_REDIRECT_URI` with `code` and `state`.
4. The backend's `/oauth/linkedin/callback` redirect handler forwards those query parameters to `/api/oauth/linkedin/callback`. The OAuth callback exchanges the code for an access token, calls LinkedIn's `/v2/userinfo`, and saves the LinkedIn account details on the client.
5. The backend redirects the browser to `${FRONTEND_URL}/dashboard?oauth=linkedin-success` (or `linkedin-error`).

The access token is stored in MongoDB on the client. The current callback does not persist a refresh token; the separate token-manager utility exists, but automatic refresh is not wired into post publishing.

### 4. Immediate post with optional media

The create-post UI sends `multipart/form-data` to `POST /api/posts/create`, including `content`, `clientId`, and optionally one `image` or `video` file. Multer temporarily writes post media under `backend/uploads/`. The controller checks that the client has LinkedIn connected, registers the asset with LinkedIn, uploads the file, and then calls LinkedIn's UGC Posts API. After success, the backend stores the post and LinkedIn post ID in MongoDB, removes the temporary file, and returns the saved post.

### 5. Scheduled post

When `scheduledFor` is supplied as an ISO date string, the API uploads media to LinkedIn immediately (if any) and stores the post with `status: "scheduled"`; it does not publish the post at creation time. A `node-cron` job runs every minute, queries scheduled records whose `scheduledFor` is due, loads each client's saved LinkedIn credentials, and publishes the post. Success changes the record to `posted` and sets `postedAt`/`linkedinPostId`; failures change it to `failed` and save the error message. The scheduler runs inside the API process, so at least one backend process must remain running for scheduled posts to be sent.

### 6. Listing and deleting posts

The dashboard requests posts for the selected client. Deleting a locally saved post also attempts to delete it from LinkedIn if it was already posted; a LinkedIn delete error is logged, but the local database record is still deleted. See the ownership warning below: the post list/delete handlers currently do not enforce that the requested client/post belongs to the authenticated admin.

## Frontend pages

| Browser path | Page | Access |
| --- | --- | --- |
| `/` | Landing page | Public |
| `/auth` | Admin signup | Public |
| `/login` | Admin login | Public |
| `/services` | Services page | Public |
| `/about` | About page | Public |
| `/contact` | Contact page | Public |
| `/blog` | Blog page | Public |
| `/dashboard` | Client and LinkedIn-post dashboard | Protected by the frontend auth context |
| `/Admin` | Admin profile page | Protected by the frontend auth context |

Unknown paths redirect to `/`. The separate `/auth/linkedin/callback` React component exists, but the configured OAuth callback flow is handled by the backend callback routes described above.

## HTTP API

Base URL for the current frontend: `http://localhost:5000`. Unless stated otherwise, protected endpoints require:

```http
Authorization: Bearer <JWT>
```

JSON endpoints use `Content-Type: application/json`. File-upload endpoints use `multipart/form-data`; do not manually set the multipart boundary when using `fetch`/`FormData`.

### Authentication

| Method | Endpoint | Auth | Request | Purpose |
| --- | --- | --- | --- | --- |
| `POST` | `/api/auth/signup` | No | JSON: `name`, `phone`, `email`, `password` | Create an admin; returns admin fields and a 7-day JWT. |
| `POST` | `/api/auth/login` | No | JSON: `email`, `password` | Verify credentials and return admin fields and JWT. |
| `GET` | `/api/auth/getAdminProfile` | Yes | None | Return the current admin profile without the password. |

Example signup:

```bash
curl -X POST http://localhost:5000/api/auth/signup \
	-H 'Content-Type: application/json' \
	-d '{"name":"Demo Admin","phone":"555-123-4567","email":"admin@example.com","password":"change-me-123"}'
```

### Admin profile

| Method | Endpoint | Auth | Request | Purpose |
| --- | --- | --- | --- | --- |
| `PUT` | `/api/admin/profile` | Yes | Multipart: optional `name`, `email`, `phone`, `profileImage` | Update the authenticated admin profile. |
| `DELETE` | `/api/admin/delete` | Yes | JSON: `password` | Verify password and delete the authenticated admin. |

### Clients

| Method | Endpoint | Auth | Request | Purpose |
| --- | --- | --- | --- | --- |
| `POST` | `/api/client` | Yes | Multipart: `name`, `email`, `phone`, optional `profileImage`, optional `socialMedia` | Create a client owned by the current admin. |
| `GET` | `/api/client` | Yes | None | List clients owned by the current admin. |
| `GET` | `/api/client/:id` | Yes | Path: client MongoDB ID | Fetch one client, scoped to the current admin. |
| `PUT` | `/api/client/update/:id` | Yes | Multipart: optional `name`, `email`, `phone`, `profileImage` | Update a client. The current handler looks up by ID but does not verify ownership. |
| `DELETE` | `/api/client/delete/:id` | Yes | Path: client MongoDB ID | Delete a client. The current handler does not verify ownership. |

Client schema requires `name`, `email`, `phone`, and `addedBy`; email is unique. The frontend add-client form labels phone as optional, but the backend model requires it.

### LinkedIn OAuth

| Method | Endpoint | Auth | Request | Purpose |
| --- | --- | --- | --- | --- |
| `GET` | `/api/oauth/linkedin/connect/:clientId` | Yes | Path: client ID | Generate the authorization URL used by the dashboard. |
| `GET` | `/api/oauth/linkedin/generate-auth/:clientId` | Yes | Path: client ID | Alternate authorization URL endpoint; it uses a simpler state value. |
| `GET` | `/api/oauth/linkedin/callback` | No | Query: `code`, `state` | Exchange LinkedIn code, save the LinkedIn profile/token, redirect to frontend dashboard. |
| `GET` | `/oauth/linkedin/callback` | No | Query: `code`, `state` | Registered LinkedIn redirect handler; forwards to the API callback route. |

OAuth callback endpoints are public by design because LinkedIn redirects the browser to them. The callback `state` should be validated against a server-side value in a hardened OAuth implementation; the current code extracts a client ID from it but does not validate a stored state/CSRF token.

### Posts

| Method | Endpoint | Auth | Request | Purpose |
| --- | --- | --- | --- | --- |
| `POST` | `/api/posts/create` | Yes | Multipart: `content`, `clientId`, optional ISO `scheduledFor`, optional single `image` or `video` | Publish now or create a scheduled LinkedIn post. |
| `GET` | `/api/posts/client/:clientId` | Yes | Path: client ID | List posts for a client, newest first. Ownership is not checked by the current handler. |
| `DELETE` | `/api/posts/:postId` | Yes | Path: post MongoDB ID | Attempt LinkedIn deletion for posted items, then remove the database record. Ownership is not checked by the current handler. |

Create a text-only post immediately:

```bash
curl -X POST http://localhost:5000/api/posts/create \
	-H "Authorization: Bearer $TOKEN" \
	-F 'content=Hello from PostPilot X' \
	-F 'clientId=YOUR_CLIENT_OBJECT_ID'
```

Schedule a post with an image (use an ISO-8601 time with timezone):

```bash
curl -X POST http://localhost:5000/api/posts/create \
	-H "Authorization: Bearer $TOKEN" \
	-F 'content=Scheduled post from PostPilot X' \
	-F 'clientId=YOUR_CLIENT_OBJECT_ID' \
	-F 'scheduledFor=2026-10-20T15:00:00.000Z' \
	-F 'image=@./post-image.jpg'
```

The client must already have LinkedIn connected. A post request that fails validation or LinkedIn publishing currently returns HTTP `500` with a JSON `message`; successful creation returns HTTP `201` with `{ "success": true, "message": ..., "post": ... }`.

## Data model

| Collection | Important fields | Notes |
| --- | --- | --- |
| `admins` | `name`, `phone`, unique lowercase `email`, bcrypt-hashed `password`, `profileImage`, timestamps | JWT references the admin's MongoDB ID. Password minimum length is six characters in the schema. |
| `clients` | `name`, unique `email`, `phone`, `profileImage`, `socialMedia`, required `addedBy`, timestamps | `socialMedia.linkedin` holds `connected`, access token, profile ID/email/name/picture, and token expiry. Facebook/Instagram fields exist in the schema but are not backed by integrations here. |
| `posts` | `content`, `clientId`, `mediaType`, `mediaAsset`, `status`, `scheduledFor`, `linkedinPostId`, `postedAt`, `error`, timestamps | Allowed persisted statuses are `scheduled`, `posted`, `failed`; media type is `none`, `image`, or `video`. |

## Known limitations

- **Case-sensitive import paths currently block Linux startup/build.** `backend/app.js` imports `./routes/adminRoutes`, but the file is `backend/routes/AdminRoutes.js`; that route file imports `../controllers/adminController`, but the file is `AdminController.js`. The frontend imports `./pages/Login` and `./components/LinkedInCallback`, while the files are `login.jsx` and `linkedinCallBack.jsx`. Align import strings with actual filenames (or rename files consistently) before running the app on a case-sensitive filesystem.
- Admin profile image updates/deletes use the config module as if it were the Cloudinary client; the module exports the client under its `cloudinary` property. Those image operations need the controller import corrected. Basic profile text updates do not use that uploader path.
- Client update/delete and post list/delete handlers require a valid admin JWT but do not check that the target record belongs to that admin. Add ownership checks before exposing this API to untrusted users.
- OAuth callback state is not checked against a stored, one-time state value. The token refresh utility is not integrated into posting, and the OAuth callback does not save a refresh token.
- Scheduled posts are handled in the web-server process, not a separate durable worker. Multiple backend replicas can process the same due post; there is no distributed lock/atomic claim or retry policy.
- The frontend uses a fixed `http://localhost:5000` API origin. It needs configuration for deployed environments and HTTPS.
- Dashboard code contains mock clients/posts and UI controls for non-LinkedIn networks. Do not assume those screens represent data returned by the backend or that those networks can publish.
- The frontend callback component posts to the callback URL as JSON, while the implemented LinkedIn callback is a backend `GET` handler. The normal backend OAuth redirect flow does not need that React callback component.
- There is no automated test suite configured in `backend/package.json`; its `test` script intentionally exits with “no test specified.”

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Backend exits immediately | Confirm all Cloudinary variables exist in `backend/.env`; check MongoDB is reachable at `MONGO_URI`; fix the Admin route/controller import casing. |
| Frontend build reports unresolved imports | Fix the `Login`/`login.jsx` and `LinkedInCallback`/`linkedinCallBack.jsx` casing mismatch. |
| Browser reports connection refused | Start backend on port 5000; the frontend API origin is hardcoded to that port. |
| API returns `401` | Send the current token as `Authorization: Bearer <JWT>`; sign in again if the seven-day token expired. |
| LinkedIn connect/post fails | Verify client ID/secret, exact redirect URI, LinkedIn app products/scopes, client connection state, and whether the LinkedIn token is still valid. |
| Scheduled post stays scheduled | Keep the backend process running, check `scheduledFor` is in the past and inspect backend logs/MongoDB `error` and `status` fields. |