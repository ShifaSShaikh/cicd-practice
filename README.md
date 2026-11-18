# CI/CD Practice

A small Flask application used to learn an AWS-based Jenkins CI/CD pipeline.

## Endpoints

- `/` — returns application information as JSON
- `/health` — returns application health as JSON

## Run locally with Docker

```bash
docker build -t cicd-demo:local .
docker run --rm -p 8080:8080 cicd-demo:local
```

Then open `http://localhost:8080/health`.

## Run tests

```bash
pip install -r requirements-dev.txt
pytest -q
```

## Jenkins

The pipeline checks out this repository, runs tests, builds a Docker image, pushes it to Amazon ECR, and deploys it to the Application EC2 instance.

## Webhook automation test

This commit verifies that a GitHub push event automatically triggers the Jenkins CI/CD pipeline.
