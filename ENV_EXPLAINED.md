# 🔧 Environment Variables Explained

## Internal vs External URLs

### ✅ Container Names (Internal Communication)

These are already configured in `docker-compose.yml` and work automatically:

```yaml
# Frontend connects to database
DATABASE_URL=postgresql://collabyuser:password@database:5432/collabydraw
                                               ^^^^^^^^
                                               Container name - works inside Docker network

# WebSocket connects to database  
DATABASE_URL=postgresql://collabyuser:password@database:5432/collabydraw
                                               ^^^^^^^^
                                               Container name - works inside Docker network
```

**Why this works:**
- All containers are on the same Docker network (`collabydraw-net`)
- Docker's internal DNS resolves container names
- No need for localhost or public IPs

---

### ❌ Cannot Use Container Names (Browser/Client URLs)

These MUST be public URLs (EC2 IP or domain):

#### 1. `NEXTAUTH_URL`
```env
# ❌ Wrong - browser can't access container names
NEXTAUTH_URL=http://frontend:3000

# ❌ Wrong - localhost only works on local machine
NEXTAUTH_URL=http://localhost:3000

# ✅ Correct - public EC2 IP
NEXTAUTH_URL=http://3.145.67.89:3000

# ✅ Correct - your domain
NEXTAUTH_URL=https://yourdomain.com
```

**Why:** NextAuth redirects the user's browser for OAuth login. The browser needs a publicly accessible URL.

---

#### 2. `NEXT_PUBLIC_WS_URL`
```env
# ❌ Wrong - browser can't access container names
NEXT_PUBLIC_WS_URL=ws://websocket:8080

# ❌ Wrong - localhost only works on local machine
NEXT_PUBLIC_WS_URL=ws://localhost:8080

# ✅ Correct - public EC2 IP
NEXT_PUBLIC_WS_URL=ws://3.145.67.89:8080

# ✅ Correct - your domain with SSL
NEXT_PUBLIC_WS_URL=wss://yourdomain.com/ws
```

**Why:** The browser (running on user's computer) needs to connect to the WebSocket server. It must use the public IP or domain.

---

#### 3. `NEXT_PUBLIC_BASE_URL`
```env
# ❌ Wrong - sharing links won't work
NEXT_PUBLIC_BASE_URL=http://frontend:3000

# ❌ Wrong - only works locally
NEXT_PUBLIC_BASE_URL=http://localhost:3000

# ✅ Correct - public EC2 IP
NEXT_PUBLIC_BASE_URL=http://3.145.67.89:3000

# ✅ Correct - your domain
NEXT_PUBLIC_BASE_URL=https://yourdomain.com
```

**Why:** Used for generating shareable room links. Other users need to access these URLs.

---

## Summary

### 🏗️ Internal (Container-to-Container)
- ✅ Use container names (`database`, `websocket`, `frontend`)
- ✅ Already configured in `docker-compose.yml`
- ✅ No changes needed

### 🌐 External (Browser/Client Access)
- ✅ Use public EC2 IP (e.g., `3.145.67.89`)
- ✅ Or use your domain (e.g., `yourdomain.com`)
- ❌ Cannot use `localhost` or container names
- ❌ These are accessed by user's browser, not containers

---

## Complete Example

### Development (Local Machine)
```env
NEXTAUTH_URL=http://localhost:3000
NEXT_PUBLIC_WS_URL=ws://localhost:8080
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

### Production (EC2 with IP)
```env
NEXTAUTH_URL=http://3.145.67.89:3000
NEXT_PUBLIC_WS_URL=ws://3.145.67.89:8080
NEXT_PUBLIC_BASE_URL=http://3.145.67.89:3000
```

### Production (EC2 with Domain + SSL)
```env
NEXTAUTH_URL=https://draw.yourdomain.com
NEXT_PUBLIC_WS_URL=wss://draw.yourdomain.com/ws
NEXT_PUBLIC_BASE_URL=https://draw.yourdomain.com
```

---

## Why NEXT_PUBLIC_ Variables Are Special

Variables prefixed with `NEXT_PUBLIC_` are:
- ✅ Exposed to the browser client
- ✅ Bundled into the JavaScript code
- ✅ Accessible from client-side React components

This means they **must** contain URLs that the user's browser can reach, not internal Docker container names.

---

## Quick Check

**Ask yourself:** "Who needs to access this URL?"

- **Docker containers** → Use container name
- **User's browser** → Use public IP or domain
- **OAuth provider** → Use public IP or domain
