# DevOps Fundamentals — Practical Assignment

Two-part assignment: manually provisioned AWS infrastructure (Question 1) and a CI/CD pipeline deploying a static site to S3 via GitHub Actions (Question 2).

## Question 1 — AWS Infrastructure (Manual, Console)

A VPC with a public subnet, internet gateway, route table, security group, and an EC2 instance running NGINX behind an Application Load Balancer.

| Resource | Value |
|---|---|
| Region | `ap-south-1` (Mumbai) |
| VPC | `devops-assignment-vpc` (`10.0.0.0/16`) |
| Public subnets | `devops-assignment-public-subnet` (`10.0.1.0/24`, ap-south-1a), `devops-assignment-public-subnet-2` (`10.0.2.0/24`, ap-south-1b) |
| Internet Gateway | `devops-assignment-igw` |
| Security Group | `devops-assignment-sg` (SSH 22 from my IP, HTTP 80 from anywhere) |
| EC2 instance | `devops-assignment-ec2` (Amazon Linux, NGINX) |
| Target Group | `devops-assignment-tg` |
| Load Balancer | `devops-assignment-alb` |

**Live URL**: http://devops-assignment-alb-1806509737.ap-south-1.elb.amazonaws.com

Full step-by-step build log is in [SETUP.md](SETUP.md).

## Question 2 — CI/CD to S3

Static site (`index.html`, `style.css`, `script.js`) auto-deployed to S3 on every push to `main` via [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml): `test` → `build` → `deploy`.

| Resource | Value |
|---|---|
| S3 bucket | `devops-assignment-preetsinghwal` (static website hosting, public read) |
| Region | `ap-south-1` |

**Live URL**: http://devops-assignment-preetsinghwal.s3-website.ap-south-1.amazonaws.com

### How it works

1. Push to `main`.
2. `test` job verifies `index.html`, `style.css`, `script.js` exist.
3. `build` job packages them into a `dist/` artifact.
4. `deploy` job authenticates to AWS (IAM user `github-actions-deployer`, scoped to this bucket) and syncs `dist/` to the S3 bucket.

Auto-redeploy verified by bumping the version number in `index.html`/`script.js`, pushing, and confirming the live URL updates without any manual upload.

Full setup steps (bucket policy, IAM policy, GitHub secrets) are in [SETUP.md](SETUP.md).
