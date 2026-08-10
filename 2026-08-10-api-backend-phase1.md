# API Backend Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `/api` backend for Phase 1 (MVP): PostgreSQL schema, admin auth, payment link CRUD, PayDunya integration via an adapter interface, webhook processing, and transactional emails.

**Architecture:** Express + TypeScript service backed by PostgreSQL via Prisma. All PSP interaction goes through a `PSPAdapter` interface implemented by `PayDunyaAdapter`, so the PSP can be swapped without touching routes or services. See [2026-08-10-architecture-phase1-design.md](../specs/2026-08-10-architecture-phase1-design.md) for the full architecture spec.

**Tech Stack:** Node.js, Express, TypeScript, Prisma + PostgreSQL, argon2, jsonwebtoken, zod, Resend, Jest + Supertest.

## Global Constraints

- No card data (number, expiry, CVV) is ever stored or transits through this API — enforced by keeping the Prisma schema free of any such field (spec §4, cahier §6/§12).
- Currency defaults to XOF (cahier §4.1).
- Passwords hashed with argon2 (spec §6).
- Admin sessions: JWT in an `httpOnly`, `secure` cookie, ~2h expiry (spec §6).
- Every sensitive admin action (login, create lien, deactivate lien) is written to `AuditLog` (spec §6, cahier §6).
- Webhook signature must be verified before any transaction is written (spec §7, cahier §12).
- A deactivated or expired lien must return HTTP 410 and be unusable for payment (spec §7, cahier §12 acceptance criteria).
- All user-facing text (error messages, emails) is in French (cahier §6 — localisation).

## Prerequisites (one-time, before Task 1)

- Node.js 20+ installed.
- A local PostgreSQL instance reachable at the URL you'll put in `.env` — either install PostgreSQL locally or run:
  ```bash
  docker run --name lien-paiement-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=lien_paiement -p 5432:5432 -d postgres:16
  ```

---

### Task 1: Monorepo and API scaffold

**Files:**
- Create: `package.json` (root)
- Create: `api/package.json`
- Create: `api/tsconfig.json`
- Create: `api/jest.config.js`
- Create: `api/.env.example`
- Create: `api/src/app.ts`
- Create: `api/src/server.ts`
- Test: `api/tests/health.test.ts`

**Interfaces:**
- Produces: `createApp(): Express` (from `api/src/app.ts`) — every later task imports this to mount routes and to build the Supertest app in tests.

- [ ] **Step 1: Create the root workspace `package.json`**

```json
{
  "name": "lien-de-paiement-dashbord",
  "private": true,
  "workspaces": ["web", "api"]
}
```

- [ ] **Step 2: Create `api/package.json`**

```json
{
  "name": "api",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "test": "jest --runInBand",
    "seed": "ts-node prisma/seed.ts",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev"
  },
  "dependencies": {
    "@prisma/client": "^5.19.0",
    "argon2": "^0.31.2",
    "cookie-parser": "^1.4.6",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "jsonwebtoken": "^9.0.2",
    "resend": "^3.5.0",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/cookie-parser": "^1.4.7",
    "@types/cors": "^2.8.17",
    "@types/express": "^4.17.21",
    "@types/jest": "^29.5.12",
    "@types/jsonwebtoken": "^9.0.6",
    "@types/node": "^20.14.10",
    "@types/supertest": "^6.0.2",
    "jest": "^29.7.0",
    "prisma": "^5.19.0",
    "supertest": "^7.0.0",
    "ts-jest": "^29.2.2",
    "ts-node": "^10.9.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.5.3"
  }
}
```

- [ ] **Step 3: Create `api/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "dist",
    "rootDir": ".",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src", "prisma", "tests"]
}
```

- [ ] **Step 4: Create `api/jest.config.js`**

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/tests'],
  testTimeout: 15000,
};
```

- [ ] **Step 5: Create `api/.env.example`**

```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/lien_paiement?schema=public"
JWT_SECRET="changeme-generate-a-long-random-string"
PAYDUNYA_MASTER_KEY=""
PAYDUNYA_PRIVATE_KEY=""
PAYDUNYA_PUBLIC_KEY=""
PAYDUNYA_TOKEN=""
PAYDUNYA_MODE="test"
RESEND_API_KEY=""
EMAIL_FROM="paiements@hybusinessandco.com"
ADMIN_NOTIFICATION_EMAIL="ops@hybusinessandco.com"
APP_BASE_URL="http://localhost:3000"
PORT=4000
SEED_ADMIN_EMAIL="admin@hybusinessandco.com"
SEED_ADMIN_PASSWORD="ChangeMe123!"
```

Copy it: `cp api/.env.example api/.env` and fill in real values later (PayDunya/Resend keys arrive during Phase 0 cadrage).

- [ ] **Step 6: Write the failing test**

Create `api/tests/health.test.ts`:

```typescript
import request from 'supertest';
import { createApp } from '../src/app';

