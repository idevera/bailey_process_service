# Bailey Process Service - Website

Static website for **Bailey Process Service**, a locally owned small business (est. 2016) providing legal courier and process serving throughout Western Washington.

**Live site:** https://baileyprocessservice.com

---

## Architecture Overview

```
GitHub Repo (source of truth)
    |
    v
AWS S3 Bucket (baileyprocessservice.com)
    |
    v
AWS CloudFront (CDN + SSL)
    |
    v
Route 53 DNS --> User's browser
```

- **GitHub** — stores the source files and version history
- **S3** — hosts the static files (HTML, CSS, images)
- **CloudFront** — serves the site globally via CDN with HTTPS/SSL
- **Route 53** — manages DNS, points the domain to CloudFront
- **ACM (AWS Certificate Manager)** — provides the free SSL certificate

### Files in this repo

| File | Purpose |
|------|---------|
| `index.html` | Main website page (all content, SEO meta tags, structured data) |
| `styles.css` | Stylesheet |
| `favicon.ico` | Browser tab icon (32x32) |
| `icon-192.png` | Mobile/PWA icon (192x192) |
| `og-image.png` | Social media link preview image (1200x630) |
| `robots.txt` | Search engine crawling rules |
| `sitemap.xml` | Search engine sitemap |

---

## Deployment Guide

### Step 1: Push changes to GitHub

After editing files locally, push to GitHub:

```bash
cd /path/to/bailey_process_service
git add -A
git commit -m "Describe your changes here"
git push origin master
```

If you cloned the repo fresh:

```bash
git clone https://github.com/idevera/bailey_process_service.git
cd bailey_process_service
# make your changes
git add -A
git commit -m "Describe your changes here"
git push origin master
```

### Step 2: Upload files to S3

You can upload via the **AWS CLI** or the **AWS Console**.

**Option A — AWS CLI (recommended):**

```bash
aws s3 sync . s3://baileyprocessservice.com/ --exclude ".git/*" --exclude "README.md"
```

This syncs all files to the S3 bucket, skipping the `.git` folder and this README.

> First-time setup: run `aws configure` and enter your AWS Access Key ID, Secret Access Key, and region.

**Option B — AWS Console (manual):**

1. Go to **S3** in the AWS Console
2. Open the **baileyprocessservice.com** bucket
3. Click **Upload**
4. Drag and drop all files (except `.git/` and `README.md`)
5. Click **Upload**

### Step 3: Invalidate the CloudFront cache

After uploading to S3, CloudFront may still serve old cached files. You need to invalidate the cache.

**Option A — AWS CLI:**

```bash
aws cloudfront create-invalidation --distribution-id YOUR_DISTRIBUTION_ID --paths "/*"
```

> Replace `YOUR_DISTRIBUTION_ID` with the actual distribution ID (found in CloudFront console).

**Option B — AWS Console:**

1. Go to **CloudFront** in the AWS Console
2. Click on your distribution
3. Go to the **Invalidations** tab
4. Click **Create invalidation**
5. Enter `/*` to clear everything
6. Click **Create invalidation**

The invalidation typically completes in 1-5 minutes. You can also invalidate specific files instead of `/*` if you only changed a few things (e.g., `/index.html`, `/og-image.png`).

> The first 1,000 invalidation paths per month are free. A wildcard (`/*`) counts as 1 path.

---

## SSL Certificate Setup (ACM + Route 53)

The SSL certificate is managed through **AWS Certificate Manager (ACM)**. It must cover both:

- `baileyprocessservice.com`
- `www.baileyprocessservice.com`

### Verifying the certificate

1. Go to **ACM** in the AWS Console (make sure you're in **us-east-1 / N. Virginia** — CloudFront requires certificates in this region)
2. Find the certificate for `baileyprocessservice.com`
3. Confirm:
   - **Status** = Issued
   - **Domains** include both `baileyprocessservice.com` and `www.baileyprocessservice.com`
   - **Renewal status** = Success

### If the certificate needs renewal or re-creation

1. In **ACM**, click **Request a certificate**
2. Choose **Public certificate**
3. Add both domain names:
   - `baileyprocessservice.com`
   - `www.baileyprocessservice.com`
4. Choose **DNS validation**
5. Click **Create records in Route 53** (ACM will add CNAME records automatically)
6. Wait for status to change to **Issued** (usually a few minutes)

---

## DNS Setup (Route 53)

Route 53 must point both the root domain and `www` subdomain to the CloudFront distribution.

### Required records in the `baileyprocessservice.com` hosted zone

| Record Name | Type | Alias | Points To |
|-------------|------|-------|-----------|
| `baileyprocessservice.com` | A | Yes | CloudFront distribution |
| `www.baileyprocessservice.com` | A | Yes | CloudFront distribution |

Plus the CNAME records created by ACM for certificate validation (do not delete these — they're needed for auto-renewal).

### How to add/verify records

1. Go to **Route 53** > **Hosted zones** > `baileyprocessservice.com`
2. Check that both A records exist and point to the CloudFront distribution
3. If a record is missing:
   - Click **Create record**
   - **Record name:** leave blank for root, or enter `www`
   - **Record type:** A
   - **Alias:** toggle ON
   - **Route traffic to:** Alias to CloudFront distribution
   - Select your distribution from the dropdown
   - Click **Create records**

### CloudFront alternate domain names

The CloudFront distribution must also list both domains. To verify:

1. Go to **CloudFront** > your distribution > **General** tab > **Settings**
2. Under **Alternate domain names (CNAMEs)**, confirm both are listed:
   - `baileyprocessservice.com`
   - `www.baileyprocessservice.com`
3. Under **Custom SSL certificate**, confirm it's using the ACM certificate
4. If either is missing, click **Edit** and add them

---

## HTTP to HTTPS Redirect

All HTTP traffic should redirect to HTTPS. This is configured in CloudFront.

### How to verify/set up

1. Go to **CloudFront** > your distribution
2. Click the **Behaviors** tab
3. Select the default behavior and click **Edit**
4. Under **Viewer protocol policy**, select **Redirect HTTP to HTTPS**
5. Save changes

### Testing

After setup, all of these should load the site over HTTPS:

| URL | Expected behavior |
|-----|-------------------|
| `https://baileyprocessservice.com` | Loads site (direct) |
| `https://www.baileyprocessservice.com` | Loads site (direct) |
| `http://baileyprocessservice.com` | Redirects to `https://` |
| `http://www.baileyprocessservice.com` | Redirects to `https://` |

---

## Quick Reference - Full Deployment Checklist

When making changes to the site:

- [ ] Edit files locally
- [ ] Push to GitHub (`git add -A && git commit -m "message" && git push origin master`)
- [ ] Upload to S3 (`aws s3 sync` or drag-and-drop in console)
- [ ] Invalidate CloudFront cache (`/*`)
- [ ] Verify changes on the live site (use incognito window to avoid browser cache)

When setting up from scratch or fixing infrastructure:

- [ ] ACM certificate covers both `baileyprocessservice.com` and `www.baileyprocessservice.com`
- [ ] ACM certificate is in **us-east-1** region
- [ ] CloudFront distribution lists both domains as alternate CNAMEs
- [ ] CloudFront uses the ACM certificate (not default CloudFront cert)
- [ ] CloudFront behavior redirects HTTP to HTTPS
- [ ] Route 53 has A record (alias) for root domain pointing to CloudFront
- [ ] Route 53 has A record (alias) for `www` pointing to CloudFront
- [ ] ACM validation CNAME records exist in Route 53 (for auto-renewal)
