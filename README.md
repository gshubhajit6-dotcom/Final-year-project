# Deploying a Flask Web App on AWS Elastic Beanstalk

Beautiful, pragmatic guide to package, configure, and deploy a Flask application on AWS Elastic Beanstalk (EB). This README covers local setup, EB configuration, environment variables, common production concerns, monitoring, and an example GitHub Actions workflow for CI/CD.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Repository layout](#repository-layout)
- [Local development](#local-development)
- [Preparing the app for Elastic Beanstalk](#preparing-the-app-for-elastic-beanstalk)
- [Deploying with the EB CLI](#deploying-with-the-eb-cli)
- [Environment variables & secrets](#environment-variables--secrets)
- [Data persistence (databases)](#data-persistence-databases)
- [Scaling, health, and monitoring](#scaling-health-and-monitoring)
- [Blue/Green and rolling deployments](#bluegreen-and-rolling-deployments)
- [Logs & troubleshooting](#logs--troubleshooting)
- [CI/CD example (GitHub Actions)](#cicd-example-github-actions)
- [Security & cost considerations](#security--cost-considerations)
- [Contributing](#contributing)
- [License & Contacts](#license--contacts)

---

## Project Overview

This repository demonstrates how to deploy a production-ready Flask application to AWS Elastic Beanstalk using the EB CLI. It assumes your Flask app is structured in a standard way (an `app` or `application` object exposed) and uses Gunicorn as the WSGI server.

Key goals:
- Simple, reproducible deployment steps
- Clear separation of config, secrets, and code
- Guidance for scaling, logging, and monitoring

---

## Prerequisites

- Python 3.8+ (match the runtime you want to use on EB)
- git
- AWS account with permissions to use Elastic Beanstalk, EC2, IAM, and optionally RDS
- AWS CLI: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html
- EB CLI: https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html
- Recommended: virtualenv or pipenv

Configure AWS CLI:
```bash
aws configure
# Provide AWS Access Key ID, Secret Access Key, region (e.g. us-east-1), and default output format
```

Install EB CLI (example with pip):
```bash
pip install awsebcli --upgrade
```

---

## Repository layout

A suggested simple layout:

```
.
├── app.py                # or wsgi.py / your Flask entrypoint
├── requirements.txt
├── runtime.txt           # python version for EB
├── Procfile              # command to start your app
├── .ebextensions/        # optional EB config files
├── .elasticbeanstalk/    # created by EB CLI (project config)
├── templates/
└── static/
```

Example `Procfile`:
```text
web: gunicorn app:app --bind 0.0.0.0:$PORT
```
Adjust `app:app` to `your_module:application` if different.

Example `runtime.txt` (pick a supported runtime):
```text
python-3.9
```

---

## Local development

Create a virtual environment and install dev dependencies:
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Run your app locally:
```bash
export FLASK_APP=app.py
export FLASK_ENV=development
flask run
# or with gunicorn for a production-like environment:
gunicorn app:app
```

Create `requirements.txt` from your environment:
```bash
pip freeze > requirements.txt
```

---

## Preparing the app for Elastic Beanstalk

1. Ensure a WSGI entry point exists (e.g., `app.py` exposing `app` or `application`).
2. Use Gunicorn in production. Add it to `requirements.txt`.
3. Add a `Procfile` to instruct EB how to run the app (see above).
4. Add `runtime.txt` to pin Python version.
5. Create a `.gitignore` and commit your code.

Optional: `.ebextensions` for additional EB-level configuration. Example to set a system package or environment variable defaults:

```
.ebextensions/01_packages.config
option_settings:
  aws:elasticbeanstalk:application:environment:
    EXAMPLE_SETTING: "default-value"
packages:
  yum:
    git: []
```

Note: Many environment variables should be set in EB configuration rather than source-controlled files.

---

## Deploying with the EB CLI

Initialize the EB project (run once per repo):

```bash
eb init -p python-3.9 my-flask-app
# Follow prompts: region, application name, SSH key (optional)
```

Create an environment and deploy (first deployment):
```bash
eb create my-flask-env --single
# or for a load-balanced environment:
eb create my-flask-env --elb-type application
```

Deploy new versions:
```bash
git add .
git commit -m "Deploying new version"
eb deploy my-flask-env
```

Check environment health and status:
```bash
eb status
eb health
eb events
```

Terminate environment:
```bash
eb terminate my-flask-env
```

---

## Environment variables & secrets

Set secrets in the console or with EB CLI to avoid committing secrets:

Using EB CLI:
```bash
eb setenv FLASK_ENV=production SECRET_KEY="your-secret" DATABASE_URL="postgres://..."
```

You can also set them per-environment via the AWS Console: Elastic Beanstalk > Configuration > Software.

For more advanced secret management, consider using:
- AWS Systems Manager Parameter Store (SSM)
- AWS Secrets Manager
- Storing only references in EB environment and fetching secrets at runtime

---

## Data persistence (databases)

Elastic Beanstalk instances are ephemeral — don't store important data on instance disk.

Recommended approaches:
- Use RDS (managed PostgreSQL/MySQL) and set DATABASE_URL in environment variables.
- Or use managed services like Amazon DynamoDB or S3 for file storage.

If creating RDS inside an EB environment, consider lifecycle and backups: creating RDS from EB console ties the DB lifecycle to the EB environment (which may be deleted accidentally). Often better to create RDS separately.

Example using SQLAlchemy and DATABASE_URL:
```python
from sqlalchemy import create_engine
import os

engine = create_engine(os.getenv('DATABASE_URL'))
```

---

## Scaling, health, and monitoring

- Use an Application Load Balancer (ALB) for production.
- Configure autoscaling based on CPU or request latency.
- Set appropriate health checks and timeouts (inside EB configuration).
- Use CloudWatch for logs, metrics, and alarms.
- Consider enabling enhanced health reporting in EB.

---

## Blue/Green and rolling deployments

Elastic Beanstalk supports several deployment policies:
- All at once (fast, downtime)
- Rolling
- Rolling with additional batch
- Immutable (safer, creates full new instances)
- Blue/Green (create a separate environment, test, then swap CNAMEs)

For zero-downtime, prefer Rolling or Immutable or Blue/Green. Use `eb clone` to create a blue environment, test it, then swap CNAMEs.

---

## Logs & troubleshooting

Fetch logs:
```bash
eb logs
eb logs --zip   # gets a zip of logs
```

SSH into instances (if you enabled an EC2 key pair):
```bash
eb ssh
# then check /var/log/nginx/error.log, /var/log/web.stdout.log, etc.
```

Common issues:
- Port binding: EB expects the app to bind to $PORT environment variable; use Gunicorn and the `%PORT%` or `$PORT`.
- Missing dependencies: ensure requirements.txt includes everything.
- Wrong Procfile or WSGI callable: verify `Procfile` matches `module:callable`.

---

## CI/CD example (GitHub Actions)

Simple workflow to deploy on push to `main` (requires EB CLI and configured AWS credentials as secrets):

```yaml
name: Deploy to Elastic Beanstalk
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install awsebcli
          pip install -r requirements.txt

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy to Elastic Beanstalk
        run: |
          eb init -p python-3.9 my-flask-app --region us-east-1
          eb use my-flask-env
          eb deploy --staged
```

Store AWS credentials in GitHub Secrets (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) and consider using IAM roles for GitHub Actions or OIDC trust for better security.

---

## Security & cost considerations

Security:
- Never commit secrets. Use EB environment properties, SSM, or Secrets Manager.
- Use an IAM user/role with least privileges for CI/CD.
- Use HTTPS/ALB and configure security groups properly.
- Keep dependencies updated and monitor CVEs.

Cost:
- EB farms EC2 instances, load balancers, and optional RDS — monitor and size carefully.
- Use smaller instance types for dev and reserved instances for long-running production if appropriate.

---

## Troubleshooting checklist

- App works locally but not on EB:
  - Check `Procfile`, `runtime.txt`, and `requirements.txt`.
  - Confirm WSGI callable name.
  - Fetch EB logs (`eb logs`) and EC2 instance logs.
- Deployment fails due to permissions:
  - Verify AWS credentials and IAM permissions.
- Database connectivity issues:
  - Verify security groups, DB URL, and that the DB is publicly accessible or accessible from EB VPC.

---

## Contributing

PRs welcome. For changes to deployment docs, include:
- Clear explanation of the motivation
- Repro steps (if adding a new config)
- Tests or verification steps where applicable

---

## License & Contacts

- License: choose a license (e.g., MIT) and add a LICENSE file.
- Maintainer: your name / contact / repo owner

---

Thanks for using this guide — if you'd like I can:
- Customize this README to your repo structure (point to your app entrypoint),
- Create the necessary EB config files (.ebextensions),
- Or push this file to a branch and open a PR.