describe('GET /api/health', () => {
  it('returns status ok', async () => {
    const app = createApp();
    const res = await request(app).get('/api/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: 'ok' });
  });
});
```

- [ ] **Step 7: Run test to verify it fails**

Run: `cd api && npm install && npm test`
Expected: FAIL — `Cannot find module '../src/app'`

- [ ] **Step 8: Implement `api/src/app.ts`**

```typescript
import express from 'express';
import cors from 'cors';
import cookieParser from 'cookie-parser';

export function createApp() {
  const app = express();
  app.use(cors({ origin: process.env.APP_BASE_URL, credentials: true }));
  app.use(express.json());
  app.use(cookieParser());

  app.get('/api/health', (_req, res) => {
    res.json({ status: 'ok' });
  });

  return app;
}
```

- [ ] **Step 9: Implement `api/src/server.ts`**

```typescript
import 'dotenv/config';
import { createApp } from './app';

const app = createApp();
const port = process.env.PORT ? Number(process.env.PORT) : 4000;
app.listen(port, () => {
  console.log(`API listening on port ${port}`);
});
```

- [ ] **Step 10: Run test to verify it passes**

Run: `npm test`
Expected: PASS — 1 test passed

- [ ] **Step 11: Commit**

```bash
git add package.json api/package.json api/tsconfig.json api/jest.config.js api/.env.example api/src/app.ts api/src/server.ts api/tests/health.test.ts
git commit -m "chore: scaffold api workspace with health check"
```

---

### Task 2: PostgreSQL schema and Prisma client

**Files:**
- Create: `api/prisma/schema.prisma`
- Create: `api/src/db/prisma.ts`
- Test: `api/tests/db.test.ts`

**Interfaces:**
- Consumes: nothing from prior tasks.
- Produces: `prisma` singleton (from `api/src/db/prisma.ts`), and the generated Prisma models `Client`, `LienPaiement`, `Transaction`, `AdminUser`, `AuditLog` — every subsequent service imports `prisma` from this file.

This task is infrastructure-first (schema must exist before any test can compile against generated types), so steps run schema → migrate → client → test, in that order.

- [ ] **Step 1: Write `api/prisma/schema.prisma`**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Client {
  id               String         @id @default(cuid())
  nom              String
  email            String?
  telephone        String?
  referenceDossier String?
  liens            LienPaiement[]
  createdAt        DateTime       @default(now())
}

model LienPaiement {
  id                    String        @id @default(cuid())
  reference             String        @unique
  montant               Decimal?      @db.Decimal(12, 2)
  montantLibre          Boolean       @default(false)
  devise                String        @default("XOF")
  description           String
  clientId              String
  client                Client        @relation(fields: [clientId], references: [id])
  modeAuthentification  String
  statutLien            String        @default("actif")
  usageUnique           Boolean       @default(true)
  dateExpiration        DateTime?
  urlToken              String        @unique
  transactions          Transaction[]
  createdAt             DateTime      @default(now())
  createdBy             String
}

model Transaction {
  id               String       @id @default(cuid())
  lienId           String
  lien             LienPaiement @relation(fields: [lienId], references: [id])
  idTransactionPSP String?      @unique
  pspNom           String
  montant          Decimal      @db.Decimal(12, 2)
  devise           String
  reseauCarte      String?
  modeAuthEffectif String?
  statut           String       @default("en_attente")
  messageErreur    String?
  webhookPayload   Json?
  dateCreation     DateTime     @default(now())
  dateMiseAJour    DateTime     @updatedAt
}

model AdminUser {
  id                String    @id @default(cuid())
  nom               String
  email             String    @unique
  motDePasseHash    String
  role              String    @default("admin")
  derniereConnexion DateTime?
  createdAt         DateTime  @default(now())
}

model AuditLog {
  id        String   @id @default(cuid())
  userId    String
  action    String
  cibleId   String?
  detail    Json?
  createdAt DateTime @default(now())
}
```

- [ ] **Step 2: Run the migration**

Run: `cd api && npx prisma migrate dev --name init`
Expected: Migration created under `prisma/migrations/`, all 5 tables created in the database, Prisma Client generated.

- [ ] **Step 3: Implement `api/src/db/prisma.ts`**

```typescript
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient();
```

- [ ] **Step 4: Write the test**

Create `api/tests/db.test.ts`:

```typescript
import { prisma } from '../src/db/prisma';

describe('Prisma client', () => {
  afterAll(async () => {
    await prisma.$disconnect();
  });

  it('creates and reads an AdminUser', async () => {
    const email = `test-${Date.now()}@example.com`;
    const user = await prisma.adminUser.create({
      data: { nom: 'Test Admin', email, motDePasseHash: 'hash' },
    });
    const found = await prisma.adminUser.findUnique({ where: { id: user.id } });
    expect(found?.email).toBe(email);
    await prisma.adminUser.delete({ where: { id: user.id } });
  });
});
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test`
Expected: PASS — 2 tests passed (health + db)

- [ ] **Step 6: Commit**

```bash
git add api/prisma api/src/db/prisma.ts api/tests/db.test.ts
git commit -m "feat: add Prisma schema and client singleton"
```

---

### Task 3: Password hashing and JWT utilities

**Files:**
- Create: `api/src/utils/password.ts`
- Create: `api/src/utils/jwt.ts`
- Test: `api/tests/utils/password.test.ts`
- Test: `api/tests/utils/jwt.test.ts`

**Interfaces:**
- Produces: `hashPassword(plain: string): Promise<string>`, `verifyPassword(hash: string, plain: string): Promise<boolean>`, `signAdminToken(payload: AdminTokenPayload): string`, `verifyAdminToken(token: string): AdminTokenPayload`, `interface AdminTokenPayload { sub: string; email: string }` — used by Task 4's auth service and middleware.

- [ ] **Step 1: Write the failing tests**

Create `api/tests/utils/password.test.ts`:

```typescript
import { hashPassword, verifyPassword } from '../../src/utils/password';

describe('password utils', () => {
  it('hashes a password and verifies it correctly', async () => {
    const hash = await hashPassword('Sup3rSecret!');
    expect(hash).not.toBe('Sup3rSecret!');
    await expect(verifyPassword(hash, 'Sup3rSecret!')).resolves.toBe(true);
    await expect(verifyPassword(hash, 'wrong')).resolves.toBe(false);
  });
});
```

Create `api/tests/utils/jwt.test.ts`:

```typescript
import { signAdminToken, verifyAdminToken } from '../../src/utils/jwt';

describe('jwt utils', () => {
  beforeAll(() => {
    process.env.JWT_SECRET = 'test-secret';
  });

  it('signs and verifies a token round-trip', () => {
    const token = signAdminToken({ sub: 'user-1', email: 'admin@example.com' });
    const payload = verifyAdminToken(token);
    expect(payload.sub).toBe('user-1');
    expect(payload.email).toBe('admin@example.com');
  });

  it('throws on a tampered token', () => {
    const token = signAdminToken({ sub: 'user-1', email: 'admin@example.com' });
    expect(() => verifyAdminToken(token + 'tampered')).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npm test`
Expected: FAIL — cannot find modules `../../src/utils/password` and `../../src/utils/jwt`

- [ ] **Step 3: Implement `api/src/utils/password.ts`**

```typescript
import argon2 from 'argon2';

export async function hashPassword(plain: string): Promise<string> {
  return argon2.hash(plain);
}

export async function verifyPassword(hash: string, plain: string): Promise<boolean> {
  return argon2.verify(hash, plain);
}
```

- [ ] **Step 4: Implement `api/src/utils/jwt.ts`**

```typescript
import jwt from 'jsonwebtoken';

export interface AdminTokenPayload {
  sub: string;
  email: string;
}

const EXPIRES_IN = '2h';

export function signAdminToken(payload: AdminTokenPayload): string {
  const secret = process.env.JWT_SECRET;
  if (!secret) throw new Error('JWT_SECRET is not set');
  return jwt.sign(payload, secret, { expiresIn: EXPIRES_IN });
}

export function verifyAdminToken(token: string): AdminTokenPayload {
  const secret = process.env.JWT_SECRET;
  if (!secret) throw new Error('JWT_SECRET is not set');
  return jwt.verify(token, secret) as AdminTokenPayload;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npm test`
Expected: PASS — 4 tests passed

- [ ] **Step 6: Commit**

```bash
git add api/src/utils/password.ts api/src/utils/jwt.ts api/tests/utils
git commit -m "feat: add password hashing and JWT utilities"
```

---

### Task 4: Async handler, auth service, login route, and auth middleware

**Files:**
- Create: `api/src/utils/asyncHandler.ts`
- Create: `api/src/services/auth.service.ts`
- Create: `api/src/services/audit.service.ts` (stub; replaced with a real implementation in Task 8)
- Create: `api/src/middleware/auth.ts`
- Create: `api/src/routes/auth.routes.ts`
- Modify: `api/src/app.ts` — mount `authRouter`
- Test: `api/tests/auth.test.ts`

**Interfaces:**
- Consumes: `prisma` (Task 2), `hashPassword`/`verifyPassword`/`signAdminToken`/`verifyAdminToken` (Task 3), `createApp` (Task 1).
- Produces: `asyncHandler(fn)` (used by every later route file), `requireAuth` middleware + `AuthenticatedRequest` type (used by Tasks 5–7, 11), `login(email, password)` and `InvalidCredentialsError` (used only inside this task's route).

- [ ] **Step 1: Write the failing test**

Create `api/tests/auth.test.ts`:

```typescript
import request from 'supertest';
import { createApp } from '../src/app';
import { prisma } from '../src/db/prisma';
import { hashPassword } from '../src/utils/password';

describe('POST /api/auth/login', () => {
  const app = createApp();
  const email = `admin-${Date.now()}@example.com`;

  beforeAll(async () => {
    process.env.JWT_SECRET = 'test-secret';
    await prisma.adminUser.create({
      data: { nom: 'Admin Test', email, motDePasseHash: await hashPassword('Sup3rSecret!') },
    });
  });

  afterAll(async () => {
    await prisma.adminUser.deleteMany({ where: { email } });
    await prisma.$disconnect();
  });

  it('returns 401 for wrong password', async () => {
    const res = await request(app).post('/api/auth/login').send({ email, password: 'wrong' });
    expect(res.status).toBe(401);
  });

  it('returns 200 and sets a session cookie for correct credentials', async () => {
    const res = await request(app).post('/api/auth/login').send({ email, password: 'Sup3rSecret!' });
    expect(res.status).toBe(200);
    expect(res.headers['set-cookie']?.[0]).toMatch(/session=/);
    expect(res.body.user.email).toBe(email);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `/api/auth/login` (route not mounted yet)

- [ ] **Step 3: Implement `api/src/utils/asyncHandler.ts`**

```typescript
import { Request, Response, NextFunction, RequestHandler } from 'express';

export function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<void>
): RequestHandler {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
}
```

- [ ] **Step 4: Implement `api/src/services/auth.service.ts`**

```typescript
import { prisma } from '../db/prisma';
import { verifyPassword } from '../utils/password';
import { signAdminToken } from '../utils/jwt';

export class InvalidCredentialsError extends Error {}

export async function login(email: string, password: string) {
  const user = await prisma.adminUser.findUnique({ where: { email } });
  if (!user) throw new InvalidCredentialsError();
  const valid = await verifyPassword(user.motDePasseHash, password);
  if (!valid) throw new InvalidCredentialsError();
  await prisma.adminUser.update({
    where: { id: user.id },
    data: { derniereConnexion: new Date() },
  });
  const token = signAdminToken({ sub: user.id, email: user.email });
  return { token, user: { id: user.id, nom: user.nom, email: user.email } };
}
```

- [ ] **Step 5: Implement `api/src/middleware/auth.ts`**

```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyAdminToken } from '../utils/jwt';

export interface AuthenticatedRequest extends Request {
  admin?: { id: string; email: string };
}

