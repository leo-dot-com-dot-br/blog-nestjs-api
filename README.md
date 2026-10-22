# Blog NestJS API

REST API for the blog. It handles authentication, users, posts and image uploads, and is consumed by the Next.js front end.

> Frontend repository: [blog-nextjs](https://github.com/leo-dot-com-dot-br/blog-nextjs)

## Features

- JWT authentication: login with email and password
- User sign up and self-management under `/user/me`: update profile, change password (requires the current one), delete account
- Posts: public listing and reading by slug; authenticated users create, list, read, update and delete their own posts
- Draft workflow: new posts are saved unpublished, and the author reviews and publishes them later
- Slug generated from the title when a post is created
- Image upload with validation of the real file type
- Global rate limiting, security headers, CORS whitelist and a standardized error format

## Tech Stack

- NestJS 11, TypeScript
- TypeORM with SQLite (`better-sqlite3`)
- Passport (`passport-jwt`) and `@nestjs/jwt` for authentication, `bcryptjs` for password hashing
- `class-validator` and `class-transformer` for input validation
- `helmet`, `@nestjs/throttler` and CORS for hardening
- `file-type` for upload validation, `@nestjs/serve-static` for serving images
- Jest and Supertest for tests

## Architecture

```text
Next.js (Server Components / Server Actions)  ->  NestJS API  ->  SQLite
                                                      |
                                                      +->  uploads/ (image files on disk)
```

The API is the only layer that touches the database. The front end sends requests with the JWT issued here, and every protected route is validated by the API's JWT guard. Uploaded images are written to disk, and the upload endpoint returns a relative path that the front end combines with its own `IMAGE_SERVER_URL`.

## Technical Decisions

**Upload validation by content, not by declared type.** The `mimetype` sent by the client can be forged, so the service reads the file buffer with `file-type` and checks the detected type. Only PNG, JPEG, WebP and GIF are accepted, with a 900 KB size limit. The stored extension comes from the detected type, never from the original file name.

**Unpredictable file names, organized by date.** Files are saved as `uploads/<YYYY-MM-DD>/<timestamp>-<random suffix>.<ext>`. The original name is discarded, which avoids collisions and path tricks.

**Strict input validation.** The global `ValidationPipe` runs with `whitelist` and `forbidNonWhitelisted`, so properties that are not declared in a DTO are rejected instead of silently reaching the service.

**CORS by whitelist.** Allowed origins come from the `CORS_WHITELIST` environment variable (comma-separated). Requests from any other origin are blocked.

**Security headers with cross-origin images.** Helmet is enabled, but `Cross-Origin-Resource-Policy` is set to `cross-origin` so the front end, which runs on another origin, can display the uploaded images.

**Schema sync is a development convenience.** `DB_SYNCHRONIZE=1` lets TypeORM create and alter tables automatically. It must be disabled in production (see [Database](#database)).

**Post management is tied to the logged-in user.** Everything that creates or changes posts lives under `/post/me` and works on behalf of the authenticated user. Public routes are read-only and find posts by slug.

**Draft first, publish later.** A new post is saved as unpublished. The author can review it and then publish it by updating `published`, so nothing goes public by accident.

**One error format for the whole API.** A global exception filter (`AllExceptionsFilter`) turns every error into `{ message: string[], error, statusCode }`, which is also the shape of validation errors. Expected HTTP errors are logged as warnings. Unexpected errors are logged with the stack trace and returned to the client as a generic 500, so internal details never leak.

**Global rate limiting.** `ThrottlerGuard` is registered for the whole application: each client can make 10 requests per 10 seconds, and is blocked for 5 seconds after exceeding that. It protects every route, including `/auth/login`, against brute-force attempts and abuse.

## Project Structure

| Path | Purpose |
| --- | --- |
| `src/main.ts` | Bootstrap: Helmet, CORS, global validation pipe |
| `src/app.module.ts` | Root module |
| `src/auth` | Login, JWT strategy and `JwtAuthGuard` |
| `src/user` | User entity, DTOs, service and controller |
| `src/post` | Post entity, DTOs, service and controller |
| `src/upload` | Upload controller, service and Multer configuration |
| `src/common/hashing` | Hashing abstraction with a bcrypt implementation |
| `src/common/filters` | Global exception filter |
| `src/common/pipes` | Custom pipes |
| `src/common/utils` | Helpers: slug generation, CORS whitelist parsing, random suffix |
| `rest-client/requests.http` | Ready-made HTTP requests for manual testing |
| `dev/images` | Sample images used by `rest-client/requests.http` to test uploads |
| `uploads/` | Image files written at runtime (ignored by git) |

## API Endpoints

Protected routes expect the header `Authorization: Bearer <accessToken>`, where the token comes from `POST /auth/login`.

| Method | Route | Auth | Description |
| --- | --- | --- | --- |
| `POST` | `/auth/login` | No | Log in with email and password, returns `accessToken` |
| `POST` | `/user` | No | Create a user |
| `GET` | `/user/me` | Yes | Read the current user |
| `PATCH` | `/user/me` | Yes | Update name and email |
| `PATCH` | `/user/me/password` | Yes | Change password (`currentPassword`, `newPassword`) |
| `DELETE` | `/user/me` | Yes | Delete the account |
| `GET` | `/post` | No | List published posts |
| `GET` | `/post/:slug` | No | Read a post by slug |
| `POST` | `/post/me` | Yes | Create a post |
| `GET` | `/post/me` | Yes | List your posts |
| `GET` | `/post/me/:id` | Yes | Read one of your posts |
| `PATCH` | `/post/me/:id` | Yes | Update one of your posts (including `published`) |
| `DELETE` | `/post/me/:id` | Yes | Delete one of your posts |
| `POST` | `/upload` | Yes | Upload an image (multipart, field `file`), returns `{ url }` |
| `GET` | `/uploads/:date/:file` | No | Serve an uploaded image |

Ready-made requests for all routes are in [`rest-client/requests.http`](rest-client/requests.http) (VS Code REST Client extension).

## Getting Started

### Prerequisites

- Node.js 20.9 or later
- npm

### Installation

```bash
git clone https://github.com/leo-dot-com-dot-br/blog-nestjs-api.git
cd blog-nestjs-api
npm install
cp .env-example .env
```

Set a strong value for `JWT_SECRET` in `.env`, then start the server:

```bash
npm run start:dev
```

The API runs on `http://localhost:3001` by default.

### Environment variables

| Variable | Description |
| --- | --- |
| `DB_TYPE` | TypeORM driver. Use `better-sqlite3` |
| `DB_DATABASE` | Path of the SQLite file (default `./db.sqlite`) |
| `DB_SYNCHRONIZE` | `1` syncs the schema automatically. **Set to `0` in production** |
| `DB_AUTO_LOAD_ENTITIES` | `1` loads entities registered in the modules automatically |
| `JWT_SECRET` | Secret used to sign tokens. Required, must be long and random |
| `JWT_EXPIRATION` | Token lifetime (for example `1d`) |
| `APP_PORT` | Port the API listens on (default `3001`) |
| `NODE_ENV` | Runtime environment |
| `CORS_WHITELIST` | Allowed origins, comma-separated (for example `http://localhost:3000`) |

### PostgreSQL variables (optional)

| Variable | Description |
| --- | --- |
| `DB_TYPE` | Anything other than `better-sqlite3` selects PostgreSQL |
| `DB_HOST` | Database host |
| `DB_PORT` | Database port (default `5432`) |
| `DB_USERNAME` | Database user |
| `DB_PASSWORD` | Database password |
| `DB_DATABASE` | Database name |

## Database

The project uses SQLite through TypeORM, so no external database is needed to run it locally. The file is created at `DB_DATABASE` on the first start.

The database configuration also has a PostgreSQL branch, selected when `DB_TYPE` is anything other than `better-sqlite3`. It is meant to make a future migration easy and is not part of the default setup: install the driver (`npm install pg`) and set the variables listed under [Environment variables](#environment-variables).

With `DB_SYNCHRONIZE=1` the schema follows the entities automatically, which is convenient in development but can alter or drop data when entities change. In production, disable it.

## Scripts

| Command | Description |
| --- | --- |
| `npm run start:dev` | Start in watch mode |
| `npm run build` | Compile to `dist/` |
| `npm run start:prod` | Run the compiled build |
| `npm run lint` | Run ESLint (with `--fix`) |
| `npm run format` | Format the code with Prettier |
| `npm test` | Run unit tests |
| `npm run test:cov` | Run unit tests with coverage |
| `npm run test:e2e` | Run end-to-end tests |
