# Goal & Architecture
 
* S3 (private) stores your static site.
* CloudFront serves it over HTTPS with caching. Access to S3 is locked down via Origin Access Control (OAC).
* (Optional) Route 53 + ACM for your own domain.
* GitHub Actions deploys on every push.

# Names we’ll use (feel free to keep them)
 
* Project alias: devopslearner
* S3 bucket: p1-devopslearner-static-ap-south-1(must be globally unique; append a random suffix if needed)
* CloudFront OAC: oac-p1-devopslearner
* CloudFront distribution comment: p1-devopslearner-static-site
* GitHub repo: p1-static-website
* IAM user for CI: gh-actions-p1-static
* (Optional) Domain used in examples: devopslearner.in (replace with yours)
* Region: ap-south-1 (except ACM for CloudFront: us-east-1)
 
# Part A — Local repo & site code
 
## 1) Create repo & scaffold
 
bash
# Local folder
mkdir p1-static-website && cd p1-static-website
git init
git branch -M main
 
#Repo structure
 

p1-static-website/
├─ site/
│  ├─ index.html
│  ├─ error.html
│  ├─ assets/
│  │  └─ style.css
├─ deploy.sh
├─ .gitignore
└─ .github/
   └─ workflows/
      └─ deploy.yml

 
.gitignore
 
.DS_Store
node_modules
dist
.env

##site/index.html
 
html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>DevOpsLearner | Static Site</title>
  <link rel="stylesheet" href="assets/style.css" />
</head>
<body>
  <header>
    <h1>🚀 DevOpsLearner — Static Site on S3 + CloudFront</h1>
    <p>Built with Git, Linux, AWS. Deployed via GitHub Actions.</p>
  </header>
  <main>
    <section class="card">
      <h2>Stack</h2>
      <ul>
        <li>Amazon S3 (private) + CloudFront (OAC)</li>
        <li>Optional: Route 53 + ACM (HTTPS on your domain)</li>
        <li>CI/CD: GitHub Actions</li>
      </ul>
    </section>
    <section class="card">
      <h2>Change me!</h2>
      <p>Make a commit and watch the pipeline deploy automatically.</p>
    </section>
  </main>
  <footer>© <span id="y"></span> DevOpsLearner</footer>
  <script>document.getElementById('y').textContent=new Date().getFullYear()</script>
</body>
</html>

##site/error.html
 
html
<!doctype html>
<html><head><meta charset="utf-8"><title>Oops!</title>
<link rel="stylesheet" href="assets/style.css" /></head>
<body><main class="card"><h1>404 / Error</h1><p>Something went wrong.</p></main></body></html>

##site/assets/style.css
 