export function requireAuth(req: AuthenticatedRequest, res: Response, next: NextFunction) {
  const token = req.cookies?.session;
  if (!token) {
    return res.status(401).json({ error: 'Non authentifié' });
  }
  try {
    const payload = verifyAdminToken(token);
    req.admin = { id: payload.sub, email: payload.email };
    next();
  } catch {
    return res.status(401).json({ error: 'Session invalide ou expirée' });
  }
}
```

- [ ] **Step 6: Implement `api/src/routes/auth.routes.ts`**

```typescript
import { Router } from 'express';
import { z } from 'zod';
import { login, InvalidCredentialsError } from '../services/auth.service';
import { recordAudit } from '../services/audit.service';
import { asyncHandler } from '../utils/asyncHandler';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});

export const authRouter = Router();

authRouter.post(
  '/login',
  asyncHandler(async (req, res) => {
    const parsed = loginSchema.safeParse(req.body);
    if (!parsed.success) {
      res.status(400).json({ error: 'Email ou mot de passe manquant' });
      return;
    }
    try {
      const { token, user } = await login(parsed.data.email, parsed.data.password);
      res.cookie('session', token, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 2 * 60 * 60 * 1000,
      });
      await recordAudit(user.id, 'login');
      res.json({ user });
    } catch (err) {
      if (err instanceof InvalidCredentialsError) {
        res.status(401).json({ error: 'Identifiants invalides' });
        return;
      }
      throw err;
    }
  })
);
```

This route depends on `recordAudit`, defined in Task 8. Until Task 8 lands, stub it in `api/src/services/audit.service.ts` so this task compiles standalone:

```typescript
export async function recordAudit(_userId: string, _action: string, _cibleId?: string, _detail?: Record<string, unknown>) {
  // implemented in full in Task 8
}
```

- [ ] **Step 7: Mount the router in `api/src/app.ts`**

Add near the top of the file:

```typescript
import { authRouter } from './routes/auth.routes';
```

Add after the `/api/health` route:

```typescript
  app.use('/api/auth', authRouter);
```

- [ ] **Step 8: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 2 new tests pass

- [ ] **Step 9: Commit**

```bash
git add api/src/utils/asyncHandler.ts api/src/services/auth.service.ts api/src/services/audit.service.ts api/src/middleware/auth.ts api/src/routes/auth.routes.ts api/src/app.ts api/tests/auth.test.ts
git commit -m "feat: add admin login endpoint with session cookie"
```

---

### Task 5: Create payment link — `POST /api/liens`

**Files:**
- Create: `api/src/utils/token.ts`
- Create: `api/src/services/lien.service.ts`
- Create: `api/src/routes/liens.routes.ts`
- Modify: `api/src/app.ts` — mount `liensRouter`
- Test: `api/tests/liens.test.ts`

**Interfaces:**
- Consumes: `prisma`, `requireAuth`/`AuthenticatedRequest`, `asyncHandler`, `recordAudit` (stub from Task 4).
- Produces: `generateReference(prisma): Promise<string>`, `generateUrlToken(): string`, `createLien(input: CreateLienInput)` — `createLien` and its `CreateLienInput` type are consumed by Task 6/7's service extensions and Task 6's test fixtures.

- [ ] **Step 1: Write the failing test**

Create `api/tests/liens.test.ts`:

```typescript
import request from 'supertest';
import { createApp } from '../src/app';
import { prisma } from '../src/db/prisma';
import { hashPassword } from '../src/utils/password';

