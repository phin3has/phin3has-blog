+++
date = '2025-07-26T11:41:42-06:00'
draft = false
title = 'Running Sublime Security in K8s'
+++
# Sublime Security on Kubernetes: A Home Labber's Journey Through Email Security and SSO Poo

*Or: How I Spent Two Days Fighting Authentik SAML Only to Discover It Was a Simple Environment Variable*

## The Beginning: "This Looks Cool!"

Picture this: It's a Thursday evening and you stumble upon Sublime Security - an "open" email security platform that promises to protect your homelab email with the same sophistication as enterprise solutions. Their Docker quickstart? One line: `curl -sL https://sublime.security/install.sh | sh`. 

"How hard could it be to run this on Kubernetes?" you think. "I'll just convert the Docker Compose file, slap it behind Authentik, and call it a night."

*Narrator: It was not, in fact, called a night.*

## Why Sublime Security?

Before we dive into the trenches, let's talk about why Sublime Security is worth the effort:

- **Detection-as-Code**: Write email security rules like you write code
- **Transparency**: See exactly how threats are detected (no black box nonsense)
- **Community-Driven**: 1,200+ detection rules with a third from the community
- **Strelka Integration**: Enterprise-grade file scanning from Target's security team
- **ML-Powered**: Actual AI that does useful things (shocking, I know)
- **Free for up to 600 mailboxes**: Perfect for home labs!

## The Challenge: Docker Compose → Kubernetes

Sublime provides a nice Docker Compose setup, but we're Kubernetes people here. We don't do "docker-compose up" - we do "kubectl apply -f" with 37 YAML files and YOLO vibes.

Here's what we're dealing with:
- **9 different services** (Mantis, Bora-lite, Dashboard, PostgreSQL, Redis, Strelka components, MinIO)
- **Complex inter-service communication**
- **Persistent storage requirements**
- **No official Kubernetes support** (that's right, we're pioneers!)
- **Integration with Authentik for SSO**

## Prerequisites: What You'll Need

Before we start this adventure, make sure you have:

1. **A Kubernetes cluster** (k3s, k8s, whatever floats your boat)
2. **Traefik** as your ingress controller (or adapt for nginx/others)
3. **Authentik** deployed and working
4. **Cloudflare Tunnel** or similar for external access
5. **ArgoCD** (optional but recommended for GitOps)
6. **Patience** (required for the SSO debugging)
7. **Coffee** (lots of it)

## Step 1: Understanding the Architecture

First, let's understand what we're building. Sublime Security isn't just one container - it's an entire ecosystem:

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
└────────────────────────┬────────────────────────────────────┘
                         │
                    Cloudflare Tunnel
                         │
┌────────────────────────┴────────────────────────────────────┐
│                     Traefik Ingress                          │
│                           │                                  │
│                   Authentik Middleware                       │
│                           │                                  │
│    ┌──────────────────────┴──────────────────────┐         │
│    │                                              │         │
│    ▼                                              ▼         │
│ sublime.domain.com                    sublime-api.domain.com│
│    │                                              │         │
│    ▼                                              ▼         │
│ Dashboard (:3000)                          Mantis API (:8000)│
│                                                   │         │
│                          ┌────────────────────────┤         │
│                          │                        │         │
│                          ▼                        ▼         │
│                    Bora-lite              ┌──────────────┐  │
│                   (Task Queue)            │              │  │
│                                          │  PostgreSQL   │  │
│                 ┌─────────────┐         │              │  │
│                 │   Strelka   │         └──────────────┘  │
│                 │  Frontend   │                            │
│                 │  Backend    │         ┌──────────────┐  │
│                 │  Manager    │         │              │  │
│                 │ Coordinator │         │    Redis     │  │
│                 └─────────────┘         │              │  │
│                                         └──────────────┘  │
│                 ┌─────────────┐                            │
│                 │             │         ┌──────────────┐  │
│                 │   Hydra     │         │              │  │
│                 │  (ML/AI)    │         │    MinIO     │  │
│                 │             │         │   (S3-like)  │  │
│                 └─────────────┘         └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Step 2: Setting Up the Namespace and Secrets

Let's start with the basics. Create your namespace and secrets:

