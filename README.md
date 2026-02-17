# dhwani Application - Network Architecture & Flow

## Current Architecture (Production)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           INTERNET / USERS                              │
└───────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │ HTTP
                               │ http://x.x.x.x:3000
                               │ (Direct EC2 IP - No Load Balancer)
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    EC2 Instance (us-east-1)                             │
│  Public IP: x.x.x.x (dhwani-app-server)                                │
│  Private IP: 172.31.x.x                                                │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                    Docker Container                           │     │
│  │  Container Name: dhwani-app                                   │     │
│  │  Image: xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/         │     │
│  │          dhwani-app:${IMAGE_TAG}                              │     │
│  │  Port Mapping: 3000:3000 (Host:Container)                     │     │
│  │  Status: Running                                              │     │
│  └──────────────────┬───────────────────────────────────────────┘     │
│                     │                                                    │
│                     │ Container Port 3000                                │
│                     ▼                                                    │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │              dhwani Application (Node.js)                     │     │
│  │  - Express Server (port 3000)                               │     │
│  │  - Static Files (HTML, CSS, JS)                              │     │
│  │  - REST API (/api/call, /api/call/batch)                    │     │
│  │  - Health Check (/health)                                   │     │
│  └───────────────────────────┬───────────────────────────────────┘     │
└───────────────────────────────┼──────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
        ┌──────────────────┐    ┌──────────────────────┐
        │  LiveKit Cloud   │    │  Chatterbox Server    │
        │  (Phone Calls)   │    │  (AI Processing)      │
        │  wss://...       │    │  ws://y.y.y.y         │
        │  Port: 443       │    │  Port: 9000           │
        └──────────────────┘    └──────────────────────┘
                    │                       │
                    ▼                       ▼
        ┌──────────────────┐    ┌──────────────────────┐
        │  Twilio SIP      │    │  AI Model Server     │
        │  (Phone Network) │    │  - STT (Speech)      │
        │                  │    │  - LLM (Claude)      │
        │                  │    │  - TTS (Voice)       │
        └──────────────────┘    └──────────────────────┘
```

---

## Detailed Network Flow

### 1. **API Call Request Flow**

```
API Client (curl, Postman, etc.)
    │
    │ HTTP POST http://x.x.x.x:3000/api/call
    │ Body: {"phoneNumber": "+91XXXXXXXXXX"}
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ EC2 Security Group (dhwani-app-server)                      │
│ - Inbound Rule: Allow TCP 3000 from 0.0.0.0/0              │
│   (or restrict to specific IPs for security)                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ HTTP (port 3000)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Docker Container: dhwani-app                                │
│ - Port Mapping: Host 3000 → Container 3000                   │
│ - Environment: Loaded from .env file                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Express.js Application (server.js)                          │
│ 1. Receives POST /api/call                                  │
│ 2. Validates phone number                                   │
│ 3. Returns 200 OK immediately                               │
│ 4. Background: Calls makeCall() function                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ Background Process
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ makeCall() Function                                         │
│ 1. Creates LiveKit room                                     │
│ 2. Connects LiveKit agent                                   │
│ 3. Initiates Twilio SIP call                                │
│ 4. Bridges audio to Chatterbox WebSocket                    │
└─────────────────────────────────────────────────────────────┘
```

### 2. **WebSocket Connection Flow (Browser → Chatterbox)**

```
Browser JavaScript (app.js)
    │
    │ WebSocket Connection
    │ ws://y.y.y.y:9000
    │ (Direct connection, NOT through dhwani server)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Chatterbox Server Security Group (chatterbox-server)        │