describe('liens routes', () => {
  const app = createApp();
  const email = `admin-liens-${Date.now()}@example.com`;
  const agent = request.agent(app);

  beforeAll(async () => {
    process.env.JWT_SECRET = 'test-secret';
    await prisma.adminUser.create({
      data: { nom: 'Admin', email, motDePasseHash: await hashPassword('Sup3rSecret!') },
    });
    await agent.post('/api/auth/login').send({ email, password: 'Sup3rSecret!' });
  });

  afterAll(async () => {
    await prisma.adminUser.deleteMany({ where: { email } });
    await prisma.$disconnect();
  });

  describe('POST /api/liens', () => {
    it('rejects unauthenticated requests', async () => {
      const res = await request(app).post('/api/liens').send({});
      expect(res.status).toBe(401);
    });

    it('creates a lien with a unique reference and urlToken', async () => {
      const res = await agent.post('/api/liens').send({
        montant: 50000,
        montantLibre: false,
        devise: 'XOF',
        description: 'Frais de dossier',
        modeAuthentification: 'STANDARD',
        usageUnique: true,
        client: { nom: 'Jean Client', email: 'jean@example.com' },
      });
      expect(res.status).toBe(201);
      expect(res.body.reference).toMatch(/^HYBC-\d{4}-\d{4}$/);
      expect(res.body.urlToken).toBeTruthy();
      expect(res.body.statutLien).toBe('actif');
    });

    it('rejects a fixed-amount request with no montant', async () => {
      const res = await agent.post('/api/liens').send({
        montantLibre: false,
        description: 'Frais',
        modeAuthentification: 'STANDARD',
        client: { nom: 'Jean' },
      });
      expect(res.status).toBe(400);
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `/api/liens` (route not mounted yet)

- [ ] **Step 3: Implement `api/src/utils/token.ts`**

```typescript
import { randomBytes } from 'crypto';
import { PrismaClient } from '@prisma/client';

export function generateUrlToken(): string {
  return randomBytes(24).toString('base64url');
}

export async function generateReference(
  prisma: Pick<PrismaClient, 'lienPaiement'>,
  year = new Date().getFullYear()
): Promise<string> {
  const count = await prisma.lienPaiement.count({
    where: { createdAt: { gte: new Date(`${year}-01-01T00:00:00.000Z`) } },
  });
  const seq = String(count + 1).padStart(4, '0');
  return `HYBC-${year}-${seq}`;
}
```

- [ ] **Step 4: Implement `api/src/services/lien.service.ts`**

```typescript
import { prisma } from '../db/prisma';
import { generateReference, generateUrlToken } from '../utils/token';

export interface CreateLienInput {
  montant?: number;
  montantLibre: boolean;
  devise: string;
  description: string;
  modeAuthentification: 'STANDARD' | '3DS';
  usageUnique: boolean;
  dateExpiration?: Date;
  createdBy: string;
  client: { nom: string; email?: string; telephone?: string; referenceDossier?: string };
}

export async function createLien(input: CreateLienInput) {
  const client = await prisma.client.create({ data: input.client });
  const reference = await generateReference(prisma);
  const urlToken = generateUrlToken();

  return prisma.lienPaiement.create({
    data: {
      reference,
      urlToken,
      montant: input.montantLibre ? null : input.montant,
      montantLibre: input.montantLibre,
      devise: input.devise,
      description: input.description,
      modeAuthentification: input.modeAuthentification,
      usageUnique: input.usageUnique,
      dateExpiration: input.dateExpiration,
      createdBy: input.createdBy,
      clientId: client.id,
    },
    include: { client: true },
  });
}
```

- [ ] **Step 5: Implement `api/src/routes/liens.routes.ts`**

```typescript
import { Router } from 'express';
import { z } from 'zod';
import { requireAuth, AuthenticatedRequest } from '../middleware/auth';
import { asyncHandler } from '../utils/asyncHandler';
import { recordAudit } from '../services/audit.service';
import { createLien } from '../services/lien.service';

const createLienSchema = z.object({
  montant: z.number().positive().optional(),
  montantLibre: z.boolean().default(false),
  devise: z.string().default('XOF'),
  description: z.string().min(1),
  modeAuthentification: z.enum(['3DS', 'STANDARD']),
  usageUnique: z.boolean().default(true),
  dateExpiration: z.string().datetime().optional(),
  client: z.object({
    nom: z.string().min(1),
    email: z.string().email().optional(),
    telephone: z.string().optional(),
    referenceDossier: z.string().optional(),
  }),
});

export const liensRouter = Router();

liensRouter.post(
  '/',
  requireAuth,
  asyncHandler(async (req: AuthenticatedRequest, res) => {
    const parsed = createLienSchema.safeParse(req.body);
    if (!parsed.success) {
      res.status(400).json({ error: parsed.error.flatten() });
      return;
    }
    if (!parsed.data.montantLibre && parsed.data.montant === undefined) {
      res.status(400).json({ error: 'Montant requis si montantLibre est faux' });
      return;
    }
    const lien = await createLien({
      ...parsed.data,
      dateExpiration: parsed.data.dateExpiration ? new Date(parsed.data.dateExpiration) : undefined,
      createdBy: req.admin!.id,
    });
    await recordAudit(req.admin!.id, 'creation_lien', lien.id);
    res.status(201).json(lien);
  })
);
```

- [ ] **Step 6: Mount the router in `api/src/app.ts`**

Add import: `import { liensRouter } from './routes/liens.routes';`
Add after the auth mount: `app.use('/api/liens', liensRouter);`

- [ ] **Step 7: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 3 new tests pass

- [ ] **Step 8: Commit**

```bash
git add api/src/utils/token.ts api/src/services/lien.service.ts api/src/routes/liens.routes.ts api/src/app.ts api/tests/liens.test.ts
git commit -m "feat: add POST /api/liens to create payment links"
```

---

### Task 6: List and fetch payment links — `GET /api/liens`, `GET /api/liens/:id`

**Files:**
- Modify: `api/src/services/lien.service.ts` — add `listLiens`, `isLienUsable`, `getLienById`
- Modify: `api/src/routes/liens.routes.ts` — add `GET /` and `GET /:id`
- Modify: `api/tests/liens.test.ts` — add coverage

**Interfaces:**
- Consumes: `createLien` (Task 5), `prisma`.
- Produces: `listLiens(filters): Promise<LienWithClient[]>`, `isLienUsable(lien): boolean`, `getLienById(id): Promise<LienWithRelations | null>` — `isLienUsable` and `getLienById` are consumed by Task 7's deactivate route and by the Phase 2 frontend/payment-page plan.

- [ ] **Step 1: Extend the failing test**

Add to `api/tests/liens.test.ts`, inside the outer `describe('liens routes', ...)` block:

```typescript
  describe('GET /api/liens', () => {
    it('lists liens created by the admin, most recent first', async () => {
      const res = await agent.get('/api/liens');
      expect(res.status).toBe(200);
      expect(Array.isArray(res.body)).toBe(true);
      expect(res.body.length).toBeGreaterThan(0);
    });
  });

  describe('GET /api/liens/:id', () => {
    it('returns 404 for an unknown id', async () => {
      const res = await request(app).get('/api/liens/unknown-id');
      expect(res.status).toBe(404);
    });

    it('returns 410 for an expired lien', async () => {
      const client = await prisma.client.create({ data: { nom: 'Client Expiré' } });
      const expiredLien = await prisma.lienPaiement.create({
        data: {
          reference: `HYBC-TEST-EXP-${Date.now()}`,
          urlToken: `token-exp-${Date.now()}`,
          montant: 1000,
          devise: 'XOF',
          description: 'Test expiration',
          modeAuthentification: 'STANDARD',
          usageUnique: true,
          createdBy: 'test',
          clientId: client.id,
          dateExpiration: new Date(Date.now() - 1000),
        },
      });
      const res = await request(app).get(`/api/liens/${expiredLien.id}`);
      expect(res.status).toBe(410);
    });

    it('returns the lien for a valid, active id', async () => {
      const created = await agent.post('/api/liens').send({
        montant: 10000,
        montantLibre: false,
        devise: 'XOF',
        description: 'Lien valide',
        modeAuthentification: 'STANDARD',
        usageUnique: true,
        client: { nom: 'Client Actif' },
      });
      const res = await request(app).get(`/api/liens/${created.body.id}`);
      expect(res.status).toBe(200);
      expect(res.body.id).toBe(created.body.id);
    });
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `GET /api/liens` and `GET /api/liens/:id` (routes not defined yet)

- [ ] **Step 3: Extend `api/src/services/lien.service.ts`**

Append:

```typescript
export interface ListLiensFilters {
  statut?: string;
  modeAuthentification?: string;
  search?: string;
}

export async function listLiens(filters: ListLiensFilters) {
  return prisma.lienPaiement.findMany({
    where: {
      statutLien: filters.statut,
      modeAuthentification: filters.modeAuthentification,
      OR: filters.search
        ? [
            { reference: { contains: filters.search, mode: 'insensitive' } },
            { client: { nom: { contains: filters.search, mode: 'insensitive' } } },
          ]
        : undefined,
    },
    include: { client: true },
    orderBy: { createdAt: 'desc' },
  });
}

export function isLienUsable(lien: { statutLien: string; dateExpiration: Date | null }): boolean {
  if (lien.statutLien !== 'actif') return false;
  if (lien.dateExpiration && lien.dateExpiration.getTime() < Date.now()) return false;
  return true;
}

export async function getLienById(id: string) {
  return prisma.lienPaiement.findUnique({
    where: { id },
    include: { client: true, transactions: true },
  });
}
```

- [ ] **Step 4: Extend `api/src/routes/liens.routes.ts`**

Update the import line to add the new service functions:

```typescript
import { createLien, listLiens, getLienById, isLienUsable } from '../services/lien.service';
```

Append the two routes:

```typescript
liensRouter.get(
  '/',
  requireAuth,
  asyncHandler(async (req, res) => {
    const { statut, modeAuthentification, search } = req.query;
    const liens = await listLiens({
      statut: typeof statut === 'string' ? statut : undefined,
      modeAuthentification: typeof modeAuthentification === 'string' ? modeAuthentification : undefined,
      search: typeof search === 'string' ? search : undefined,
    });
    res.json(liens);
  })
);

// Intentionally unauthenticated: the public payment page (Phase 2 frontend)
// calls this to render the recap before the client pays.
liensRouter.get(
  '/:id',
  asyncHandler(async (req, res) => {
    const lien = await getLienById(req.params.id);
    if (!lien) {
      res.status(404).json({ error: 'Lien introuvable' });
      return;
    }
    if (!isLienUsable(lien)) {
      res.status(410).json({ error: 'Lien expiré ou désactivé' });
      return;
    }
    res.json(lien);
  })
);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 4 new tests pass

- [ ] **Step 6: Commit**

```bash
git add api/src/services/lien.service.ts api/src/routes/liens.routes.ts api/tests/liens.test.ts
git commit -m "feat: add list and detail endpoints for payment links"
```

---

### Task 7: Deactivate payment link — `PATCH /api/liens/:id/desactiver`

**Files:**
- Modify: `api/src/services/lien.service.ts` — add `desactiverLien`
- Modify: `api/src/routes/liens.routes.ts` — add `PATCH /:id/desactiver`
- Modify: `api/tests/liens.test.ts` — add coverage

**Interfaces:**
- Consumes: `getLienById`, `isLienUsable` (Task 6), `requireAuth`, `recordAudit`.
- Produces: `desactiverLien(id): Promise<LienPaiement>`.

- [ ] **Step 1: Extend the failing test**

Add to `api/tests/liens.test.ts`:

```typescript
  describe('PATCH /api/liens/:id/desactiver', () => {
    it('deactivates an active lien, making it unusable afterwards', async () => {
      const created = await agent.post('/api/liens').send({
        montant: 5000,
        montantLibre: false,
        devise: 'XOF',
        description: 'Lien à désactiver',
        modeAuthentification: 'STANDARD',
        usageUnique: true,
        client: { nom: 'Client Désactivé' },
      });

      const patchRes = await agent.patch(`/api/liens/${created.body.id}/desactiver`);
      expect(patchRes.status).toBe(200);
      expect(patchRes.body.statutLien).toBe('desactive');

      const getRes = await request(app).get(`/api/liens/${created.body.id}`);
      expect(getRes.status).toBe(410);
    });

    it('rejects unauthenticated requests', async () => {
      const res = await request(app).patch('/api/liens/some-id/desactiver');
      expect(res.status).toBe(401);
    });
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `PATCH /api/liens/:id/desactiver`

- [ ] **Step 3: Extend `api/src/services/lien.service.ts`**

Append:

```typescript
export async function desactiverLien(id: string) {
  return prisma.lienPaiement.update({ where: { id }, data: { statutLien: 'desactive' } });
}
```

- [ ] **Step 4: Extend `api/src/routes/liens.routes.ts`**

Update the import line:

```typescript
import { createLien, listLiens, getLienById, isLienUsable, desactiverLien } from '../services/lien.service';
```

Append the route:

```typescript
liensRouter.patch(
  '/:id/desactiver',
  requireAuth,
  asyncHandler(async (req: AuthenticatedRequest, res) => {
    const lien = await getLienById(req.params.id);
    if (!lien) {
      res.status(404).json({ error: 'Lien introuvable' });
      return;
    }
    const updated = await desactiverLien(req.params.id);
    await recordAudit(req.admin!.id, 'desactivation_lien', updated.id);
    res.json(updated);
  })
);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 2 new tests pass

- [ ] **Step 6: Commit**

```bash
git add api/src/services/lien.service.ts api/src/routes/liens.routes.ts api/tests/liens.test.ts
git commit -m "feat: add endpoint to deactivate a payment link"
```

---

### Task 8: Audit log service, wired into login/create/deactivate

**Files:**
- Modify: `api/src/services/audit.service.ts` — replace the Task 4 stub with the real implementation
- Test: `api/tests/audit.test.ts`

**Interfaces:**
- Consumes: `prisma`, `createApp`, existing `/api/auth/login`, `/api/liens`, `/api/liens/:id/desactiver` routes (Tasks 4–7), which already call `recordAudit`.
- Produces: `recordAudit(userId, action, cibleId?, detail?): Promise<void>` (real implementation — signature unchanged from the Task 4 stub, so no caller needs to change).

- [ ] **Step 1: Write the failing test**

Create `api/tests/audit.test.ts`:

```typescript
import request from 'supertest';
import { createApp } from '../src/app';
import { prisma } from '../src/db/prisma';
import { hashPassword } from '../src/utils/password';

describe('audit logging', () => {
  const app = createApp();
  const email = `admin-audit-${Date.now()}@example.com`;
  const agent = request.agent(app);

  beforeAll(async () => {
    process.env.JWT_SECRET = 'test-secret';
    await prisma.adminUser.create({
      data: { nom: 'Admin Audit', email, motDePasseHash: await hashPassword('Sup3rSecret!') },
    });
  });

  afterAll(async () => {
    await prisma.adminUser.deleteMany({ where: { email } });
    await prisma.$disconnect();
  });

  it('records a login audit entry', async () => {
    await agent.post('/api/auth/login').send({ email, password: 'Sup3rSecret!' });
    const entries = await prisma.auditLog.findMany({ where: { action: 'login' }, orderBy: { createdAt: 'desc' } });
    expect(entries.length).toBeGreaterThan(0);
  });

  it('records a creation_lien audit entry with the lien id as cible', async () => {
    const res = await agent.post('/api/liens').send({
      montant: 1000,
      montantLibre: false,
      devise: 'XOF',
      description: 'Test audit',
      modeAuthentification: 'STANDARD',
      usageUnique: true,
      client: { nom: 'Client Audit' },
    });
    const entry = await prisma.auditLog.findFirst({
      where: { action: 'creation_lien', cibleId: res.body.id },
    });
    expect(entry).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — the stub `recordAudit` from Task 4 does nothing, so no `AuditLog` rows exist

- [ ] **Step 3: Implement `api/src/services/audit.service.ts`**

Replace the stub with:

```typescript
import { prisma } from '../db/prisma';

export async function recordAudit(
  userId: string,
  action: string,
  cibleId?: string,
  detail?: Record<string, unknown>
) {
  await prisma.auditLog.create({ data: { userId, action, cibleId, detail } });
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 2 new tests pass

- [ ] **Step 5: Commit**

```bash
git add api/src/services/audit.service.ts api/tests/audit.test.ts
git commit -m "feat: persist audit log entries for sensitive admin actions"
```

---

### Task 9: PSP adapter interface and PayDunya implementation

**Files:**
- Create: `api/src/psp/PSPAdapter.ts`
- Create: `api/src/psp/PayDunyaAdapter.ts`
- Test: `api/tests/psp/PayDunyaAdapter.test.ts`

**Interfaces:**
- Consumes: nothing from prior tasks (pure module, no DB).
- Produces: `interface PSPAdapter { createPaymentSession, parseWebhook }`, `class PayDunyaAdapter implements PSPAdapter`, types `CreatePaymentSessionInput`, `CreatePaymentSessionResult`, `WebhookParseResult` — consumed by Task 10's webhook route and `psp/index.ts` factory.

> **Note for Phase 0 cadrage:** the exact field names PayDunya uses in its `checkout-invoice/create` response and webhook payload must be confirmed against PayDunya's live sandbox documentation before going to production — this adapter encodes PayDunya's publicly documented shape (sha512 hash of the master key for webhook verification, `response_code`/`token` on invoice creation). Only `PayDunyaAdapter.ts` would need to change if any field name differs; the `PSPAdapter` interface and every caller stay the same.

- [ ] **Step 1: Write the failing test**

Create `api/tests/psp/PayDunyaAdapter.test.ts`:

```typescript
import { createHash } from 'crypto';
import { PayDunyaAdapter } from '../../src/psp/PayDunyaAdapter';

describe('PayDunyaAdapter', () => {
  const config = { masterKey: 'master', privateKey: 'priv', publicKey: 'pub', token: 'tok', mode: 'test' as const };
  const adapter = new PayDunyaAdapter(config);

  afterEach(() => {
    jest.restoreAllMocks();
  });

  it('creates a payment session and builds the checkout URL', async () => {
    jest.spyOn(global, 'fetch').mockResolvedValue({
      json: async () => ({ response_code: '00', token: 'abc123' }),
    } as Response);

    const result = await adapter.createPaymentSession({
      reference: 'HYBC-2026-0001',
      montant: 50000,
      devise: 'XOF',
      description: 'Frais de dossier',
      modeAuthentification: 'STANDARD',
      clientNom: 'Jean Client',
      returnUrl: 'https://example.com/return',
      cancelUrl: 'https://example.com/cancel',
    });

    expect(result.checkoutUrl).toBe('https://paydunya.com/checkout/invoice/abc123');
    expect(result.pspTransactionId).toBe('abc123');
  });

  it('throws when PayDunya rejects the request', async () => {
    jest.spyOn(global, 'fetch').mockResolvedValue({
      json: async () => ({ response_code: '01', response_text: 'Solde insuffisant' }),
    } as Response);

    await expect(
      adapter.createPaymentSession({
        reference: 'HYBC-2026-0002',
        montant: 50000,
        devise: 'XOF',
        description: 'Frais',
        modeAuthentification: 'STANDARD',
        clientNom: 'Jean',
        returnUrl: 'https://example.com/return',
        cancelUrl: 'https://example.com/cancel',
      })
    ).rejects.toThrow('PayDunya a refusé la création de session');
  });

  it('parses a valid webhook payload', () => {
    const hash = createHash('sha512').update('master').digest('hex');
    const result = adapter.parseWebhook({
      data: {
        hash,
        status: 'completed',
        invoice: { token: 'abc123' },
        custom_data: { reference: 'HYBC-2026-0001' },
      },
    });
    expect(result.statut).toBe('reussi');
    expect(result.pspTransactionId).toBe('abc123');
    expect(result.reference).toBe('HYBC-2026-0001');
  });

  it('rejects a webhook with an invalid signature', () => {
    expect(() => adapter.parseWebhook({ data: { hash: 'wrong' } })).toThrow(
      'Signature de webhook PayDunya invalide'
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — cannot find module `../../src/psp/PayDunyaAdapter`

- [ ] **Step 3: Implement `api/src/psp/PSPAdapter.ts`**

```typescript
export interface CreatePaymentSessionInput {
  reference: string;
  montant: number;
  devise: string;
  description: string;
  modeAuthentification: 'STANDARD' | '3DS';
  clientNom: string;
  clientEmail?: string;
  returnUrl: string;
  cancelUrl: string;
}

export interface CreatePaymentSessionResult {
  checkoutUrl: string;
  pspTransactionId: string;
}

export interface WebhookParseResult {
  pspTransactionId: string;
  reference: string;
  statut: 'reussi' | 'echoue';
  reseauCarte?: string;
  messageErreur?: string;
  rawPayload: unknown;
}

export interface PSPAdapter {
  createPaymentSession(input: CreatePaymentSessionInput): Promise<CreatePaymentSessionResult>;
  parseWebhook(rawBody: unknown, headers?: Record<string, string | string[] | undefined>): WebhookParseResult;
}
```

- [ ] **Step 4: Implement `api/src/psp/PayDunyaAdapter.ts`**

```typescript
import { createHash } from 'crypto';
import {
  PSPAdapter,
  CreatePaymentSessionInput,
  CreatePaymentSessionResult,
  WebhookParseResult,
} from './PSPAdapter';

export interface PayDunyaConfig {
  masterKey: string;
  privateKey: string;
  publicKey: string;
  token: string;
  mode: 'test' | 'live';
}

export class PayDunyaAdapter implements PSPAdapter {
  constructor(private config: PayDunyaConfig) {}

  private get baseUrl(): string {
    return this.config.mode === 'live'
      ? 'https://app.paydunya.com/api/v1'
      : 'https://app.paydunya.com/sandbox-api/v1';
  }

  async createPaymentSession(input: CreatePaymentSessionInput): Promise<CreatePaymentSessionResult> {
    const response = await fetch(`${this.baseUrl}/checkout-invoice/create`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'PAYDUNYA-MASTER-KEY': this.config.masterKey,
        'PAYDUNYA-PRIVATE-KEY': this.config.privateKey,
        'PAYDUNYA-PUBLIC-KEY': this.config.publicKey,
        'PAYDUNYA-TOKEN': this.config.token,
      },
      body: JSON.stringify({
        invoice: { total_amount: input.montant, description: input.description },
        store: { name: 'HY BUSINESS AND CO' },
        custom_data: { reference: input.reference, modeAuthentification: input.modeAuthentification },
        actions: { return_url: input.returnUrl, cancel_url: input.cancelUrl },
      }),
    });

    const data = (await response.json()) as { response_code: string; token?: string; response_text?: string };
    if (data.response_code !== '00' || !data.token) {
      throw new Error(`PayDunya a refusé la création de session: ${data.response_text ?? 'erreur inconnue'}`);
    }
    return {
      checkoutUrl: `https://paydunya.com/checkout/invoice/${data.token}`,
      pspTransactionId: data.token,
    };
  }

  parseWebhook(rawBody: any): WebhookParseResult {
    const expectedHash = createHash('sha512').update(this.config.masterKey).digest('hex');
    if (rawBody?.data?.hash !== expectedHash) {
      throw new Error('Signature de webhook PayDunya invalide');
    }
    const status = rawBody.data?.status;
    return {
      pspTransactionId: rawBody.data?.invoice?.token,
      reference: rawBody.data?.custom_data?.reference,
      statut: status === 'completed' ? 'reussi' : 'echoue',
      reseauCarte: rawBody.data?.actions?.card_type,
      messageErreur: status === 'completed' ? undefined : rawBody.data?.response_text,
      rawPayload: rawBody,
    };
  }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 4 new tests pass

- [ ] **Step 6: Commit**

```bash
git add api/src/psp/PSPAdapter.ts api/src/psp/PayDunyaAdapter.ts api/tests/psp
git commit -m "feat: add PSPAdapter interface and PayDunya implementation"
```

---

### Task 10: Webhook endpoint — `POST /api/webhooks/psp`

**Files:**
- Create: `api/src/psp/index.ts`
- Create: `api/src/services/transaction.service.ts`
- Create: `api/src/routes/webhooks.routes.ts`
- Modify: `api/src/app.ts` — mount `webhooksRouter`
- Test: `api/tests/webhooks.test.ts`

**Interfaces:**
- Consumes: `PSPAdapter`, `WebhookParseResult` (Task 9), `prisma`.
- Produces: `getPspAdapter(): PSPAdapter`, `applyWebhookResult(result: WebhookParseResult): Promise<Transaction>` — `applyWebhookResult` and `getPspAdapter` are consumed by Task 12 (email dispatch) and Task 11 reuses `transaction.service.ts`.

- [ ] **Step 1: Write the failing test**

Create `api/tests/webhooks.test.ts`:

```typescript
import request from 'supertest';
import { createHash } from 'crypto';
import { createApp } from '../src/app';
import { prisma } from '../src/db/prisma';

describe('POST /api/webhooks/psp', () => {
  const app = createApp();
  let lienId: string;
  const reference = `HYBC-TEST-${Date.now()}`;

  beforeAll(async () => {
    process.env.PAYDUNYA_MASTER_KEY = 'master';
    process.env.PAYDUNYA_MODE = 'test';
    const client = await prisma.client.create({ data: { nom: 'Client Webhook' } });
    const lien = await prisma.lienPaiement.create({
      data: {
        reference,
        urlToken: `token-${Date.now()}`,
        montant: 25000,
        devise: 'XOF',
        description: 'Test webhook',
        modeAuthentification: 'STANDARD',
        usageUnique: true,
        createdBy: 'test',
        clientId: client.id,
      },
    });
    lienId = lien.id;
  });

  afterAll(async () => {
    await prisma.transaction.deleteMany({ where: { lienId } });
    await prisma.lienPaiement.deleteMany({ where: { id: lienId } });
    await prisma.$disconnect();
  });

  function webhookPayload(pspTransactionId: string) {
    const hash = createHash('sha512').update('master').digest('hex');
    return {
      data: {
        hash,
        status: 'completed',
        invoice: { token: pspTransactionId },
        custom_data: { reference },
      },
    };
  }

  it('marks the transaction as successful and expires a single-use lien', async () => {
    const res = await request(app).post('/api/webhooks/psp').send(webhookPayload('psp-tx-1'));
    expect(res.status).toBe(200);

    const updatedLien = await prisma.lienPaiement.findUnique({ where: { id: lienId } });
    expect(updatedLien?.statutLien).toBe('expire');

    const transactions = await prisma.transaction.findMany({ where: { lienId } });
    expect(transactions).toHaveLength(1);
    expect(transactions[0].statut).toBe('reussi');
  });

  it('is idempotent when the same webhook is replayed', async () => {
    await request(app).post('/api/webhooks/psp').send(webhookPayload('psp-tx-1'));
    const transactions = await prisma.transaction.findMany({ where: { lienId } });
    expect(transactions).toHaveLength(1);
  });

  it('rejects a webhook with an invalid signature', async () => {
    const res = await request(app).post('/api/webhooks/psp').send({ data: { hash: 'invalid' } });
    expect(res.status).toBe(400);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `/api/webhooks/psp`

- [ ] **Step 3: Implement `api/src/psp/index.ts`**

```typescript
import { PayDunyaAdapter } from './PayDunyaAdapter';
import { PSPAdapter } from './PSPAdapter';

export function getPspAdapter(): PSPAdapter {
  return new PayDunyaAdapter({
    masterKey: process.env.PAYDUNYA_MASTER_KEY ?? '',
    privateKey: process.env.PAYDUNYA_PRIVATE_KEY ?? '',
    publicKey: process.env.PAYDUNYA_PUBLIC_KEY ?? '',
    token: process.env.PAYDUNYA_TOKEN ?? '',
    mode: (process.env.PAYDUNYA_MODE as 'test' | 'live') ?? 'test',
  });
}
```

- [ ] **Step 4: Implement `api/src/services/transaction.service.ts`**

```typescript
import { prisma } from '../db/prisma';
import { WebhookParseResult } from '../psp/PSPAdapter';

export async function applyWebhookResult(result: WebhookParseResult) {
  const lien = await prisma.lienPaiement.findUnique({ where: { reference: result.reference } });
  if (!lien) {
    throw new Error(`Aucun lien trouvé pour la référence ${result.reference}`);
  }

  const transaction = await prisma.transaction.upsert({
    where: { idTransactionPSP: result.pspTransactionId },
    update: {
      statut: result.statut,
      reseauCarte: result.reseauCarte,
      messageErreur: result.messageErreur,
      webhookPayload: result.rawPayload as any,
    },
    create: {
      lienId: lien.id,
      idTransactionPSP: result.pspTransactionId,
      pspNom: 'paydunya',
      montant: lien.montant ?? 0,
      devise: lien.devise,
      statut: result.statut,
      reseauCarte: result.reseauCarte,
      messageErreur: result.messageErreur,
      webhookPayload: result.rawPayload as any,
    },
  });

  if (result.statut === 'reussi' && lien.usageUnique) {
    await prisma.lienPaiement.update({ where: { id: lien.id }, data: { statutLien: 'expire' } });
  }

  return transaction;
}
```

- [ ] **Step 5: Implement `api/src/routes/webhooks.routes.ts`**

```typescript
import { Router } from 'express';
import { getPspAdapter } from '../psp';
import { applyWebhookResult } from '../services/transaction.service';
import { asyncHandler } from '../utils/asyncHandler';

export const webhooksRouter = Router();

webhooksRouter.post(
  '/psp',
  asyncHandler(async (req, res) => {
    const adapter = getPspAdapter();
    let parsed;
    try {
      parsed = adapter.parseWebhook(req.body, req.headers);
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
      return;
    }
    try {
      await applyWebhookResult(parsed);
      res.status(200).json({ received: true });
    } catch (err) {
      res.status(500).json({ error: (err as Error).message });
    }
  })
);
```

- [ ] **Step 6: Mount the router in `api/src/app.ts`**

Add import: `import { webhooksRouter } from './routes/webhooks.routes';`
Add after the liens mount: `app.use('/api/webhooks', webhooksRouter);`

- [ ] **Step 7: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 3 new tests pass

- [ ] **Step 8: Commit**

```bash
git add api/src/psp/index.ts api/src/services/transaction.service.ts api/src/routes/webhooks.routes.ts api/src/app.ts api/tests/webhooks.test.ts
git commit -m "feat: process PayDunya webhooks into transaction updates"
```

---

### Task 11: List and fetch transactions — `GET /api/transactions`, `GET /api/transactions/:id`

**Files:**
- Modify: `api/src/services/transaction.service.ts` — add `listTransactions`, `getTransactionById`
- Create: `api/src/routes/transactions.routes.ts`
- Modify: `api/src/app.ts` — mount `transactionsRouter`
- Test: `api/tests/transactions.test.ts`

**Interfaces:**
- Consumes: `applyWebhookResult` (Task 10, used in test setup), `requireAuth`, `asyncHandler`.
- Produces: `listTransactions(filters): Promise<TransactionWithLien[]>`, `getTransactionById(id): Promise<TransactionWithLien | null>`.

- [ ] **Step 1: Write the failing test**

Create `api/tests/transactions.test.ts`:

```typescript
import request from 'supertest';
import { createHash } from 'crypto';
import { createApp } from '../src/app';
import { prisma } from '../src/db/prisma';
import { hashPassword } from '../src/utils/password';

describe('transactions routes', () => {
  const app = createApp();
  const email = `admin-tx-${Date.now()}@example.com`;
  const agent = request.agent(app);
  let lienId: string;
  let transactionId: string;
  const reference = `HYBC-TX-${Date.now()}`;

  beforeAll(async () => {
    process.env.JWT_SECRET = 'test-secret';
    process.env.PAYDUNYA_MASTER_KEY = 'master';
    await prisma.adminUser.create({
      data: { nom: 'Admin TX', email, motDePasseHash: await hashPassword('Sup3rSecret!') },
    });
    await agent.post('/api/auth/login').send({ email, password: 'Sup3rSecret!' });

    const client = await prisma.client.create({ data: { nom: 'Client TX' } });
    const lien = await prisma.lienPaiement.create({
      data: {
        reference,
        urlToken: `token-tx-${Date.now()}`,
        montant: 15000,
        devise: 'XOF',
        description: 'Test transactions',
        modeAuthentification: 'STANDARD',
        usageUnique: true,
        createdBy: 'test',
        clientId: client.id,
      },
    });
    lienId = lien.id;

    const hash = createHash('sha512').update('master').digest('hex');
    const webhookRes = await request(app).post('/api/webhooks/psp').send({
      data: { hash, status: 'completed', invoice: { token: 'psp-tx-list' }, custom_data: { reference } },
    });
    expect(webhookRes.status).toBe(200);
    const transaction = await prisma.transaction.findFirst({ where: { lienId } });
    transactionId = transaction!.id;
  });

  afterAll(async () => {
    await prisma.transaction.deleteMany({ where: { lienId } });
    await prisma.lienPaiement.deleteMany({ where: { id: lienId } });
    await prisma.adminUser.deleteMany({ where: { email } });
    await prisma.$disconnect();
  });

  it('rejects unauthenticated list requests', async () => {
    const res = await request(app).get('/api/transactions');
    expect(res.status).toBe(401);
  });

  it('lists transactions including their lien and client', async () => {
    const res = await agent.get('/api/transactions');
    expect(res.status).toBe(200);
    const found = res.body.find((t: { id: string }) => t.id === transactionId);
    expect(found.lien.client.nom).toBe('Client TX');
  });

  it('returns transaction detail by id', async () => {
    const res = await agent.get(`/api/transactions/${transactionId}`);
    expect(res.status).toBe(200);
    expect(res.body.id).toBe(transactionId);
  });

  it('returns 404 for an unknown transaction id', async () => {
    const res = await agent.get('/api/transactions/unknown-id');
    expect(res.status).toBe(404);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — 404 on `/api/transactions`

- [ ] **Step 3: Extend `api/src/services/transaction.service.ts`**

Append:

```typescript
export interface ListTransactionsFilters {
  statut?: string;
  reseauCarte?: string;
}

export async function listTransactions(filters: ListTransactionsFilters = {}) {
  return prisma.transaction.findMany({
    where: {
      statut: filters.statut,
      reseauCarte: filters.reseauCarte,
    },
    include: { lien: { include: { client: true } } },
    orderBy: { dateCreation: 'desc' },
  });
}

export async function getTransactionById(id: string) {
  return prisma.transaction.findUnique({
    where: { id },
    include: { lien: { include: { client: true } } },
  });
}
```

- [ ] **Step 4: Implement `api/src/routes/transactions.routes.ts`**

```typescript
import { Router } from 'express';
import { requireAuth } from '../middleware/auth';
import { asyncHandler } from '../utils/asyncHandler';
import { listTransactions, getTransactionById } from '../services/transaction.service';

export const transactionsRouter = Router();

transactionsRouter.get(
  '/',
  requireAuth,
  asyncHandler(async (req, res) => {
    const { statut, reseauCarte } = req.query;
    const transactions = await listTransactions({
      statut: typeof statut === 'string' ? statut : undefined,
      reseauCarte: typeof reseauCarte === 'string' ? reseauCarte : undefined,
    });
    res.json(transactions);
  })
);

transactionsRouter.get(
  '/:id',
  requireAuth,
  asyncHandler(async (req, res) => {
    const transaction = await getTransactionById(req.params.id);
    if (!transaction) {
      res.status(404).json({ error: 'Transaction introuvable' });
      return;
    }
    res.json(transaction);
  })
);
```

- [ ] **Step 5: Mount the router in `api/src/app.ts`**

Add import: `import { transactionsRouter } from './routes/transactions.routes';`
Add after the webhooks mount: `app.use('/api/transactions', transactionsRouter);`

- [ ] **Step 6: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 4 new tests pass

- [ ] **Step 7: Commit**

```bash
git add api/src/services/transaction.service.ts api/src/routes/transactions.routes.ts api/src/app.ts api/tests/transactions.test.ts
git commit -m "feat: add list and detail endpoints for transactions"
```

---

### Task 12: Transactional emails via Resend, wired into the webhook handler

**Files:**
- Create: `api/src/services/email.service.ts`
- Modify: `api/src/routes/webhooks.routes.ts` — send emails after a webhook is applied
- Modify: `api/tests/webhooks.test.ts` — mock the email service so the existing integration tests don't hit the network
- Test: `api/tests/email.test.ts`

**Interfaces:**
- Consumes: `applyWebhookResult` (Task 10), `prisma`.
- Produces: `sendAdminPaymentNotification(params): Promise<void>`, `sendClientReceipt(params): Promise<void>`.

- [ ] **Step 1: Write the failing test**

Create `api/tests/email.test.ts`:

```typescript
jest.mock('resend', () => ({
  Resend: jest.fn().mockImplementation(() => ({
    emails: { send: jest.fn().mockResolvedValue({ id: 'email-1' }) },
  })),
}));

import { sendAdminPaymentNotification, sendClientReceipt } from '../src/services/email.service';

describe('email service', () => {
  beforeAll(() => {
    process.env.RESEND_API_KEY = 'test-key';
    process.env.ADMIN_NOTIFICATION_EMAIL = 'ops@example.com';
    process.env.EMAIL_FROM = 'paiements@hybusinessandco.com';
  });

  it('sends an admin notification without throwing', async () => {
    await expect(
      sendAdminPaymentNotification({
        reference: 'HYBC-2026-0001',
        montant: 50000,
        devise: 'XOF',
        statut: 'reussi',
        clientNom: 'Jean',
      })
    ).resolves.not.toThrow();
  });

  it('sends a client receipt without throwing', async () => {
    await expect(
      sendClientReceipt({ to: 'jean@example.com', reference: 'HYBC-2026-0001', montant: 50000, devise: 'XOF' })
    ).resolves.not.toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — cannot find module `../src/services/email.service`

- [ ] **Step 3: Implement `api/src/services/email.service.ts`**

```typescript
import { Resend } from 'resend';

let resendClient: Resend | null = null;

function getClient(): Resend {
  if (!resendClient) {
    resendClient = new Resend(process.env.RESEND_API_KEY ?? '');
  }
  return resendClient;
}

export async function sendAdminPaymentNotification(params: {
  reference: string;
  montant: number;
  devise: string;
  statut: 'reussi' | 'echoue';
  clientNom: string;
}) {
  const to = process.env.ADMIN_NOTIFICATION_EMAIL;
  if (!to) return;
  await getClient().emails.send({
    from: process.env.EMAIL_FROM ?? 'paiements@hybusinessandco.com',
    to,
    subject: `Paiement ${params.statut === 'reussi' ? 'réussi' : 'échoué'} — ${params.reference}`,
    text: `Le paiement de ${params.montant} ${params.devise} pour ${params.clientNom} (réf. ${params.reference}) est ${params.statut}.`,
  });
}

export async function sendClientReceipt(params: {
  to: string;
  reference: string;
  montant: number;
  devise: string;
}) {
  await getClient().emails.send({
    from: process.env.EMAIL_FROM ?? 'paiements@hybusinessandco.com',
    to: params.to,
    subject: `Reçu de paiement — ${params.reference}`,
    text: `Nous confirmons la réception de votre paiement de ${params.montant} ${params.devise} (réf. ${params.reference}). Merci.`,
  });
}
```

- [ ] **Step 4: Wire emails into `api/src/routes/webhooks.routes.ts`**

Replace the file's imports and handler body with:

```typescript
import { Router } from 'express';
import { getPspAdapter } from '../psp';
import { applyWebhookResult } from '../services/transaction.service';
import { sendAdminPaymentNotification, sendClientReceipt } from '../services/email.service';
import { prisma } from '../db/prisma';
import { asyncHandler } from '../utils/asyncHandler';

export const webhooksRouter = Router();

webhooksRouter.post(
  '/psp',
  asyncHandler(async (req, res) => {
    const adapter = getPspAdapter();
    let parsed;
    try {
      parsed = adapter.parseWebhook(req.body, req.headers);
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
      return;
    }
    try {
      const transaction = await applyWebhookResult(parsed);
      const lien = await prisma.lienPaiement.findUnique({
        where: { id: transaction.lienId },
        include: { client: true },
      });
      if (lien) {
        await sendAdminPaymentNotification({
          reference: lien.reference,
          montant: Number(transaction.montant),
          devise: transaction.devise,
          statut: transaction.statut as 'reussi' | 'echoue',
          clientNom: lien.client.nom,
        });
        if (transaction.statut === 'reussi' && lien.client.email) {
          await sendClientReceipt({
            to: lien.client.email,
            reference: lien.reference,
            montant: Number(transaction.montant),
            devise: transaction.devise,
          });
        }
      }
      res.status(200).json({ received: true });
    } catch (err) {
      res.status(500).json({ error: (err as Error).message });
    }
  })
);
```

- [ ] **Step 5: Mock the email service in the existing webhook integration test**

At the very top of `api/tests/webhooks.test.ts`, before the other imports, add:

```typescript
jest.mock('../src/services/email.service', () => ({
  sendAdminPaymentNotification: jest.fn().mockResolvedValue(undefined),
  sendClientReceipt: jest.fn().mockResolvedValue(undefined),
}));
```

- [ ] **Step 6: Run tests to verify everything passes**

Run: `npm test`
Expected: PASS — all prior tests (including the still-green `webhooks.test.ts`) plus 2 new email tests pass

- [ ] **Step 7: Commit**

```bash
git add api/src/services/email.service.ts api/src/routes/webhooks.routes.ts api/tests/webhooks.test.ts api/tests/email.test.ts
git commit -m "feat: send admin and client emails on webhook processing"
```

---

### Task 13: Centralized 404 and error handling

**Files:**
- Create: `api/src/middleware/errorHandler.ts`
- Modify: `api/src/app.ts` — register `notFoundHandler` and `errorHandler` last
- Test: `api/tests/errorHandler.test.ts`

**Interfaces:**
- Consumes: nothing (pure Express middleware).
- Produces: `notFoundHandler`, `errorHandler` — mounted once, at the end of `app.ts`; every `asyncHandler`-wrapped route already forwards unexpected errors here via `next(err)`.

- [ ] **Step 1: Write the failing test**

Create `api/tests/errorHandler.test.ts`:

```typescript
import { Request, Response } from 'express';
import { notFoundHandler, errorHandler } from '../src/middleware/errorHandler';

function mockRes() {
  const res = {} as Response;
  res.status = jest.fn().mockReturnValue(res);
  res.json = jest.fn().mockReturnValue(res);
  return res;
}

describe('notFoundHandler', () => {
  it('responds with 404 and a French error message', () => {
    const res = mockRes();
    notFoundHandler({} as Request, res);
    expect(res.status).toHaveBeenCalledWith(404);
    expect(res.json).toHaveBeenCalledWith({ error: 'Ressource introuvable' });
  });
});

describe('errorHandler', () => {
  it('responds with 500 and logs the error', () => {
    const res = mockRes();
    const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});
    errorHandler(new Error('boom'), {} as Request, res, jest.fn());
    expect(res.status).toHaveBeenCalledWith(500);
    expect(res.json).toHaveBeenCalledWith({ error: 'Erreur interne du serveur' });
    consoleSpy.mockRestore();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm test`
Expected: FAIL — cannot find module `../src/middleware/errorHandler`

- [ ] **Step 3: Implement `api/src/middleware/errorHandler.ts`**

```typescript
import { Request, Response, NextFunction } from 'express';

export function notFoundHandler(_req: Request, res: Response) {
  res.status(404).json({ error: 'Ressource introuvable' });
}

export function errorHandler(err: unknown, _req: Request, res: Response, _next: NextFunction) {
  console.error(err);
  res.status(500).json({ error: 'Erreur interne du serveur' });
}
```

- [ ] **Step 4: Register the handlers last in `api/src/app.ts`**

Add the import near the top:

```typescript
import { notFoundHandler, errorHandler } from './middleware/errorHandler';
```

Add at the very end of `createApp`, after all router mounts and before `return app;`:

```typescript
  app.use(notFoundHandler);
  app.use(errorHandler);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm test`
Expected: PASS — all prior tests plus 2 new tests pass

- [ ] **Step 6: Commit**

```bash
git add api/src/middleware/errorHandler.ts api/src/app.ts api/tests/errorHandler.test.ts
git commit -m "feat: add centralized 404 and error handling middleware"
```

---

### Task 14: Admin seed script and README

**Files:**
- Create: `api/prisma/seed.ts`
- Create: `api/README.md`

**Interfaces:**
- Consumes: `prisma`, `hashPassword`.
- Produces: a runnable `npm run seed` script that creates the first `AdminUser` — this is the account the Phase 2 frontend plan's dashboard login will use.

- [ ] **Step 1: Implement `api/prisma/seed.ts`**

```typescript
import { prisma } from '../src/db/prisma';
import { hashPassword } from '../src/utils/password';

async function main() {
  const email = process.env.SEED_ADMIN_EMAIL ?? 'admin@hybusinessandco.com';
  const password = process.env.SEED_ADMIN_PASSWORD ?? 'ChangeMe123!';
  const existing = await prisma.adminUser.findUnique({ where: { email } });
  if (existing) {
    console.log(`Admin ${email} existe déjà.`);
    return;
  }
  await prisma.adminUser.create({
    data: { nom: 'Administrateur', email, motDePasseHash: await hashPassword(password) },
  });
  console.log(`Admin créé : ${email}`);
}

main()
  .catch((err) => {
    console.error(err);
    process.exitCode = 1;
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

- [ ] **Step 2: Run the seed script and confirm it works**

Run: `cd api && npm run seed`
Expected: Console prints `Admin créé : admin@hybusinessandco.com` (or the email from `SEED_ADMIN_EMAIL` if set). Running it again prints `Admin ... existe déjà.` and does not error.

- [ ] **Step 3: Write `api/README.md`**

```markdown
# API — Lien de Paiement & Dashboard

## Prérequis
- Node.js 20+
- PostgreSQL 16 (local ou via Docker, voir ci-dessous)

## Démarrage

\`\`\`bash
docker run --name lien-paiement-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=lien_paiement -p 5432:5432 -d postgres:16
cp .env.example .env   # puis renseigner les clés PayDunya/Resend
npm install
npm run prisma:migrate
npm run seed
npm run dev
\`\`\`

L'API écoute par défaut sur http://localhost:4000.

## Tests

\`\`\`bash
npm test
\`\`\`

Les tests utilisent la base PostgreSQL définie par `DATABASE_URL` — assurez-vous qu'elle est démarrée et migrée
avant de lancer `npm test`.

## Variables d'environnement

Voir `.env.example`. Les clés `PAYDUNYA_*` et `RESEND_API_KEY` sont fournies lors de la Phase 0 (cadrage PSP).
```

- [ ] **Step 4: Run the full test suite one last time to confirm everything is green**

Run: `npm test`
Expected: PASS — all tests across every task pass together

- [ ] **Step 5: Commit**

```bash
git add api/prisma/seed.ts api/README.md
git commit -m "docs: add API README and admin seed script"
```