css
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, "Helvetica Neue", Arial; }
body { margin: 0; background: #0f172a; color: #e2e8f0; }
header { padding: 48px 24px; text-align: center; background: #111827; }
h1 { margin: 0 0 8px; }
main { max-width: 900px; margin: 24px auto; padding: 0 16px; }
.card { background: #111827; padding: 20px; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,.3); margin-bottom: 16px; }
footer { text-align: center; padding: 24px; opacity: .7; }
a { color: #93c5fd; }

Commit:
 
bash
git add .
git commit -m "feat: initial static site"

# Part B — AWS setup (S3 + CloudFront with OAC)
 
## 2) Create S3 bucket (private)
 
Console
 
1. S3 → Create bucket
2. Name: p1-devopslearner-static-ap-south-1
3. Region: ap-south-1
4. Block all public access: ON (leave checked)
5. Enable Bucket versioning: Enabled (recommended)
6. Create bucket.
 
CLI
 
bash
aws s3api create-bucket \
  --bucket p1-devopslearner-static-ap-south-1 \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1
 
aws s3api put-bucket-versioning \
  --bucket p1-devopslearner-static-ap-south-1 \
  --versioning-configuration Status=Enabled

 
> We’re not enabling “Static website hosting” on S3 because CloudFront will use the REST endpoint with OAC.
 
## 3) Create CloudFront distribution with OAC
 
Console (recommended)
 
1. Go to CloudFront → Create distribution.
2. Origin
 
   * Origin domain: pick your S3 bucket (p1-devopslearner-static-ap-south-1.s3.ap-south-1.amazonaws.com) from dropdown.
   * Origin access: *Origin access control settings (recommended)* → Create control setting
 
     * Name: oac-p1-devopslearner
     * Signing behavior: Sign requests
     * Save.
3. Default cache behavior
 
   * Viewer protocol policy: Redirect HTTP to HTTPS
   * Allowed HTTP methods: GET, HEAD
   * Cache policy: CachingOptimized (default is fine)
   * Enable Compress objects automatically
4. Settings
 
   * Default root object: index.html
   * Comment: p1-devopslearner-static-site
   * (If you have a domain + cert already, add Alternate domain names here; you can also add later.)
5. Create distribution. Note the Distribution ID (e.g., E1234567890ABC) and Domain name (e.g., dxxxxx.cloudfront.net).
 
Attach the generated bucket policy for OAC
 
1. After creation, CloudFront shows a message to update your S3 bucket policy for OAC.
2. Open S3 → Bucket → Permissions → Bucket policy → Edit. Paste this (replace ACCOUNT\_ID and DISTRIBUTION\_ID):
 
json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServiceGetObjectOAC",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::p1-devopslearner-static-ap-south-1/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
 
> Replace:
 * <ACCOUNT_ID> → your 12-digit AWS account ID.
 * <DISTRIBUTION_ID> → the ID you noted.
 
(Optional) allow ListBucket too (not strictly required for CloudFront to read objects):
 
json
{
  "Sid": "AllowCloudFrontListBucketOAC",
  "Effect": "Allow",
  "Principal": { "Service": "cloudfront.amazonaws.com" },
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::p1-devopslearner-static-ap-south-1",
  "Condition": {
    "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  }
}

Save the policy.
 
# Part C — First manual upload & test
 
## 4) Upload the site to S3 (one-time manual test)
 
CLI (from repo root)
 
bash

# Upload everything under ./site to the bucket
aws s3 sync ./site s3://p1-devopslearner-static-ap-south-1 --delete

 
Set sensible Cache-Control (optional)
 
bash
# Example: short cache for HTML, longer for assets
aws s3 cp site/index.html s3://p1-devopslearner-static-ap-south-1/index.html --cache-control "max-age=300,public" --content-type "text/html"
aws s3 cp site/error.html s3://p1-devopslearner-static-ap-south-1/error.html --cache-control "max-age=300,public" --content-type "text/html"
aws s3 cp site/assets/style.css s3://p1-devopslearner-static-ap-south-1/assets/style.css --cache-control "max-age=31536000,public" --content-type "text/css"

 
## 5) Create a CloudFront invalidation (to see fresh content)
 
bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths "/*"

 
## 6) Test
 
* Open the CloudFront domain (like https://dxxxxx.cloudfront.net/) → you should see your site.
 
---
 
# Part D — (Optional) Custom domain + HTTPS
 
> Skip if you don’t have a domain yet. (Note: Route 53 hosted zone costs \~₹50/month; domain registration costs extra.)
 
## 7) Request an ACM certificate (must be in us-east-1 for CloudFront)
 
Console
 
1. Switch region to N. Virginia (us-east-1).
2. ACM → Request certificate → Request a public certificate.
3. Add names: devopslearner.in and www.devopslearner.in (replace with yours).
4. DNS validation. ACM shows CNAME records to add.
 
## 8) Route 53 hosted zone + DNS
 
1. Route 53 → Hosted zones → Create hosted zone → devopslearner.in → Public hosted zone.
2. Add the validation CNAME records ACM gave you (choose “Create record in Route 53” if offered).
3. In your registrar, set the domain’s nameservers to the NS records of this hosted zone (if you purchased the domain in Route 53, this is automatic).
4. Wait for ACM to show Issued (usually minutes).
 
## 9) Attach domain to CloudFront
 
1. CloudFront → Your distribution → Settings → Edit
2. Alternate domain names (CNAMEs): add devopslearner.in and www.devopslearner.in.
3. Custom SSL certificate: select the ACM cert you created in us-east-1.
4. Save.
 
## 10) Route 53 alias records to CloudFront
 
1. In Hosted zone, create:
 
   * A record (alias) for devopslearner.in → Alias to CloudFront distribution.
   * CNAME or A (alias) for www → Alias to same CloudFront distribution.
2. Test https://devopslearner.in.
 
---
 
# Part E — CI/CD with GitHub Actions
 
## 11) Create an IAM user for CI (simple path: access keys)
 
> In production you’d use GitHub OIDC (no long-lived keys). To keep this project simple, we’ll use an IAM user.
 
Console
 
1. IAM → Users → Create user: gh-actions-p1-static
2. Attach inline policy below (replace account/bucket/distribution IDs).
3. After creating, go to Security credentials → Create access key for Application running outside AWS. Copy Access key ID and Secret access key.
 
Inline policy for CI
 
json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3WriteToSiteBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::p1-devopslearner-static-ap-south-1"
    },
    {
      "Sid": "S3ObjectCRUD",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::p1-devopslearner-static-ap-south-1/*"
    },
    {
      "Sid": "InvalidateCloudFront",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  ]
}

 
12) Add GitHub Secrets
 
In your GitHub repo → Settings → Secrets and variables → Actions → New repository secret:
 
AWS_ACCESS_KEY_ID → (from step 11)
 
AWS_SECRET_ACCESS_KEY → (from step 11)
 
(Optional) DISTRIBUTION_ID → your CloudFront ID
 
(Optional) AWS_REGION → ap-south-1
 
(Optional) S3_BUCKET → p1-devopslearner-static-ap-south-1
 
13) GitHub Actions workflow file
 
Create .github/workflows/deploy.yml
name: Deploy static site
 
on:
  push:
    branches: [ "main" ]
    paths:
      - "site/"
      - ".github/workflows/deploy.yml"
 
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
 
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION || 'ap-south-1' }}
 
      - name: Sync site to S3
        run: |
          aws s3 sync site s3://${{ secrets.S3_BUCKET || 'p1-devopslearner-static-ap-south-1' }} --delete
 
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.DISTRIBUTION_ID }} \
            --paths "/*"
 Commit & push:
 
git remote add origin https://github.com/<your-username>/p1-static-website.git
git push -u origin main
 
 
Check Actions tab → watch deploy → open your CloudFront/domain URL and verify changes.

Part F — Handy local deploy script (optional)
 
deploy.sh
#!/usr/bin/env bash
set -euo pipefail
REGION="${REGION:-ap-south-1}"
BUCKET="${BUCKET:-p1-devopslearner-static-ap-south-1}"
DISTRIBUTION_ID="${DISTRIBUTION_ID:-<PUT_YOUR_DISTRIBUTION_ID>}"
 
echo "Syncing ./site → s3://$BUCKET ..."
aws s3 sync ./site "s3://$BUCKET" --delete
 
echo "Invalidating CloudFront distribution $DISTRIBUTION_ID ..."
aws cloudfront create-invalidation --distribution-id "$DISTRIBUTION_ID" --paths "/*"
 
echo "Done."

chmod +x deploy.sh
./deploy.sh