```bash
# Create the namespace
kubectl create namespace sublime

# Generate secure passwords
export POSTGRES_PASSWORD=$(openssl rand -base64 32)
export JWT_SECRET=$(openssl rand -base64 64)
export POSTGRES_ENCRYPTION_KEY=$(openssl rand -base64 32)

# Create the secret (we'll use SealedSecrets later)
kubectl create secret generic sublime-secrets \
  --namespace=sublime \
  --from-literal=POSTGRES_PASSWORD="$POSTGRES_PASSWORD" \
  --from-literal=JWT_SECRET="$JWT_SECRET" \
  --from-literal=POSTGRES_ENCRYPTION_KEY="$POSTGRES_ENCRYPTION_KEY"
```

## Step 3: Deploy PostgreSQL

PostgreSQL needs special attention - the PGDATA path must be a subdirectory:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sublime-postgres
  namespace: sublime
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sublime-postgres
  template:
    metadata:
      labels:
        app: sublime-postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13.2
        args: ["-c", "max_connections=200"]
        env:
        - name: POSTGRES_USER
          value: sublime
        - name: POSTGRES_DB
          value: mantis
        - name: PGDATA
          value: /data/postgres/pgdata  # CRITICAL: Must be subdirectory!
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: sublime-secrets
              key: POSTGRES_PASSWORD
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /data/postgres
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: sublime-postgres-pvc
```

## Step 4: Deploy Redis and MinIO

These are straightforward, but remember MinIO needs bucket initialization:

```yaml
# MinIO deployment snippet
apiVersion: batch/v1
kind: Job
metadata:
  name: sublime-create-buckets
  namespace: sublime
spec:
  template:
    spec:
      restartPolicy: OnFailure
      initContainers:
      - name: wait-for-minio
        image: busybox:1.28
        command: ['sh', '-c', 'until nc -zv sublimes3 8110; do echo "waiting for minio..."; sleep 2; done;']
      containers:
      - name: create-buckets
        image: minio/mc
        command: 
        - /bin/sh
        - -c
        - |
          sleep 15  # Give MinIO time to fully start
          /usr/bin/mc alias set myminio http://sublimes3:8110 minioadmin minioadmin
          /usr/bin/mc mb myminio/email-screenshots || true
          /usr/bin/mc ls myminio
```

## Step 5: The Strelka Beast

Strelka is complex - it needs a coordinator, frontend, backend, and manager. Here's the key insight: ConfigMaps for configuration!

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: strelka-frontend-config
  namespace: sublime
data:
  frontend.yaml: |
    server: ":57314"
    coordinator:
      addr: 'sublime-strelka-coordinator:6379'
      db: 0
      pool: 100
      read: 10s
    response:
      log: "/var/log/strelka/strelka.log"
```

## Step 6: The Core Services - Mantis and Bora-lite

Here's where things get interesting. These services need specific environment variables:

```yaml
# Critical environment variables for Mantis
env:
- name: STRELKA_URL
  value: sublime-strelka-frontend  # Just hostname, no port!
- name: CORS_ALLOW_ORIGINS
  value: "https://sublime.phin3has.me"
- name: BASE_URL
  value: "https://sublime.phin3has.me"  # THE CRITICAL FIX!
- name: DASHBOARD_PUBLIC_BASE_URL
  value: "https://sublime.phin3has.me"
- name: API_PUBLIC_BASE_URL
  value: "https://sublime-api.phin3has.me"
```

## Step 7: The Ingress Dance

Set up two ingresses - one for the dashboard, one for the API:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sublime-api
  namespace: sublime
  annotations:
    external-dns.alpha.kubernetes.io/hostname: sublime-api.phin3has.me
    external-dns.alpha.kubernetes.io/target: YOUR-TUNNEL-ID.cfargotunnel.com
    # We'll add this later after setup
    # traefik.ingress.kubernetes.io/router.middlewares: traefik-authentik-forward-auth@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
  - host: sublime-api.phin3has.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sublime-mantis
            port:
              number: 8000
```

## Step 8: Initial Setup (Before SSO)

First, deploy everything WITHOUT authentication:

```bash
# Apply all your manifests
kubectl apply -f apps/sublime/

# Wait for everything to be ready
kubectl wait --for=condition=ready pod -l app=sublime-mantis -n sublime --timeout=300s

