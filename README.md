# Airflow 3.x – Installation & Prerequisites (Dev Cluster on Kubernetes)

## 1. Purpose
This document describes the **prerequisites** and **installation steps** for deploying **Apache Airflow 3.x** as a **new, independent dev cluster** on Kubernetes, running side by side with the existing Airflow deployment.

The deployment uses:
- Airflow components:
  - `airflowsvr` (Webserver)
  - `airflowsch` (Scheduler)
  - `airflowwkr` (Worker)
- Supporting services:
  - Postgres (metadata DB)
  - Redis (Celery broker)
  - Dagsync (GitRunner)

## 2. Architecture Overview
Airflow 3.x runs as a **fully independent cluster**, including:
- Separate metadata database
- Separate Redis broker
- Independent configuration and environment variables
- New URL and API endpoints

No state, DB schema, or runtime components are shared with the existing Airflow deployment.

## 3. Prerequisites

### 3.1 Kubernetes Cluster Requirements
- Access to a Kubernetes cluster (dev environment recommended)
- `kubectl` configured with appropriate context
- Helm (optional, but commonly used for Postgres/Redis)

### 3.2 Postgres – Airflow Metadata Database
Postgres runs as a dedicated **Deployment + Service** (or StatefulSet) and hosts the Airflow metadata DB.

#### 3.2.1 Database & User Setup
Create a dedicated database and user for Airflow 3.x:
- **Database**: `airflow`
- **User**: `airflow`
- **Password**: (`AIRFLOW_POSTGRES_PASSWORD`)

#### 3.2.2 Required Permissions
The Airflow DB user must have:
- Full ownership of the `airflow` database
- Permissions to:
  - CREATE tables
  - ALTER tables
  - CREATE indexes
  - CREATE sequences
  - EXECUTE migrations

Recommended SQL (executed as postgres superuser):
- Create user
- Create database
- Grant ownership of database to `airflow`

#### 3.2.3 Connectivity
- Postgres Service must be reachable within the cluster (typically via service name, e.g. `postgres-service:5432`)
- Schema migrations are executed by Airflow during startup (`airflow db migrate`)

### 3.3 Redis – Celery Broker
Redis is used exclusively as the **Celery message broker**.

**Requirements**:
- Redis version: latest (2.8 protocol compatible)
- Authentication enabled

**Required Variables**:
- `REDIS_USER_NAME`
- `REDIS_USER_PASSWD`

Redis must be reachable at:
```
redis://<user>:<password>@redis-service:6379/0
```

### 3.4 Dagsync (GitRunner)
Dagsync is responsible for synchronizing DAGs into the Airflow environment.

**Requirements**:
- Deployment running and reachable via service name `gitrunner`
- Rsync enabled

**Required Variables**:
- `GITRUNNER_RSYNC=1`
- `GITRUNNER_RSYNC_SECRET`
- `PERSISTENT_STATE_DIRECTORY=/appz/home/airflow/.rsync_state`

### 3.5 Okta (Authentication Prerequisites)
Airflow 3.x integrates with Okta for authentication.

**Required Okta Setup**:
- Okta application created
- Redirect URI configured for Airflow webserver
- Client credentials

**Required Environment Variables (Webserver)**:
- `OKTA_API_BASE_URL`
- `OKTA_API_CLIENT_ID`
- `OKTA_API_CLIENT_SECRET`

### 3.6 Mandatory Airflow Secrets
The following **must exist before startup** (store in Kubernetes Secrets):
- `AIRFLOW__API_AUTH__JWT_SECRET`
- `AIRFLOW_ADMIN_PASSWORD`
- `AIRFLOW__CORE__FERNET_KEY` (static, consistent across all components)

## 4. Installation & Deployment Steps

### 4.1 Recommended Deployment Order
Deploy resources in this sequence to avoid dependency issues:
1. **Postgres** (Deployment/StatefulSet + Service + Secret)
2. **Redis** (Deployment/StatefulSet + Service + Secret)
3. **Dagsync (GitRunner)** (Deployment + Service)
4. **Airflow Webserver** (`airflowsvr`)
5. **Airflow Scheduler** (`airflowsch`)
6. **Airflow Workers** (`airflowwkr`)

### 4.2 Prepare Airflow Image
Build and push your Airflow image to a registry accessible by the cluster. The image should contain:
- Python 3.12+
- Airflow 3.x
- Required providers and shared dependencies

Reference this image in your Airflow Deployments.

### 4.3 Deploy Airflow Webserver (`airflowsvr`)
- Uses your prepared Airflow image
- Initializes:
  - Airflow metadata DB
  - Admin user
  - API services

Responsibilities:
- UI access (via Ingress or LoadBalancer)
- REST API
- Authentication (Okta)

### 4.4 Deploy Airflow Scheduler (`airflowsch`)
Connects to:
- Postgres
- Redis
- Webserver API

Responsibilities:
- DAG parsing
- Task scheduling
- Triggering Celery tasks

### 4.5 Deploy Airflow Workers (`airflowwkr`)
Connects to:
- Redis broker
- Postgres result backend
- Webserver API

Responsibilities:
- Task execution
- Log generation

## 5. Post-Installation Validation
After all components are running:
- Airflow UI is reachable via the configured Ingress / LoadBalancer URL
- Scheduler heartbeats are healthy (`kubectl logs` or UI)
- Workers are registered in the UI
- DAGs appear from Dagsync
- Test DAG executes successfully
