# Judge0 on AWS Lightsail - Complete Setup Guide
*From zero to production-ready code execution API in 20 minutes*

---

## 📋 What You'll Build
A **self-hosted Judge0 API** that can:
- Execute code in 70+ programming languages
- Auto-grade LeetCode-style problems
- Handle competitive programming submissions
- Power your coding platform's backend

**Cost:** $3.50-$5/month (Lightsail instance)

---

## 🚀 STEP 1: Connect to Lightsail via SSH

### What We're Doing
Opening a terminal to your Ubuntu server in the cloud.

### How to Do It
```
Lightsail Console → Your Ubuntu instance → "Connect using SSH" button
```

**✅ Success:** Black terminal window opens with `ubuntu@ip-xxx-xxx:~$`

### Why This Step
You need command-line access to install software and configure the server.

---

## 🔧 STEP 2: Install Docker + Docker Compose

### What We're Doing
Installing Docker (runs containers) and Docker Compose (manages multiple containers).

### Commands
```bash
# Update package lists
sudo apt update

# Install Docker, Docker Compose, and utilities
sudo apt install -y docker.io docker-compose unzip wget curl

# Start Docker service and enable on boot
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group (no sudo needed later)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker-compose --version
docker ps
```

### Expected Output
```
Docker version 29.x.x, build xxxxx ✅
docker-compose version 1.29.x, build xxxxx ✅
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
(empty table - no containers yet)
```

### Why This Step
Judge0 runs in Docker containers. This isolates code execution safely and makes deployment easy.

---

## ⚙️ STEP 3: Fix Cgroup v1 (Ubuntu 22.04+ Issue)

### What We're Doing
Switching from cgroup v2 to v1 (required for Docker resource limits in Judge0).

### Check Current Status
```bash
stat -fc %T /sys/fs/cgroup
```

**If output is `cgroup2fs`** → Fix needed ⬇️  
**If output is `tmpfs`** → Skip to Step 4 ✅

### Fix Steps (Only if cgroup2fs)
```bash
# Open bootloader config
sudo nano /etc/default/grub
```

**Find and edit these 2 lines:**
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash systemd.unified_cgroup_hierarchy=0"
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"
```

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# Apply changes and reboot
sudo update-grub
sudo reboot
```

### After Reboot (Reconnect SSH)
```bash
stat -fc %T /sys/fs/cgroup
```
**Must show:** `tmpfs` ✅

### Why This Step
Judge0 uses Docker's resource limits (CPU, memory) to prevent malicious code from crashing the server. These limits require cgroup v1.

---

## 📥 STEP 4: Download & Extract Judge0

### What We're Doing
Getting the official Judge0 release from GitHub.

### Commands
```bash
# Download v1.13.1
wget https://github.com/judge0/judge0/releases/download/v1.13.1/judge0-v1.13.1.zip

# Extract
unzip judge0-v1.13.1.zip

# Enter directory
cd judge0-v1.13.1

# List files (optional - see what's inside)
ls -la
```

### What You'll See
```
docker-compose.yml       # Container orchestration config
judge0.conf             # Main configuration file
scripts/                # Helper scripts
```

### Why This Step
This package contains everything Judge0 needs: database schemas, API server, worker configurations, and Docker setup.

---

## 🔑 STEP 5: Configure Database Passwords

### What We're Doing
Setting secure passwords for Redis (cache) and PostgreSQL (database).

### Commands
```bash
# Open config file
nano judge0.conf
```

**Add these lines (change passwords!):**
```
REDIS_PASSWORD=your_strong_redis_password_here
POSTGRES_PASSWORD=your_strong_postgres_password_here
```

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

### Why This Step
Judge0 uses:
- **PostgreSQL:** Stores submission metadata, results, language configs
- **Redis:** Queues jobs and caches results

Default configs have no passwords → security risk.

---

## ▶️ STEP 6: Start Judge0 Services

### What We're Doing
Launching 4 Docker containers in the right order.

### Commands
```bash
# Start database + cache first
docker-compose up -d db redis
sleep 15

# Start API server + workers
docker-compose up -d
sleep 10

# Check status
docker ps
```

### Expected Output (4 containers)
```
CONTAINER ID   IMAGE             PORTS                      NAMES
abc123         judge0/judge0     0.0.0.0:2358->2358/tcp    judge0-v1131_server_1  ✅
def456         judge0/judge0     2358/tcp                   judge0-v1131_workers_1
ghi789         postgres:16.2     5432/tcp                   judge0-v1131_db_1
jkl012         redis:7.2.4       6379/tcp                   judge0-v1131_redis_1
```

### Why This Step
- **db (PostgreSQL):** Stores all data
- **redis:** Job queue for submissions
- **server:** REST API (port 2358)
- **workers:** Execute submitted code

Order matters: DB must be ready before API starts.

---

## 🧪 STEP 7: Test API Locally

### What We're Doing
Submitting test code to verify Judge0 works.

### Test 1: Hello World (Python)
```bash
curl -X POST "http://localhost:2358/submissions?base64_encoded=true&wait=true" \
  -H "Content-Type: application/json" \
  -d '{"source_code": "cHJpbnQoIkhlbGxvIFdvcmxkISIp", "language_id": 71}'
```

