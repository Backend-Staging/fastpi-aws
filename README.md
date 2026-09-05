# fastapi-aws

FastAPI application configured for AWS Lambda behind API Gateway HTTP API.

## Architecture

- AWS Lambda runs the FastAPI application.
- Mangum translates API Gateway events into ASGI requests.
- AWS SAM defines and deploys the Lambda function and HTTP API.
- PostgreSQL remains external and is configured with `DB_URL`.

## Prerequisites

Install and configure:

- AWS CLI v2
- AWS SAM CLI
- Docker Desktop (for a Lambda-compatible local build)
- AWS credentials with permission to deploy CloudFormation, Lambda, API Gateway, IAM roles, and CloudWatch Logs

Verify:

```bash
aws sts get-caller-identity
sam --version
docker --version
```

## Environment values

Prepare these values without committing them:

- `DatabaseUrl`: SQLAlchemy PostgreSQL URL, such as `postgresql://USER:PASSWORD@HOST:5432/DBNAME`
- `AllowedOrigin`: exact frontend URL, such as `https://app.example.com`
- `AuthSecretKey`: random secret with at least 32 characters
- `AuthAlgorithm`: `HS256` unless you deliberately use another supported HMAC algorithm

Generate a development signing secret:

```bash
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

## Run locally as a normal FastAPI server

Create and activate a virtual environment, then install dependencies:

```bash
python3.13 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Create a local `.env` file (already ignored by Git), then run:

```bash
uvicorn api.main:app --reload
```

Open `http://127.0.0.1:8000/docs`.

## Build and test the Lambda package

Build inside a Lambda-compatible container because `psycopg2-binary` includes native code:

```bash
sam validate --lint
sam build --use-container
```

To run the API locally through SAM, create `env.local.json` (do not commit it):

```json
{
  "FastApiFunction": {
    "DB_URL": "postgresql://USER:PASSWORD@HOST:5432/DBNAME",
    "API_URL": "http://localhost:3000",
    "AUTH_SECRET_KEY": "replace-with-a-long-random-development-secret",
    "AUTH_ALGORITHM": "HS256",
    "DEPLOYMENT_ENVIRONMENT": "AWS"
  }
}
```

Then run:

```bash
sam local start-api --env-vars env.local.json
curl http://127.0.0.1:3000/
```

Expected response:

```json
{"Healthy": 200}
```

## Deploy to AWS

The first deployment is guided:

```bash
sam build --use-container
sam deploy --guided
```

Recommended answers:

- Stack name: `fastapi-aws-dev`
- AWS Region: `us-east-1` (or the same region as your database)
- Confirm changes before deploy: `Y`
- Allow SAM CLI IAM role creation: `Y`
- Save arguments to configuration file: `Y`

SAM will prompt for the template parameters. After deployment, copy the `ApiUrl` stack output and test it:

```bash
curl "$(aws cloudformation describe-stacks \
  --stack-name fastapi-aws-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text)/"
```

## Remove the test deployment

```bash
sam delete --stack-name fastapi-aws-dev
```

## Production notes

This is suitable for a low-traffic test deployment, with these constraints:

- The current application creates database tables during Lambda cold start. Replace this with Alembic migrations before production.
- Do not expose an RDS database publicly just to let Lambda connect. For private RDS, add Lambda VPC subnets and a security group; consider RDS Proxy when concurrency grows.
- CloudFormation `NoEcho` masks parameter display, but Lambda environment variables are not a full secrets-management design. Move the database password and JWT key to AWS Secrets Manager before production.
- Protect or remove `POST /populate/`; it inserts data and must not remain publicly callable in production.
- Set an API Gateway authorizer, throttling, alarms, and a Lambda concurrency limit before public launch.
