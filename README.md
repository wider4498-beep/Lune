# Lune
A role play chat app
rp-monorepo/
├─ backend/
│  ├─ package.json
│  ├─ tsconfig.json
│  ├─ prisma/
│  │  └─ schema.prisma
│  ├─ .env.example
│  └─ src/
│     ├─ index.ts
│     ├─ prismaClient.ts
│     ├─ auth.ts
│     ├─ socketHandlers.ts
│     └─ types.ts
└─ web-client/
   ├─ package.json
   ├─ tsconfig.json
   └─ src/
      ├─ main.tsx
      ├─ App.tsx
      └─ components/
         └─ Chat.tsx
       {
  "name": "rp-backend",
  "version": "1.0.0",
  "main": "dist/index.js",
  "license": "MIT",
  "scripts": {
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "build": "tsc -p .",
    "start": "node dist/index.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev --name init --preview-feature",
    "prisma:studio": "prisma studio"
  },
  "dependencies": {
    "@prisma/client": "^4.16.0",
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "helmet": "^7.0.0",
    "jsonwebtoken": "^9.0.0",
    "socket.io": "^4.7.2"
  },
  "devDependencies": {
    "@types/bcrypt": "^5.0.0",
    "@types/cors": "^2.8.12",
    "@types/express": "^4.17.17",
    "@types/jsonwebtoken": "^9.0.1",
    "@types/node": "^20.3.1",
    "prisma": "^4.16.0",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.1.3"
  }
}
{
  "compilerOptions": {
    "target": "es2020",
    "module": "commonjs",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model User {
  id         String    @id @default(uuid())
  email      String    @unique
  password   String
  display    String
  createdAt  DateTime  @default(now())
  characters Character[]
  messages   Message[]
  reports    Report[]  @relation("reports_by_user")
}

model Character {
  id        String   @id @default(uuid())
  user      User     @relation(fields: [userId], references: [id])
  userId    String
  name      String
  avatarUrl String?
  isPublic  Boolean  @default(true)
  createdAt DateTime @default(now())
  messages  Message[] 
}

model Room {
  id        String    @id @default(uuid())
  slug      String    @unique
  name      String
  createdAt DateTime  @default(now())
  messages  Message[]
}

model Message {
  id          String   @id @default(uuid())
  room        Room     @relation(fields: [roomId], references: [id])
  roomId      String
  sender      User     @relation(fields: [senderId], references: [id])
  senderId    String
  character   Character? @relation(fields: [characterId], references: [id])
  characterId String?
  body        String
  createdAt   DateTime @default(now())
}

model Report {
  id         String   @id @default(uuid())
  reporter   User     @relation("reports_by_user", fields: [reporterId], references: [id])
  reporterId String
  targetType String
  targetId   String
  reason     String
  createdAt  DateTime @default(now())
  status     String   @default("open")
}
import { Socket } from "socket.io";

export type TSocket = Socket & {
  user?: { id: string; email: string; display: string };
};
import express from "express";
import prisma from "./prismaClient";
import bcrypt from "bcrypt";
import jwt from "jsonwebtoken";

const router = express.Router();
const JWT_SECRET = process.env.JWT_SECRET || "dev-secret";

type RegisterBody = { email: string; password: string; display?: string };
type LoginBody = { email: string; password: string };

router.post("/register", async (req, res) => {
  const body = req.body as RegisterBody;
  if (!body.email || !body.password) return res.status(400).json({ error: "email & password required" });

  const existing = await prisma.user.findUnique({ where: { email: body.email } });
  if (existing) return res.status(409).json({ error: "email already in use" });

  const hashed = await bcrypt.hash(body.password, 10);
  const user = await prisma.user.create({
    data: { email: body.email.toLowerCase(), password: hashed, display: body.display || body.email.split("@")[0] }
  });

  const token = jwt.sign({ sub: user.id, email: user.email, display: user.display }, JWT_SECRET, { expiresIn: "7d" });
  res.json({ token, user: { id: user.id, email: user.email, display: user.display } });
});

router.post("/login", async (req, res) => {
  const body = req.body as LoginBody;
  if (!body.email || !body.password) return res.status(400).json({ error: "email & password required" });

  const user = await prisma.user.findUnique({ where: { email: body.email.toLowerCase() }, include: { characters: true } });
  if (!user) return res.status(401).json({ error: "invalid credentials" });

  const ok = await bcrypt.compare(body.password, user.password);
  if (!ok) return res.status(401).json({ error: "invalid credentials" });

  const token = jwt.sign({ sub: user.id, email: user.email, display: user.display }, JWT_SECRET, { expiresIn: "7d" });
  res.json({ token, user: { id: user.id, email: user.email, display: user.display, characters: user.characters } });
});

export default router;
