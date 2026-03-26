 STEP 1: Connect to Lightsail (30 seconds)
text
Lightsail Console → Your Ubuntu instance → "Connect using SSH"
Black terminal opens → Ready!

🔧 STEP 2: Install Docker + Docker Compose (5 minutes)
bash
# Update system
sudo apt update

# Install Docker + Docker Compose
sudo apt install docker.io docker-compose unzip wget curl -y

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (no sudo needed later)
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker-compose --version
docker ps
Expected:

text
Docker version 29.x.x ✅
docker-compose version 1.29.x ✅
CONTAINER ID table (empty) ✅
⚙️ STEP 3: Fix Cgroup v1 (CRITICAL - 4 minutes)
bash
# Check cgroup (Ubuntu 22.04+ problem)
stat -fc %T /sys/fs/cgroup
Shows cgroup2fs? → Fix needed:

bash
sudo nano /etc/default/grub
Edit these EXACT 2 lines:

text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash systemd.unified_cgroup_hierarchy=0"
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"
Save: Ctrl+O → Enter → Ctrl+X

bash
sudo update-grub
sudo reboot
After reboot (reconnect SSH):

bash
stat -fc %T /sys/fs/cgroup
MUST show tmpfs ✅

📥 STEP 4: Download + Configure Judge0 (3 minutes)
bash
# Download Judge0 v1.13.1
wget https://github.com/judge0/judge0/releases/download/v1.13.1/judge0-v1.13.1.zip
unzip judge0-v1.13.1.zip
cd judge0-v1.13.1
Set database passwords:

bash
nano judge0.conf
Add these lines (change passwords):

text
REDIS_PASSWORD=myredis123securepass
POSTGRES_PASSWORD=mypostgres456securepass
Save: Ctrl+O → Enter → Ctrl+X

▶️ STEP 5: Start Judge0 (2 minutes)
bash
# Start database first
docker-compose up -d db redis
sleep 15

# Start everything
docker-compose up -d
sleep 10

# Check status
docker ps
Success (4 containers running):

text
CONTAINER ID   IMAGE             PORTS                    NAMES
xxx            judge0/judge0     0.0.0.0:2358->2358/tcp  judge0-v1131_server_1  ✅
xxx            judge0/judge0     2358/tcp                 judge0-v1131_workers_1
xxx            postgres          5432/tcp                 judge0-v1131_db_1
xxx            redis             6379/tcp                 judge0-v1131_redis_1
🧪 STEP 6: Test Locally (1 minute)
Copy-paste these EXACT commands:

bash
# Hello World (Python)
curl -X POST "http://localhost:2358/submissions?base64_encoded=true&wait=true" -H "Content-Type: application/json" -d '{"source_code": "cHJpbnQoIkhlbGxvIFdvcmxkISIp", "language_id": 71}'
bash
# 2+2 (Python)
curl -X POST "http://localhost:2358/submissions?base64_encoded=true&wait=true" -H "Content-Type: application/json" -d '{"source_code": "cHJpbnQoMiArIDIp", "language_id": 71}'
Success looks like:

json
{"stdout": "SGVsbG8gV29ybGQhCg==", "status": {"id": 3, "description": "Accepted"}} ✅
Decode output:

bash
echo "SGVsbG8gV29ybGQhCg==" | base64 -d
Result: Hello World! ✅

🌐 STEP 7: Make Public (2 minutes)
7A. Get Public IP:
bash
curl ifconfig.me
Example output: 54.123.45.67 ← Copy this!

7B. Open Firewall:
text
Lightsail Console → Your instance → Networking → "+ Add rule"
Application: Custom
Protocol: TCP  
Port: 2358
Source: 0.0.0.0/0 (All IPv4)
→ Save
7C. Test Externally:
text
Browser → http://54.123.45.67:2358/docs
Swagger UI loads = SUCCESS! ✅

✅ STEP 8: Backend Integration (.env)
text
# Your backend/frontend config
JUDGE0_URL=http://54.123.45.67:2358
JUDGE0_SUBMIT_URL=http://54.123.45.67:2358/submissions
JUDGE0_LANGUAGES_URL=http://54.123.45.67:2358/languages
🔄 Essential Commands (Bookmark This!)
bash
# Logs
docker-compose logs server      # API logs
docker-compose logs workers     # Code execution

# Control
docker-compose down             # Stop
docker-compose up -d            # Start
docker-compose restart          # Restart

# Status
curl http://localhost:2358/languages     # 70+ languages
curl http://localhost:2358/system_info   # System status
🎉 TOTAL TIME: 20 minutes
text
SSH → Docker → Cgroup fix → Judge0 → Live API → Frontend ready!
✅ Final Checklist
Step	✅ Check	Command
Docker	docker ps works	No permission error
Cgroup	stat -fc %T /sys/fs/cgroup	Shows tmpfs
Containers	docker ps	4 containers + port 2358
Local test	curl localhost:2358	"Accepted" status
Firewall	Lightsail Networking	TCP 2358 open
External	http://IP:2358/docs	Swagger UI
Production-ready LeetCode judge → React/Next.js integration ready!
