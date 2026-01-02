# Quick Deployment Reference

## 🚀 Quick Start (Recommended)

### Vercel + Railway Deployment (5 minutes)

1. **Deploy Frontend on Vercel**
   ```bash
   # Push to GitHub
   git push origin main
   
   # Go to vercel.com → Import from GitHub
   # Root Directory: apps/collabydraw
   # Add environment variables from apps/collabydraw/.env.example
   ```

2. **Deploy WebSocket on Railway**
   ```bash
   # Go to railway.app → Deploy from GitHub
   # Root Directory: apps/ws
   # Add PostgreSQL service
   # Add environment variables from apps/ws/.env.example
   ```

3. **Update Frontend with WS URL**
   - Copy Railway WebSocket URL
   - Update `NEXT_PUBLIC_WS_URL` in Vercel
   - Redeploy

---

## 🐳 Docker Deployment (VPS)

```bash
# 1. Clone repository
git clone <your-repo>
cd CollabyDraw

# 2. Copy and configure environment
cp .env.example .env
# Edit .env with your values

# 3. Build and run
docker-compose -f docker-compose.prod.yml up -d --build

# 4. Run migrations
docker-compose -f docker-compose.prod.yml exec frontend npx prisma migrate deploy
```

---

## 📋 Environment Variables Checklist

### Generate Secrets
```bash
# Generate JWT_SECRET and NEXTAUTH_SECRET
openssl rand -base64 32
```

### Required Variables
- [ ] `DATABASE_URL` - PostgreSQL connection string
- [ ] `JWT_SECRET` - Same in frontend & websocket
- [ ] `NEXTAUTH_SECRET` - NextAuth secret key
- [ ] `NEXTAUTH_URL` - Your domain URL
- [ ] `NEXT_PUBLIC_WS_URL` - WebSocket server URL (wss://...)
- [ ] `NEXT_PUBLIC_BASE_URL` - Frontend URL

---

## 🔧 Manual VPS Setup

```bash
# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install PM2
sudo npm install -g pm2 pnpm

# Clone and build
git clone <your-repo>
cd CollabyDraw
pnpm install
pnpm build

# Setup database
cd packages/db
pnpm prisma migrate deploy
cd ../..

# Start with PM2
pm2 start ecosystem.config.js
pm2 save
pm2 startup
```

---

## 🌐 Domain & SSL Setup

```bash
# Install Nginx & Certbot
sudo apt install nginx certbot python3-certbot-nginx

# Configure Nginx (see DEPLOYMENT_GUIDE.md)
sudo nano /etc/nginx/sites-available/collabydraw

# Enable site
sudo ln -s /etc/nginx/sites-available/collabydraw /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# Setup SSL
sudo certbot --nginx -d yourdomain.com -d ws.yourdomain.com
```

---

## 📊 Monitoring

```bash
# PM2 monitoring
pm2 monit
pm2 logs

# Docker monitoring
docker-compose -f docker-compose.prod.yml logs -f
docker stats
```

---

## 🆘 Common Issues

**WebSocket won't connect**
- Check firewall allows port 8080
- Use `wss://` for HTTPS sites
- Verify `NEXT_PUBLIC_WS_URL` is correct

**Database migration failed**
- Run: `cd packages/db && pnpm prisma migrate deploy`
- Check DATABASE_URL is correct

**JWT Token errors**
- Ensure JWT_SECRET matches in both frontend and websocket

---

## 📚 Full Documentation

See [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) for complete deployment instructions.

---

**Need Help?** Check the full deployment guide or open an issue on GitHub.
