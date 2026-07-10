# Lab : CI/CD with GitHub Actions → Ubuntu EC2
## Deploy a static website automatically on every push

---

## 🎯 What you will build

```
You push code to GitHub
        │
        ▼
[ GitHub Actions ]  ← triggered automatically
        │
        ├─ Copy files (SCP) ──→ EC2 /var/www/html/
        │
        └─ Reload Nginx ──────→ Site is live ✅
```

Every time you push to `main`, your site updates automatically — no manual steps.

---

## 🛠️ Prerequisites

- AWS account (free tier is enough)
- GitHub account
- Basic terminal knowledge

---

## PART 1 — Prepare the EC2 Server

---

### STEP 1 — Launch an EC2 Instance

1. Go to **AWS Console** → **EC2** → **Launch Instance**
2. Fill in:

   | Field | Value |
   |-------|-------|
   | **Name** | `cicd-web-server` |
   | **AMI** | Ubuntu Server 22.04 LTS |
   | **Instance type** | `t2.micro` (free tier) |
   | **Key pair** | Create new → name it `cicd-key` → Download `.pem` file |

3. In **Network settings**:
   - Allow **SSH** (port 22) — Source: My IP
   - Allow **HTTP** (port 80) — Source: Anywhere

4. Click **Launch Instance**

> ⚠️ Save the `.pem` file — you will need it for GitHub Actions

---

### STEP 2 — Install Nginx on EC2

Connect to your instance:

```bash
# Replace with your EC2 public IP
ssh -i cicd-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Then install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Test it — open `http://<EC2_PUBLIC_IP>` in your browser.
You should see the default Nginx page ✅

Fix permissions so GitHub Actions can write files:

```bash
sudo chown -R ubuntu:ubuntu /var/www/html
```

---

## PART 2 — Prepare the GitHub Repository

---

### STEP 3 — Create a GitHub Repository

