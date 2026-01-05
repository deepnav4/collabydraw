# Vercel Deployment Fix

## Issue
Vercel can't detect Next.js in the monorepo structure.

## Solution Steps

### 1. Configure Vercel Project Settings

In your Vercel project dashboard, set these values:

**Framework Preset:** `Next.js`

**Root Directory:** `apps/collabydraw`

**Build & Development Settings:**
- **Build Command:** `pnpm run build`
- **Output Directory:** `.next` (leave as default)
- **Install Command:** `pnpm install --filter=collabydraw...`

### 2. Environment Variables

Add these in Vercel Dashboard → Settings → Environment Variables:

```env
# Database
DATABASE_URL=your_database_url_here

# NextAuth
NEXTAUTH_URL=https://your-domain.vercel.app
NEXTAUTH_SECRET=generate_with_openssl_rand_base64_32

# JWT (must match websocket)
JWT_SECRET=same_as_websocket_server

# WebSocket
NEXT_PUBLIC_WS_URL=wss://your-websocket-url.railway.app

# Base URL
NEXT_PUBLIC_BASE_URL=https://your-domain.vercel.app
```

### 3. Deploy via CLI

```bash
cd apps/collabydraw

# Login to Vercel
vercel login

# Deploy
vercel

# For production
vercel --prod
```

### 4. Alternative: Use Vercel Dashboard

1. Go to https://vercel.com/new
2. Import your GitHub repository
3. **Important Settings:**
   - Framework Preset: **Next.js**
   - Root Directory: **apps/collabydraw**
   - Build Command: Leave default or use: `pnpm run build`
   - Install Command: `pnpm install`
4. Add all environment variables
5. Click Deploy

## Troubleshooting

### "No Next.js version detected"

**Fix 1: Update package.json**
Make sure `next` is in dependencies (already done in your project).

**Fix 2: Correct Root Directory**
- In Vercel Dashboard → Settings → General
- Set Root Directory to: `apps/collabydraw`
- Save and redeploy

**Fix 3: Use Build Override**
In Vercel Dashboard → Settings → General → Build & Development Settings:
- Build Command: `cd ../.. && pnpm install && cd apps/collabydraw && pnpm build`

### "pnpm version mismatch"

Add to Vercel environment variables:
```
PNPM_VERSION=9.0.0
```

### "Workspace dependencies not found"

This happens if Vercel doesn't install from root. Use this build command:
```bash
pnpm install --filter=collabydraw... && pnpm run build
```

## Recommended: Railway for WebSocket

1. Deploy WebSocket on Railway first
2. Get the public URL (e.g., `wss://collabydraw-ws.up.railway.app`)
3. Add it as `NEXT_PUBLIC_WS_URL` in Vercel
4. Redeploy Vercel

## Quick Success Path

```bash
# 1. Push to GitHub
git add .
git commit -m "Prepare for Vercel deployment"
git push origin main

# 2. Deploy WebSocket on Railway
# - Go to railway.app
# - Deploy apps/ws directory
# - Add DATABASE_URL and JWT_SECRET
# - Copy the public URL

# 3. Deploy Frontend on Vercel
# - Go to vercel.com/new
# - Import from GitHub
# - Root Directory: apps/collabydraw
# - Add all environment variables (use Railway WS URL)
# - Deploy
```

Done! 🎉
