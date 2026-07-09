# CI/CD Pipeline for a Static Website

A DevOps intern project demonstrating an automated CI/CD pipeline that builds and deploys a static website to AWS, using GitHub Actions, S3, CloudFront, and OIDC-based authentication.

**Live site:** https://cicd-demo.qossim005.online

---

## Overview

This project takes a simple static HTML/CSS site from a `git push` to a live, secured, monitored production URL - fully automatically, with no manual deployment steps.

**Objective:** Set up a basic CI/CD pipeline using GitHub Actions for automatically deploying a static website to AWS S3.

## Architecture

```
Developer
   │  git push (main)
   ▼
GitHub Repository
   │  triggers on push
   ▼
GitHub Actions Workflow
   │  1. Checkout code
   │  2. Assume AWS IAM role via OIDC (no stored credentials)
   │  3. Sanity check (abort if index.html missing/empty)
   │  4. aws s3 sync → S3 bucket
   ▼
S3 Bucket (private - Block Public Access enabled)
   │  origin, accessed only via CloudFront OAC
   ▼
CloudFront Distribution
   │  HTTPS via ACM certificate
   │  custom domain: cicd-demo.qossim005.online
   ▼
End User (HTTPS)
```

**Monitoring:** CloudFront standard access logs → dedicated S3 logs bucket, plus a CloudWatch alarm on `5xxErrorRate` with email notification via SNS.

## Tech Stack

| Layer | Technology |
|---|---|
| Version control | Git & GitHub |
| CI/CD | GitHub Actions |
| Hosting | AWS S3 (static assets) |
| CDN / HTTPS | AWS CloudFront + ACM |
| DNS | AWS Route 53 (subdomain-delegated) |
| Auth (CI → AWS) | OIDC federation (no long-lived keys) |
| Monitoring | CloudFront access logs + CloudWatch alarms |
| Scripting | Bash / YAML |

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml       # CI/CD workflow definition
├── index.html                # Site markup
├── style.css                 # Site styling
└── README.md                 # This file
```

## How Deployment Works

Every push to `main` triggers `.github/workflows/deploy.yml`, which:

1. Checks out the repository
2. Authenticates to AWS by assuming an IAM role via GitHub's OIDC provider - no access keys are stored anywhere
3. Runs a sanity check to abort the deploy if `index.html` is missing or empty
4. Syncs the repository contents to the S3 bucket with `aws s3 sync --delete`, so removed files are cleaned up in S3 too

Typical deploy time: **under 10 seconds** from push to live in S3.

## Security Design

- **No long-lived AWS credentials.** GitHub Actions assumes a short-lived IAM role per run via OIDC (`AssumeRoleWithWebIdentity`), scoped to this repository and the `main` branch only.
- **Least-privilege IAM policy** - the assumed role can only `PutObject` / `DeleteObject` / `ListBucket` on this project's specific S3 bucket.
- **Private S3 bucket.** Public access is fully blocked; the bucket is only reachable via CloudFront using Origin Access Control (OAC).
- **HTTPS enforced** via an AWS Certificate Manager certificate attached to the CloudFront distribution.

## Reliability & Rollback

- **S3 Versioning** is enabled - any previous version of any deployed file can be restored directly from S3.
- **Git revert** is the primary rollback method - reverting a bad commit and pushing redeploys the previous known-good state automatically.
- **Pre-deploy sanity check** in the workflow prevents obviously broken builds (e.g. an empty `index.html`) from ever reaching S3.

## Monitoring

- **CloudFront standard access logs** delivered to a dedicated S3 logs bucket (kept separate from the site content bucket).
- **CloudWatch metrics** (automatic, no setup required): `Requests`, `4xxErrorRate`, `5xxErrorRate`, `TotalErrorRate`, `BytesDownloaded`, `BytesUploaded`.
- **CloudWatch Alarm** on `5xxErrorRate` (> 5%, 5-minute period) sends an email notification via SNS if the site starts erroring.

## Setup Summary (for anyone replicating this)

1. Create an S3 bucket, enable versioning, block public access
2. Request an ACM certificate (`us-east-1`) for your domain/subdomain, validate via Route 53
3. Create a CloudFront distribution with Origin Access Control pointed at the S3 bucket, attach the certificate and custom domain
4. Update the S3 bucket policy to allow only the CloudFront distribution (via OAC)
5. Register an OIDC identity provider (`token.actions.githubusercontent.com`) in IAM
6. Create an IAM role with a trust policy scoped to your repo/branch, and a least-privilege S3 permissions policy
7. Add `AWS_ROLE_ARN`, `AWS_S3_BUCKET`, `AWS_REGION` as GitHub Secrets
8. Add `.github/workflows/deploy.yml` (see this repo) and push to `main`

## Full Project Documentation

A complete day-by-day technical walkthrough - including every command run, every issue hit, and how each was resolved - is maintained separately as the project's internship deliverable: `https://app.notion.com/p/CICD-Static-Site-393604d0a68980bd8a96e92400ebc05b`.

## Outcome

An automated pipeline that builds and deploys a website, giving hands-on insight into version control, CI/CD concepts, and real AWS DevOps practices - including IAM/OIDC authentication, CDN configuration, HTTPS, and monitoring.