1. Go to [github.com](https://github.com) → **New repository**
2. Name it `cicd-demo`
3. Set it to **Public**
4. Click **Create repository**

---

### STEP 4 — Add the Application Files

Your project structure:

```
cicd-demo/
├── index.html
├── style.css
└── .github/
    └── workflows/
        └── deploy.yml
```

**index.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>My CI/CD App</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div class="container">
    <h1>🚀 Hello from CI/CD!</h1>
    <p>This app was deployed automatically with <strong>GitHub Actions</strong>.</p>
    <div class="badge">Deployed on Ubuntu EC2</div>
  </div>
</body>
</html>
```

**style.css**
```css
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: Arial, sans-serif;
  background: #0f172a;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.container { text-align: center; color: #f1f5f9; }
h1 { font-size: 3rem; margin-bottom: 1rem; }
p { font-size: 1.2rem; color: #94a3b8; margin-bottom: 2rem; }

.badge {
  display: inline-block;
  background: #22c55e;
  color: #fff;
  padding: 0.5rem 1.5rem;
  border-radius: 999px;
  font-size: 0.9rem;
  font-weight: bold;
}
```

**.github/workflows/deploy.yml**
```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Fix permissions on EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: sudo chown -R $USER:$USER /var/www/html

      - name: Copy files to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "index.html,style.css"
          target: "/var/www/html/"

      - name: Reload Nginx
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: sudo systemctl reload nginx
```

---

### STEP 5 — Add GitHub Secrets

GitHub Actions needs your EC2 credentials — we store them as **Secrets** (never in the code).

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret** for each:

   | Secret Name | Value |
   |-------------|-------|
   | `EC2_HOST` | Your EC2 public IP (ex: `54.123.45.67`) |
   | `EC2_USER` | `ubuntu` |
   | `EC2_SSH_KEY` | Content of your `cicd-key.pem` file |

**To get the content of your .pem file:**
```bash
cat cicd-key.pem
```
Copy everything including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`

> ⚠️ Never commit the `.pem` file to GitHub — always use Secrets

---

## PART 3 — Trigger the Pipeline

---

### STEP 6 — Push Your Code

```bash
# Clone your repo locally
git clone https://github.com/<your-username>/cicd-demo.git
cd cicd-demo

# Add the files (index.html, style.css, .github/workflows/deploy.yml)
git add .
git commit -m "Initial deployment"
git push origin main
```

---

### STEP 7 — Watch the Pipeline Run

1. Go to your GitHub repository
2. Click on the **Actions** tab
3. You see the workflow **Deploy to EC2** running
4. Click on it to see each step in real time:
   - ✅ Checkout code
   - ✅ Copy files to EC2
   - ✅ Reload Nginx

> ⏱️ The whole pipeline takes about 30 seconds

---

### STEP 8 — See Your Site Live

Open your browser and go to:
```
http://<EC2_PUBLIC_IP>
```

You should see your page with **"🚀 Hello from CI/CD!"** 🎉

---

## PART 4 — Test the CI/CD Loop

This is the most important part — make a change and watch it deploy automatically.

### STEP 9 — Make a Change and Push

Edit `index.html` — change the title:

```html
<h1>🚀 Version 2 — Updated automatically!</h1>
```

Then push:

```bash
git add index.html
git commit -m "Update title to version 2"
git push origin main
```

1. Go to **Actions** tab on GitHub — a new pipeline starts automatically
2. Wait ~30 seconds
3. Refresh `http://<EC2_PUBLIC_IP>`

> ✅ Your change is live — without touching the server manually!

---

## 🔍 Understanding the Workflow File

```yaml
on:
  push:
    branches:
      - main          # ← triggers only on push to main branch
```

```yaml
runs-on: ubuntu-latest  # ← GitHub spins up a fresh Ubuntu VM for each run
```

```yaml
- name: Copy files to EC2
  uses: appleboy/scp-action@v0.1.7   # ← pre-built action that does SCP
  with:
    host: ${{ secrets.EC2_HOST }}     # ← reads from GitHub Secrets (safe)
    source: "index.html,style.css"    # ← files to copy
    target: "/var/www/html/"          # ← destination on EC2
```

```yaml
- name: Reload Nginx
  uses: appleboy/ssh-action@v1.0.0   # ← pre-built action that runs SSH commands
  with:
    script: sudo systemctl reload nginx  # ← command to run on EC2
```

---

## 🔍 Comprehension Questions

1. **What triggers the GitHub Actions workflow?**
   > *(Answer: A push to the `main` branch)*

2. **Why do we use GitHub Secrets instead of putting the IP directly in the YAML?**
   > *(Answer: Security — secrets are encrypted and never visible in logs or code)*

3. **What would happen if Nginx was not reloaded after copying files?**
   > *(Answer: The old files might still be served from cache — reload forces Nginx to pick up the new files)*

4. **What is `runs-on: ubuntu-latest`?**
   > *(Answer: GitHub creates a temporary Ubuntu VM to run the pipeline steps — it is destroyed after the job finishes)*

---

## 🧹 Cleanup

To avoid AWS charges:

1. Go to **EC2** → select `cicd-web-server` → **Instance State** → **Terminate**
2. Delete the key pair in **EC2** → **Key Pairs**

---

## ✅ Success Checklist

- [ ] EC2 instance running with Nginx installed
- [ ] GitHub repository created with `index.html`, `style.css`, `deploy.yml`
- [ ] 3 GitHub Secrets configured (`EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`)
- [ ] First push triggers the pipeline automatically
- [ ] Site accessible at `http://<EC2_PUBLIC_IP>`
- [ ] Second push (change) deploys automatically in ~30 seconds

---

## 📊 Summary

| Component | Role |
|-----------|------|
| **GitHub repository** | Stores the source code |
| **GitHub Actions** | Runs the pipeline on every push |
| **SCP Action** | Copies files from GitHub runner to EC2 |
| **SSH Action** | Runs commands on EC2 remotely |
| **Nginx** | Serves the HTML/CSS files on port 80 |
| **GitHub Secrets** | Stores sensitive credentials safely |
