# 🚀 CollabyDraw Deployment Guide

Complete guide to deploy CollabyDraw in production.

## 📋 Table of Contents
1. [Prerequisites](#prerequisites)
2. [Environment Variables](#environment-variables)
3. [Database Setup](#database-setup)
4. [Deployment Options](#deployment-options)
5. [Post-Deployment](#post-deployment)
6. [Monitoring & Maintenance](#monitoring--maintenance)

---

## Prerequisites

- Node.js 18+ installed
- PostgreSQL database
- Domain name (for production)
- SSL certificates (Let's Encrypt recommended)
- Git repository access

---

## Environment Variables

### 1. Create `.env` files

#### **Root `.env`** (for workspace)
```env
DATABASE_URL="postgresql://username:password@host:5432/collabydraw"
```

#### **`apps/collabydraw/.env.local`** (Next.js Frontend)
```env
# Database
DATABASE_URL="postgresql://username:password@host:5432/collabydraw"

# NextAuth Configuration
NEXTAUTH_URL="https://yourdomain.com"
NEXTAUTH_SECRET="generate-with-openssl-rand-base64-32"

# JWT Secret (must match websocket server)
JWT_SECRET="generate-with-openssl-rand-base64-32"

# WebSocket Server URL
NEXT_PUBLIC_WS_URL="wss://ws.yourdomain.com"

# Base URL for sharing
NEXT_PUBLIC_BASE_URL="https://yourdomain.com"
```

#### **`apps/ws/.env`** (WebSocket Server)
```env
# Server Configuration
PORT=8080
NODE_ENV=production

# Database
DATABASE_URL="postgresql://username:password@host:5432/collabydraw"

# JWT Secret (must match frontend)
JWT_SECRET="same-secret-as-frontend"
```

#### **`packages/db/.env`** (Prisma)
```env
DATABASE_URL="postgresql://username:password@host:5432/collabydraw"
```

### 2. Generate Secrets
```bash
# Generate NEXTAUTH_SECRET and JWT_SECRET
openssl rand -base64 32
```

---

## Database Setup

### Option 1: Managed Database (Recommended)
Use managed PostgreSQL services:
- **Neon** (Free tier available) - https://neon.tech
- **Supabase** (Free tier available) - https://supabase.com
- **Railway** - https://railway.app
- **AWS RDS**
- **DigitalOcean Managed Databases**

### Option 2: Self-Hosted PostgreSQL

1. **Install PostgreSQL**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# Create database
sudo -u postgres createdb collabydraw
sudo -u postgres psql
CREATE USER collabyuser WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE collabydraw TO collabyuser;
```

2. **Run Migrations**
```bash
cd packages/db
pnpm prisma migrate deploy
pnpm prisma generate
```

---

## Deployment Options

### 🌟 Option 1: Vercel + Railway (Recommended for Quick Start)

#### **Frontend (Vercel)**

1. **Push to GitHub**
```bash
git add .
git commit -m "Ready for deployment"
git push origin main
```

2. **Deploy on Vercel**
   - Go to https://vercel.com
   - Import your GitHub repository
   - Select the `apps/collabydraw` directory as root
   - Add environment variables from `apps/collabydraw/.env.local`
   - Deploy!

3. **Configure Build Settings**
   - Framework Preset: `Next.js`
   - Build Command: `cd ../.. && pnpm install && cd apps/collabydraw && pnpm build`
   - Output Directory: `.next`
   - Install Command: `pnpm install`

#### **WebSocket Server (Railway)**

1. **Deploy on Railway**
   - Go to https://railway.app
   - New Project → Deploy from GitHub
   - Select your repository
   - Add PostgreSQL database (optional if using external)
   - Set root directory to `apps/ws`

2. **Add Environment Variables**
   - Add all variables from `apps/ws/.env`
   - Railway will auto-generate `PORT`

3. **Configure Build**
   - Build Command: `pnpm install && pnpm build`
   - Start Command: `pnpm start`

4. **Generate Public Domain**
   - Railway Settings → Generate Domain
   - Use this URL for `NEXT_PUBLIC_WS_URL` in Vercel

---

### 🐳 Option 2: Docker Deployment (VPS/Cloud)

#### **1. Create Dockerfiles**

**`apps/collabydraw/Dockerfile`**
```dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install pnpm
RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

# Copy workspace files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY apps/collabydraw/package.json ./apps/collabydraw/
COPY packages ./packages

# Install dependencies
RUN pnpm install --frozen-lockfile

# Build stage
FROM base AS builder
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Generate Prisma Client
RUN cd packages/db && pnpm prisma generate

# Build Next.js
RUN cd apps/collabydraw && pnpm build

# Production stage
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/apps/collabydraw/public ./apps/collabydraw/public
COPY --from=builder --chown=nextjs:nodejs /app/apps/collabydraw/.next ./apps/collabydraw/.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/apps/collabydraw/package.json ./apps/collabydraw/package.json

USER nextjs

EXPOSE 3000

CMD ["node", "apps/collabydraw/.next/standalone/apps/collabydraw/server.js"]
```

**`apps/ws/Dockerfile`**
```dockerfile
FROM node:18-alpine AS base

FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY apps/ws/package.json ./apps/ws/
COPY packages ./packages

RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@9.0.0 --activate

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN cd packages/db && pnpm prisma generate
RUN cd apps/ws && pnpm build

FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodejs

COPY --from=builder /app/apps/ws/dist ./apps/ws/dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/packages/db ./packages/db

USER nodejs

EXPOSE 8080

CMD ["node", "apps/ws/dist/index.js"]
```

#### **2. Deploy with Docker Compose**

**Production `docker-compose.prod.yml`**
```yaml
version: '3.8'

services:
  database:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: collabydraw
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  websocket:
    build:
      context: .
      dockerfile: apps/ws/Dockerfile
    restart: always
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
      - PORT=8080
    depends_on:
      database:
        condition: service_healthy

  frontend:
    build:
      context: .
      dockerfile: apps/collabydraw/Dockerfile
    restart: always
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - NEXTAUTH_URL=${NEXTAUTH_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - JWT_SECRET=${JWT_SECRET}
      - NEXT_PUBLIC_WS_URL=${NEXT_PUBLIC_WS_URL}
    depends_on:
      - websocket
      - database

volumes:
  postgres-data:
```

**Deploy Commands**
```bash
# Build and start
docker-compose -f docker-compose.prod.yml up -d --build

# View logs
docker-compose -f docker-compose.prod.yml logs -f

# Stop
docker-compose -f docker-compose.prod.yml down
```

---

### 🔧 Option 3: VPS Manual Deployment

#### **1. Setup Server (Ubuntu 22.04)**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install pnpm
sudo npm install -g pnpm

# Install PM2 for process management
sudo npm install -g pm2

# Install Nginx
sudo apt install -y nginx

# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Install Certbot for SSL
sudo apt install -y certbot python3-certbot-nginx
```

#### **2. Clone and Build**

```bash
# Clone repository
git clone https://github.com/yourusername/CollabyDraw.git
cd CollabyDraw

# Install dependencies
pnpm install

# Setup database
cd packages/db
pnpm prisma migrate deploy
pnpm prisma generate
cd ../..

# Build applications
pnpm build
```

#### **3. PM2 Configuration**

**`ecosystem.config.js`**
```javascript
module.exports = {
  apps: [
    {
      name: 'collabydraw-frontend',
      cwd: './apps/collabydraw',
      script: 'pnpm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
        PORT: 3000
      }
    },
    {
      name: 'collabydraw-ws',
      cwd: './apps/ws',
      script: 'pnpm',
      args: 'start',
      env: {
        NODE_ENV: 'production',
        PORT: 8080
      }
    }
  ]
};
```

```bash
# Start with PM2
pm2 start ecosystem.config.js

# Save PM2 configuration
pm2 save

# Setup PM2 to start on boot
pm2 startup
```

#### **4. Nginx Configuration**

**`/etc/nginx/sites-available/collabydraw`**
```nginx
# Frontend
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# WebSocket Server
server {
    listen 80;
    server_name ws.yourdomain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }
}
```

```bash
# Enable site
sudo ln -s /etc/nginx/sites-available/collabydraw /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# Setup SSL with Let's Encrypt
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
sudo certbot --nginx -d ws.yourdomain.com
```

---

## Post-Deployment

### 1. Database Backup

**Automated Backup Script** (`backup.sh`)
```bash
#!/bin/bash
BACKUP_DIR="/var/backups/collabydraw"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

pg_dump -U collabyuser collabydraw | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Keep only last 7 days
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete
```

**Setup Cron**
```bash
chmod +x backup.sh
crontab -e
# Add: 0 2 * * * /path/to/backup.sh
```

### 2. DNS Configuration

Point your domains:
- `yourdomain.com` → Your server IP
- `www.yourdomain.com` → Your server IP
- `ws.yourdomain.com` → Your server IP (WebSocket)

### 3. Firewall Setup

```bash
# UFW Firewall
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

---

## Monitoring & Maintenance

### 1. Application Monitoring

**PM2 Monitoring**
```bash
pm2 monit                    # Real-time monitoring
pm2 logs                     # View logs
pm2 restart all              # Restart apps
pm2 reload all               # Zero-downtime reload
```

**Log Management**
```bash
# Setup log rotation
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
```

### 2. Performance Monitoring

Install monitoring tools:
- **New Relic** - Application performance monitoring
- **Sentry** - Error tracking
- **Uptime Robot** - Uptime monitoring
- **Grafana + Prometheus** - Metrics visualization

### 3. Regular Maintenance

```bash
# Update dependencies (test in staging first!)
pnpm update

# Update system
sudo apt update && sudo apt upgrade -y

# Monitor disk space
df -h

# Monitor database size
psql -U collabyuser collabydraw -c "SELECT pg_size_pretty(pg_database_size('collabydraw'));"

# Check SSL certificate expiry
sudo certbot certificates
```

---

## Scaling Considerations

### Horizontal Scaling

1. **Load Balancer** - Use Nginx or AWS ALB
2. **Multiple Frontend Instances** - Scale Next.js horizontally
3. **WebSocket Clustering** - Use Redis for pub/sub
4. **Database Replication** - Master-slave PostgreSQL setup
5. **CDN** - CloudFlare or AWS CloudFront for static assets

### Vertical Scaling

- Increase server resources (CPU, RAM)
- Optimize database queries
- Add database indexes
- Enable Redis caching

---

## Troubleshooting

### Common Issues

**WebSocket Connection Failed**
- Check firewall allows port 8080
- Verify `NEXT_PUBLIC_WS_URL` is correct
- Check SSL certificate on WebSocket domain

**Database Connection Error**
- Verify `DATABASE_URL` is correct
- Check database is running: `sudo systemctl status postgresql`
- Test connection: `psql $DATABASE_URL`

**JWT Token Issues**
- Ensure `JWT_SECRET` matches between frontend and WebSocket
- Check token expiry settings

**Build Failures**
- Clear node_modules: `rm -rf node_modules && pnpm install`
- Clear build cache: `rm -rf .next dist`
- Check Node.js version: `node -v` (should be 18+)

---

## Security Checklist

- [ ] All environment variables properly set
- [ ] Strong passwords for database
- [ ] SSL certificates installed and auto-renewing
- [ ] Firewall configured
- [ ] Database not publicly accessible
- [ ] Regular security updates enabled
- [ ] Rate limiting configured
- [ ] CORS properly configured
- [ ] XSS protection headers set
- [ ] Regular backups automated
- [ ] Monitoring and alerting setup

---

## Support & Resources

- **Documentation**: Update with your repo URL
- **Issues**: GitHub Issues
- **Discord/Slack**: Community support
- **Email**: Support email

---

**Happy Deploying! 🚀**