│ - Inbound Rule: Allow TCP 9000 from 0.0.0.0/0              │
│   (or restrict to specific IPs)                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ WebSocket (ws://)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Chatterbox AI Server (Port 9000)                           │
│ - Receives audio chunks (binary PCM data)                   │
│ - Processes: STT → LLM → TTS                               │
│ - Sends back: Audio chunks + Transcripts                   │
└─────────────────────────────────────────────────────────────┘
```

### 3. **Phone Call Flow (Twilio → LiveKit → Chatterbox)**

```
Phone Caller
    │
    │ PSTN Call
    │ +1XXXXXXXXXX
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Twilio SIP Trunk                                            │
│ - Receives phone call                                       │
│ - Routes to LiveKit SIP endpoint                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ SIP Protocol
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ LiveKit Cloud (wss://your-project-id.livekit.cloud)        │
│ - Creates room: call-outbound-{phone}-{timestamp}          │
│ - SIP participant joins room                                │
│ - WebSocket connection for audio                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ WebSocket (wss://)
                            │ Audio Stream (Float32 PCM)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ dhwani Container (EC2 x.x.x.x)                             │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ LiveKit Agent (phoneCallHandler.js)                 │     │
│ │ - Connects to LiveKit room                          │     │
│ │ - Receives audio from phone call                    │     │
│ │ - Converts: Float32 (48kHz) → Int16 (16kHz)        │     │
│ │ - Sends to Chatterbox WebSocket                    │     │
│ └───────────────────────┬─────────────────────────────┘     │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            │ WebSocket (ws://)
                            │ Int16 PCM Audio (16kHz)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Chatterbox AI Server (y.y.y.y:9000)                         │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ AI Processing Pipeline                                │     │
│ │ 1. STT: Speech-to-Text (Faster-Whisper)              │     │
│ │ 2. LLM: Generate Response (Claude Haiku)             │     │
│ │ 3. TTS: Text-to-Speech (Chatterbox Turbo)           │     │
│ │ 4. Returns: Audio chunks + Transcript                │     │
│ └───────────────────────┬─────────────────────────────┘     │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            │ WebSocket Response
                            │ Audio chunks (WAV format)
                            │ JSON: transcripts, chunks
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ dhwani Container (EC2 x.x.x.x)                             │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ LiveKit Agent                                       │     │
│ │ - Receives audio from Chatterbox                    │     │
│ │ - Converts: Int16 (16kHz) → Float32 (48kHz)        │     │
│ │ - Publishes to LiveKit room                         │     │
│ └───────────────────────┬─────────────────────────────┘     │
└───────────────────────────┼──────────────────────────────────┘
                            │
                            │ LiveKit Audio Stream
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ LiveKit Cloud                                              │
│ - Sends audio to SIP participant                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ SIP Audio Stream
                            │
                            ▼
                    Phone Caller Hears AI Response
```

---

## Component Details

### **1. EC2 Instance - dhwani Application**

**Instance Details:**
- **Public IP:** x.x.x.x (dhwani-app-server)
- **Private IP:** 172.31.x.x
- **Region:** us-east-1
- **OS:** Ubuntu
- **Directory:** `/home/ubuntu/dhwani-app`

**Security Group:**
```
Inbound Rules:
- TCP 3000 from 0.0.0.0/0 (HTTP API access)
- TCP 22 from Jenkins IP (SSH for deployment)

Outbound Rules:
- All traffic (0.0.0.0/0)
```

**Docker Compose Configuration:**
```yaml
version: "3.9"

services:
  dhwani-app:
    image: xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/dhwani-app:${IMAGE_TAG}
    container_name: dhwani-app
    restart: always
    ports:
      - "3000:3000"   # Host:Container
    env_file:
      - .env
```

**Environment Variables (.env):**
```bash
AWS_REGION=us-east-1
EC2_WEBSOCKET_URL=ws://y.y.y.y:9000
LIVEKIT_API_KEY=YOUR_LIVEKIT_API_KEY
LIVEKIT_API_SECRET=YOUR_LIVEKIT_API_SECRET
LIVEKIT_SIP_TRUNK_ID=YOUR_SIP_TRUNK_ID
LIVEKIT_URL=wss://your-project-id.livekit.cloud
PORT=3000
TWILIO_ACCOUNT_SID=YOUR_TWILIO_ACCOUNT_SID
TWILIO_AUTH_TOKEN=YOUR_TWILIO_AUTH_TOKEN
TWILIO_PHONE=+1XXXXXXXXXX
```

---

### **2. EC2 Instance - Chatterbox AI Server**

**Instance Details:**
- **Public IP:** y.y.y.y (chatterbox-server)
- **Region:** us-east-1
- **Port:** 9000
- **Protocol:** WebSocket (ws://)
- **Purpose:** AI processing (STT → LLM → TTS)

**Security Group:**
```
Inbound Rules:
- TCP 9000 from x.x.x.x/32 (dhwani server only)
  OR
- TCP 9000 from 0.0.0.0/0 (if browser clients connect directly)

Outbound Rules:
- All traffic
```

---

### **3. Application Components**

**Express.js Server (server.js):**
- **Port:** 3000
- **Endpoints:**
  - `GET /` - Web UI (index.html)
  - `GET /health` - Health check
  - `POST /api/call` - Initiate phone call
  - `POST /api/call/batch` - Batch phone calls
  - Static files: `/public/*`

**LiveKit Agent (phoneCallHandler.js):**
- Connects to LiveKit rooms
- Bridges audio between phone calls and Chatterbox WebSocket
- Handles audio format conversion

**Chatterbox WebSocket Bridge (EC2WebSocketBridge.js):**
- Manages WebSocket connection to Chatterbox server
- Handles audio streaming
- Processes transcripts and responses

---

### **4. External Services**

**LiveKit Cloud:**
- **URL:** `wss://your-project-id.livekit.cloud`
- **Purpose:** Real-time audio/video infrastructure
- **Features:**
  - SIP trunk integration
  - Room management
  - Audio streaming

**Chatterbox AI Server:**
- **IP:** y.y.y.y
- **Port:** 9000
- **Protocol:** WebSocket (ws://)
- **Purpose:** AI processing (STT → LLM → TTS)
- **Region:** us-east-1

**Twilio:**
- **Purpose:** Phone number and SIP trunk
- **Phone Number:** +1XXXXXXXXXX
- **Integration:** LiveKit SIP trunk

**AWS ECR:**
- **Registry:** `xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com`
- **Repository:** `dhwani-app`
- **Region:** us-east-1
- **Purpose:** Docker image storage

---

## Deployment Flow (Jenkins)

```
┌─────────────────────────────────────────────────────────────┐
│ Jenkins Pipeline                                             │
│ 1. Git Checkout (Branch: ${Branch})                         │
│ 2. Build Docker Image (linux/amd64)                         │
│ 3. Push to ECR (us-east-1)                                   │
│ 4. SSH to EC2                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ SSH (port 22)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ EC2 Instance (x.x.x.x)                                      │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Step 1: Login to ECR                                 │     │
│ │   aws ecr get-login-password | docker login          │     │
│ └─────────────────────────────────────────────────────┘     │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Step 2: Pull new image                               │     │
│ │   docker pull xxxxxxxxxxxx.dkr.ecr.../dhwani-app:${BUILD_NUMBER}│
│ └─────────────────────────────────────────────────────┘     │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Step 3: Update docker-compose.yml                   │     │
│ │   export IMAGE_TAG=${BUILD_NUMBER}                  │     │
│ └─────────────────────────────────────────────────────┘     │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Step 4: Recreate container                          │     │
│ │   docker compose up -d --no-deps                    │     │
│ │   (Stops old, starts new - ~5-10s downtime)         │     │
│ └─────────────────────────────────────────────────────┘     │
│ ┌─────────────────────────────────────────────────────┐     │
│ │ Step 5: Cleanup                                      │     │
│ │   docker image prune -f                             │     │
│ └─────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**Deployment Timeline:**
- **T+0s:** Old container running
- **T+5s:** New image pulled
- **T+8s:** Old container stopped
- **T+10s:** New container started
- **T+12s:** Container healthy
- **Downtime:** ~5-10 seconds (during container swap)

---

## Network Ports Summary

| Component | Port | Protocol | Purpose |
|-----------|------|----------|---------|
| dhwani EC2 | 3000 | HTTP | Application API |
| dhwani EC2 | 22 | SSH | Deployment access |
| Chatterbox Server | 9000 | WebSocket | AI processing |
| LiveKit | 443 | WSS | WebSocket (TLS) |
| Twilio SIP | 5060 | SIP | Phone calls |

---

## Data Flow Summary

**Request Types:**

1. **API Call Request:**
   ```
   Client → http://x.x.x.x:3000/api/call
   → Docker Container (dhwani-app:3000)
   → Express.js Application
   → Background: makeCall() → LiveKit → Twilio → Phone Call
   ```

2. **WebSocket (Browser):**
   ```
   Browser → ws://y.y.y.y:9000 (Direct, no dhwani server)
   → Chatterbox AI Server
   → STT → LLM → TTS
   → Audio chunks + Transcripts back to browser
   ```

3. **Phone Call:**
   ```
   Phone → Twilio → LiveKit → dhwani Container
   → WebSocket → Chatterbox Server (y.y.y.y:9000)
   → AI Processing
   → Audio back through chain → Phone
   ```

---

## Tools & Technologies

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Compute** | EC2 (us-east-1) | Application hosting |
| **Containerization** | Docker | Application packaging |
| **Orchestration** | Docker Compose | Container management |
| **Registry** | AWS ECR (us-east-1) | Docker image storage |
| **CI/CD** | Jenkins | Automated deployment |
| **Runtime** | Node.js 20 | Application runtime |
| **Framework** | Express.js | Web server |
| **Real-time** | LiveKit | Audio/video infrastructure |
| **Telephony** | Twilio | Phone number & SIP |
| **AI Processing** | Chatterbox Server (us-east-1) | STT, LLM, TTS |
| **Security** | AWS Security Groups | Network firewall |

---

## Current Deployment Process

### Jenkins Script Flow:

```bash
1. Checkout code from branch
2. Build Docker image (linux/amd64)
3. Login to ECR (us-east-1)
4. Tag and push image to ECR
5. SSH to EC2 (x.x.x.x)
6. Pull new image
7. Set IMAGE_TAG environment variable
8. Run: docker compose up -d --no-deps
9. Cleanup old images
```

### Manual Deployment:

```bash
# SSH to EC2
ssh ubuntu@x.x.x.x

# Navigate to app directory
cd /home/ubuntu/dhwani-app

# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com

# Pull latest image
docker pull xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com/dhwani-app:latest

# Update and restart
export IMAGE_TAG=latest
docker compose up -d --no-deps

# Check status
docker ps
docker logs -f dhwani-app
```

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Internet                                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ HTTP (port 3000)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ EC2 Security Group (dhwani-app-server)                       │
│ - Allow TCP 3000 from 0.0.0.0/0                             │
│ - Allow TCP 22 from Jenkins IP                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ Docker Container (dhwani-app)                              │
│ - Isolated network namespace                                │
│ - Only exposes port 3000                                    │
│ - Environment variables from .env                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Monitoring & Health Checks

### Health Check Endpoint
```bash
curl http://x.x.x.x:3000/health
```

**Response:**
```json
{
  "status": "ok",
  "service": "conversational-ai-client",
  "timestamp": "2026-02-17T07:00:00.000Z"
}
```

### Container Status
```bash
# Check running containers
docker ps

# Check logs
docker logs -f dhwani-app

# Check resource usage
docker stats dhwani-app

# Check compose status
docker compose ps
```

---

## Future Enhancements

### When Adding ALB:

1. **Create Target Group:**
   - Target: EC2 instance (x.x.x.x)
   - Port: 3000
   - Health check: `/health`

2. **Create ALB:**
   - Listener: HTTPS 443
   - Forward to target group

3. **Update Security Group:**
   - Allow port 3000 from ALB security group only
   - Remove public access to port 3000

4. **Update API endpoint:**
   - From: `http://x.x.x.x:3000/api/call`
   - To: `https://your-domain.com/dhwani/api/call`

### Zero-Downtime Deployment:

Currently using simple `docker compose up -d` which has ~5-10s downtime.

For true zero-downtime, implement blue-green:
- Add `dhwani-app-green` service to docker-compose.yml
- Start green on port 3001
- Health check green
- Stop blue, start new blue on port 3000
- Cleanup green

---

## Cost Estimation (Monthly)

| Component | Cost |
|-----------|------|
| EC2 (dhwani) | ~$30-50 (depends on instance type) |
| EC2 (Chatterbox) | ~$50-100 (GPU instance for AI) |
| ECR Storage | ~$1-5 |
| Data Transfer | ~$5-20 |
| **Total** | **~$86-175/month** |

*Excludes: LiveKit, Twilio costs*

---

## Quick Reference

**Current Setup:**
- **Application:** http://x.x.x.x:3000
- **API:** http://x.x.x.x:3000/api/call
- **Health:** http://x.x.x.x:3000/health
- **Chatterbox:** ws://y.y.y.y:9000

**Deployment:**
- **Jenkins:** Automated via Jenkins script
- **Manual:** `docker compose up -d --no-deps`

**Files on EC2:**
- `/home/ubuntu/dhwani-app/docker-compose.yml`
- `/home/ubuntu/dhwani-app/.env`

**ECR:**
- Registry: `xxxxxxxxxxxx.dkr.ecr.us-east-1.amazonaws.com`
- Image: `dhwani-app:${BUILD_NUMBER}`

---

This architecture provides:
- ✅ Simple deployment (Docker Compose)
- ✅ Direct access (no ALB overhead)
- ✅ Cost-effective (no ALB charges)
- ⚠️  Small downtime during deployments (~5-10s)
- ⚠️  Public IP exposure (consider ALB for production)
