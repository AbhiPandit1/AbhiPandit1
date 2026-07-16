# AmoreVnb — Deployment Guide

## Prerequisites
- AWS CLI configured with profile `amorebnb`
- Docker Desktop running
- Node.js 20+

---

## Step-by-Step Execution Order

### Step 1 — Copy Dockerfiles into your projects

```bash
# Backend
cp backend/Dockerfile ~/pavitra-project/amorebnb-backend/
cp backend/.dockerignore ~/pavitra-project/amorebnb-backend/

# Chat
cp chat/Dockerfile ~/pavitra-project/amorebnb-chat/

# Frontend (only if doing ECS, skip if using S3)
cp frontend/Dockerfile ~/pavitra-project/amor-bnb/
cp frontend/nginx.conf ~/pavitra-project/amor-bnb/
```

---

### Step 2 — Store secrets in SSM

Edit `scripts/store-secrets.sh` and fill in:
- `YOUR_RDS_DB_PASSWORD` — your RDS master password
- `YOUR_AWS_SECRET_ACCESS_KEY` — AWS secret for S3 uploads
- `YOUR_GOOGLE_CLIENT_SECRET` — from Google Cloud Console

Then run:
```bash
chmod +x scripts/store-secrets.sh
./scripts/store-secrets.sh
```

---

### Step 3 — Add health check to backend

Make sure your backend has a `/health` route:
```typescript
// In your Express app
app.get('/health', (req, res) => res.json({ status: 'ok' }));
```

---

### Step 4 — Create IAM Role (if not exists)

In AWS Console → IAM → Roles, ensure `ecsTaskExecutionRole` exists with:
- `AmazonECSTaskExecutionRolePolicy`
- `AmazonSSMReadOnlyAccess` (for secrets)

---

### Step 5 — Run the initial setup (ALB + CloudFront)

```bash
chmod +x scripts/create-alb.sh scripts/create-cloudfront.sh
./scripts/create-alb.sh       # Creates ALB, Target Groups, ECS Services
./scripts/create-cloudfront.sh # Creates S3 bucket + CloudFront
```

---

### Step 6 — Get ACM SSL Certificate

1. Go to AWS Console → Certificate Manager → ap-south-1
2. Request public cert for `*.amorevnb.com` and `amorevnb.com`
3. Validate via DNS (add CNAME at your registrar)
4. Copy the cert ARN
5. Uncomment the HTTPS listener section in `create-alb.sh`, paste ARN, re-run

Also request a cert in **us-east-1** (required for CloudFront).

---

### Step 7 — Full deploy

```bash
chmod +x scripts/deploy.sh
./scripts/deploy.sh
```

---

### Step 8 — DNS Setup

At your domain registrar (or Route 53):

| Record | Type | Value |
|--------|------|-------|
| `amorevnb.com` | CNAME / Alias | CloudFront domain (xxxx.cloudfront.net) |
| `api.amorevnb.com` | CNAME | ALB DNS name |
| `chat.amorevnb.com` | CNAME | ALB DNS name |

---

### Step 9 — Update Google OAuth

In Google Cloud Console → Credentials → OAuth Client:
- Add Authorized redirect URI: `https://api.amorevnb.com/api/auth/google/callback`

---

### Step 10 — Smoke Test

```bash
curl https://api.amorevnb.com/health        # → {"status":"ok"}
curl https://chat.amorevnb.com/             # → Socket.IO response
open https://amorevnb.com                   # → Frontend loads
```

---

## Ongoing Redeploys

After code changes:
```bash
./scripts/redeploy.sh api       # Redeploy only backend
./scripts/redeploy.sh chat      # Redeploy only chat
./scripts/redeploy.sh frontend  # Redeploy only frontend
./scripts/redeploy.sh all       # Redeploy everything
```

---

## Monitoring

```bash
# View ECS service logs
aws logs tail /ecs/amorebnb-api --follow --region ap-south-1 --profile amorebnb
aws logs tail /ecs/amorebnb-chat --follow --region ap-south-1 --profile amorebnb

# Check ECS service status
aws ecs describe-services --cluster amorebnb-cluster \
  --services amorebnb-api amorebnb-chat \
  --region ap-south-1 --profile amorebnb \
  --query 'services[*].{Name:serviceName,Running:runningCount,Desired:desiredCount,Status:status}'
```
