# TruckPilot — Landing Page

Marketing landing page for TruckPilot, hosted on Netlify for free.

## Deploy to Netlify (free, ~3 minutes)

### Option A — Drag & Drop (fastest)

1. Go to [netlify.com](https://netlify.com) and create a free account
2. From the dashboard, drag the entire `Hub_Transport_Landing/` folder onto the deploy area
3. Done — you get a live URL like `https://random-name.netlify.app`

### Option B — Git (recommended, auto-deploy on push)

1. Create a new GitHub repository (public or private)
2. Push this folder:
   ```bash
   git init
   git add .
   git commit -m "feat: initial landing page"
   git remote add origin https://github.com/YOUR_USER/hub-transport-landing.git
   git push -u origin main
   ```
3. In Netlify: **Add new site → Import from Git → select the repo**
4. Build settings: leave blank (static HTML, no build step)
5. Every push to `main` triggers a new deploy automatically

### Custom domain (optional, free on Netlify)

In Netlify: **Site settings → Domain management → Add custom domain**

---

## Form submissions

Demo request form uses **Netlify Forms** (built-in, free tier = 100 submissions/month).

Submissions appear in: **Netlify dashboard → Forms → demo-request**

To get email notifications: **Forms → demo-request → Form notifications → Add email**

---

## Editing content

All content is in `index.html`. Key sections:

| Section | Search for |
|---------|-----------|
| Hero headline | `Your fleet's` |
| Stats bar | `&lt;30s` |
| Regulatory cards | `id="compliance"` |
| How it works | `id="how-it-works"` |
| Feature cards | `id="features"` |
| Demo form | `id="demo"` |
| Footer | `&lt;footer` |