### Test 2: Math (Python)
```bash
curl -X POST "http://localhost:2358/submissions?base64_encoded=true&wait=true" \
  -H "Content-Type: application/json" \
  -d '{"source_code": "cHJpbnQoMiArIDIp", "language_id": 71}'
```

### Success Response
```json
{
  "stdout": "SGVsbG8gV29ybGQhCg==",
  "status": {
    "id": 3,
    "description": "Accepted"
  }
}
```

### Decode Output
```bash
echo "SGVsbG8gV29ybGQhCg==" | base64 -d
# Output: Hello World!
```

### Why This Step
Confirms:
- API server is running
- Workers can execute code
- Database stores results
- Base64 encoding/decoding works

---

## 🌐 STEP 8: Make Publicly Accessible

### Part A: Get Your Public IP
```bash
curl ifconfig.me
```
**Copy output:** `54.123.45.67` (example)

### Part B: Open Firewall
```
Lightsail Console → Your instance → Networking tab → "+ Add rule"

Application: Custom
Protocol: TCP
Port: 2358
Source: 0.0.0.0/0 (all IPv4 addresses)

→ Save
```

### Part C: Test Externally
```
Open browser: http://54.123.45.67:2358/docs
```

**Success:** Swagger UI API documentation loads! ✅

### Why This Step
- Default: Only localhost (127.0.0.1) can access Judge0
- Opening port 2358: Allows your frontend/backend to connect
- Public access: Required for production apps

---

## ✅ STEP 9: Backend Integration

### Environment Variables (.env)
```env
JUDGE0_URL=http://54.123.45.67:2358
JUDGE0_SUBMIT_URL=http://54.123.45.67:2358/submissions
JUDGE0_LANGUAGES_URL=http://54.123.45.67:2358/languages
```

### Node.js Example
```javascript
const JUDGE0_URL = process.env.JUDGE0_URL;

async function submitCode(code, languageId = 71) {
  const base64Code = Buffer.from(code).toString('base64');
  
  const response = await fetch(
    `${JUDGE0_URL}/submissions?base64_encoded=true&wait=true`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        source_code: base64Code,
        language_id: languageId
      })
    }
  );
  
  return response.json();
}

// Usage
const result = await submitCode('print("Hello World!")', 71);
console.log(result.status.description); // "Accepted"
```

### Why This Step
Your frontend sends code → Your backend forwards to Judge0 → Results return → Display to user.

---

## 🔄 Daily Operations

### View Logs
```bash
# API server logs
docker-compose logs -f server

# Worker logs (code execution)
docker-compose logs -f workers

# All logs
docker-compose logs -f
```

### Control Services
```bash
# Stop everything
docker-compose down

# Start everything
docker-compose up -d

# Restart
docker-compose restart

# Restart single service
docker-compose restart server
```

### Check Status
```bash
# Container health
docker ps

# System info
curl http://localhost:2358/system_info

# Supported languages
curl http://localhost:2358/languages
```

---

## 📊 Language IDs Reference

| Language | ID | Example |
|----------|----|---------| 
| Python 3 | 71 | `print("Hello")` |
| JavaScript | 63 | `console.log("Hello")` |
| C++ | 54 | `cout << "Hello";` |
| Java | 62 | `System.out.println("Hello");` |
| Go | 60 | `fmt.Println("Hello")` |
| Rust | 73 | `println!("Hello");` |
| C# | 51 | `Console.WriteLine("Hello");` |

**Full list:** `curl http://localhost:2358/languages`

---

## 🚨 Troubleshooting

### Problem: "Cannot connect to Docker daemon"
```bash
sudo systemctl start docker
sudo usermod -aG docker $USER
newgrp docker
```

### Problem: Containers exit immediately
```bash
# Check cgroup
stat -fc %T /sys/fs/cgroup  # Must be tmpfs

# Check logs
docker-compose logs
```

### Problem: "Address already in use (port 2358)"
```bash
# Find what's using port
sudo lsof -i :2358

# Kill and restart
docker-compose down
docker-compose up -d
```

### Problem: Submissions timeout
```bash
# Check worker status
docker-compose logs workers

# Restart workers
docker-compose restart workers
```

---

## 🎉 Success Checklist

- [ ] Docker installed: `docker --version` works
- [ ] Cgroup v1: `stat -fc %T /sys/fs/cgroup` → `tmpfs`
- [ ] 4 containers running: `docker ps` shows all
- [ ] Local test passes: `curl localhost:2358` returns JSON
- [ ] Firewall open: TCP 2358 rule added
- [ ] External access: `http://IP:2358/docs` loads Swagger UI
- [ ] Backend connected: `.env` variables set

---

## 📈 Next Steps

1. **Add HTTPS:** Use Nginx + Let's Encrypt for SSL
2. **Rate limiting:** Prevent abuse with API keys
3. **Monitoring:** Set up health checks
4. **Backups:** PostgreSQL database dumps
5. **Scaling:** Multiple worker containers for high traffic

---

## 🔗 Resources

- **Judge0 Docs:** https://ce.judge0.com/
- **GitHub:** https://github.com/judge0/judge0
- **API Reference:** http://YOUR_IP:2358/docs
- **Community:** Judge0 Discord/GitHub Issues

---

**Total Setup Time: 20 minutes**  
**Production-Ready LeetCode Judge: ✅**  
**Cost: $3.50-5/month (Lightsail)**

**Happy Coding! 🚀**