# Check the API health
curl https://sublime-api.phin3has.me/v1/health
```

Create your admin account through the API (the dashboard might have SSR issues initially).

## Step 9: The SSO Saga Begins

Now for the fun part - setting up SSO with Authentik. This is where I lost two days of my life.

### Setting up SAML in Authentik:

1. **Create a SAML Provider:**
   ```
   Name: Sublime SAML Provider
   ACS URL: https://sublime-api.phin3has.me/v0/organizations/YOUR-ORG-ID/saml
   Issuer: (same as ACS URL)
   Service Provider Binding: Post
   Audience: (same as ACS URL)
   Name ID Property: Email
   ```

2. **Create an Application:**
   ```
   Name: Sublime Platform
   Slug: sublime
   Provider: (select your SAML provider)
   ```

3. **Get the metadata URL** from Authentik

4. **Configure Sublime** (through the admin panel once you're logged in)

### The Plot Twist: It's Not SAML, It's the Redirect!

Here's what happened: Everything was configured correctly, but after authentication, Sublime was redirecting to `https://sublime-api.phin3has.me/login?error=SSO_ERROR` - which doesn't exist because the login page is on the dashboard domain!

The culprit? Sublime uses `BASE_URL` for post-authentication redirects, not `DASHBOARD_PUBLIC_BASE_URL` like you'd expect.

**The fix:**
```yaml
# In mantis deployment
- name: BASE_URL
  value: "https://sublime.phin3has.me"  # NOT the API URL!
```

### Alternative: Use OIDC Instead

After the SAML nightmare, I discovered OIDC works great too:

1. **Create an OAuth2/OpenID Provider in Authentik:**
   ```
   Client type: Confidential
   Redirect URIs: https://sublime-api.phin3has.me/v0/organizations/YOUR-ORG-ID/oidc/callback
   Scopes: openid, email, profile
   Subject mode: Based on the User's Email
   ```

2. **Configure in Sublime:**
   ```
   Issuer URL: https://auth.yourdomain.com/application/o/sublime/
   Client ID: (from Authentik)
   Client Secret: (from Authentik)
   ```

## Troubleshooting Tips

### "Connection refused" or "502 Bad Gateway"
- Check service names and ports
- Verify pods are actually running: `kubectl get pods -n sublime`
- Check logs: `kubectl logs -f deployment/sublime-mantis -n sublime`

### "SSO_ERROR" after authentication
- Check the `BASE_URL` environment variable
- Verify the user exists in Sublime with the same email
- Check time synchronization between servers

### Dashboard shows 500 error
- This is a known SSR issue with the Docker image
- The API still works fine
- Use port-forwarding for local access if needed

### Strelka connection issues
- Use just the hostname, not hostname:port
- Verify the coordinator is running
- Check Redis connectivity

## Lessons Learned

1. **Always check environment variables** - The `BASE_URL` issue cost me two days
2. **Start without authentication** - Get it working first, then add security
3. **Read the logs carefully** - The clues are usually there
4. **OIDC > SAML** - Simpler is often better
5. **Document everything** - Your future self will thank you

## The Victory Lap

After two days of debugging, multiple coffee pots, and questioning my life choices, Sublime Security now runs beautifully on Kubernetes with Authentik SSO. The platform is protecting my homelab email, and I can write custom detection rules to catch those pesky "Nigerian prince" emails.

## Full Deployment Files

All the Kubernetes manifests are available in my homelab repo. Here's the structure:

```
apps/sublime/
├── namespace.yaml
├── sealed-sublime-secrets.yaml
├── postgres-deployment.yaml
├── redis-deployment.yaml
├── minio-deployment.yaml
├── strelka-configs.yaml
├── strelka-deployment.yaml
├── mantis-deployment.yaml
├── bora-lite-deployment.yaml
├── dashboard-deployment.yaml
├── screenshot-deployment.yaml
├── hydra-deployment.yaml
└── ingress.yaml
```

## Final Thoughts

Is running Sublime Security on Kubernetes overkill for a homelab? Absolutely. 
Did I learn a ton about SAML, OIDC, and the importance of environment variables? You bet. 
Would I do it again? ...Ask me after I've recovered.

But here's the thing - this is what homelabbing is all about. Taking enterprise software, bending it to your will, and learning valuable skills in the process. Plus, now you have enterprise-grade email security running in your cluster. How cool is that?

Remember: Every 502 error is a learning opportunity, every misconfigured environment variable is a chance to grow, and every successful deployment is a victory worth celebrating.

Happy homelabbing, and may your SSO always redirect to the right domain! 🚀

---

*P.S. - If you're from Sublime Security and reading this: Please add official Kubernetes support. My sanity thanks you in advance.*