# Setup Guide

## Question 1 — Manual AWS Infrastructure (Console)

Do these in the AWS Console (I can drive the browser pane alongside you once you're logged in):

1. **VPC** → VPC Dashboard → Create VPC → name it, CIDR e.g. `10.0.0.0/16`.
2. **Public Subnet** → Subnets → Create subnet → pick the VPC, CIDR e.g. `10.0.1.0/24`.
3. **Internet Gateway** → Create IGW → Attach to the VPC.
4. **Route Table** → Create route table (or edit the main one) → Add route `0.0.0.0/0` → target the IGW → Associate with the public subnet.
5. **Security Group** → Create SG in the VPC → Inbound rules: SSH (22) from your IP, HTTP (80) from anywhere (0.0.0.0/0).
6. **EC2 instance** → Launch instance → Amazon Linux/Ubuntu, free-tier `t2.micro`/`t3.micro` → select the VPC + public subnet → enable "Auto-assign public IP" → attach the security group → create/select a key pair (download the `.pem`, keep it private).
7. **SSH in**: `ssh -i your-key.pem ec2-user@<public-ip>` (Amazon Linux) or `ubuntu@<public-ip>` (Ubuntu).
8. **Install NGINX**:
   - Amazon Linux: `sudo yum install -y nginx && sudo systemctl enable --now nginx`
   - Ubuntu: `sudo apt update && sudo apt install -y nginx && sudo systemctl enable --now nginx`
9. **Serve a page**: edit `/usr/share/nginx/html/index.html` (Amazon Linux) or `/var/www/html/index.html` (Ubuntu) with the sample content, then `sudo systemctl restart nginx`.
10. **Verify**: open `http://<EC2-public-ip>` in a browser.
11. **Target Group**: EC2 → Target Groups → Create → type Instances, protocol HTTP, port 80 → register the EC2 instance.
12. **ALB**: EC2 → Load Balancers → Create → Application Load Balancer → Internet-facing, IPv4 → select the VPC + public subnet(s) → attach a security group allowing port 80 inbound.
13. **Listener**: HTTP:80 → forward to the target group.
14. **Health check**: wait until the target shows **Healthy** in the Target Group.
15. **Access**: open `http://<ALB-DNS-name>`.

Take screenshots at each stage (IGW, VPC, subnet, route table) for the submission doc.

## Question 2 — CI/CD to S3 + CloudFront

### A. Create the S3 bucket
```bash
aws s3 mb s3://YOUR-UNIQUE-BUCKET-NAME --region ap-south-1
aws s3 website s3://YOUR-UNIQUE-BUCKET-NAME --index-document index.html
```

### B. Bucket policy (public read) — save as `bucket-policy.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-UNIQUE-BUCKET-NAME/*"
    }
  ]
}
```
Apply it (after disabling "Block all public access" on the bucket):
```bash
aws s3api put-bucket-policy --bucket YOUR-UNIQUE-BUCKET-NAME --policy file://bucket-policy.json
```

### C. Create a CloudFront distribution
- Origin: your S3 bucket (use the REST endpoint, e.g. `YOUR-UNIQUE-BUCKET-NAME.s3.ap-south-1.amazonaws.com`).
- Viewer protocol policy: Redirect HTTP to HTTPS.
- Default root object: `index.html`.
- Note the **Distribution ID** and the **Domain name** (`dxxxxxx.cloudfront.net`) once deployed.

### D. IAM user for GitHub Actions
Create an IAM user (or better, an OIDC role) with a policy limited to this bucket + this distribution:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::YOUR-UNIQUE-BUCKET-NAME",
        "arn:aws:s3:::YOUR-UNIQUE-BUCKET-NAME/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation"],
      "Resource": "*"
    }
  ]
}
```
Generate an access key for this user.

### E. GitHub repo secrets
In your GitHub repo → Settings → Secrets and variables → Actions, add:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

Then edit [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) and replace:
- `S3_BUCKET` with your bucket name
- `CLOUDFRONT_DISTRIBUTION_ID` with your distribution ID
- `AWS_REGION` if not `ap-south-1`

### F. Push and verify
```bash
git init
git add .
git commit -m "Initial DevOps training site"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git
git push -u origin main
```
Watch the workflow run under the repo's **Actions** tab. Once green, open `https://<distribution-domain>.cloudfront.net`.

To prove auto-redeploy: bump the version in `index.html`/`script.js`, commit, push again, and confirm the CloudFront URL updates (you may need to wait for the invalidation, ~1 min).
