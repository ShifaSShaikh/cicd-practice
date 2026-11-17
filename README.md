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

The initial `Jenkinsfile` checks out this repository, runs tests in a temporary Python container, and builds a Docker image. Publishing to Amazon ECR and deploying to EC2 will be added in later stages.
