# Airflow 3.x – Installation & Prerequisites (Dev Cluster)

## 1. Purpose

This document describes the **prerequisites** and **installation steps** for deploying **Apache Airflow 3.x** as a **new, independent dev cluster**, running side by side with the existing Airflow deployment.

The deployment is container-based and consists of:

* A custom **airflow-3 base image**
* Airflow components:

  * `airflowsvr` (Webserver)
  * `airflowsch` (Scheduler)
  * `airflowwkr` (Worker)
* Supporting services:

  * Postgres (metadata DB)
  * Redis (Celery broker)
  * Dagsync (GitRunner)

## 2. Architecture Overview

Airflow 3.x runs as a **fully independent cluster**, including:

* Separate metadata database
* Separate Redis broker
* Independent configuration and environment variables
* New URL and API endpoints

No state, DB schema, or runtime components are shared with the existing Airflow deployment.

## 3. Prerequisites

### 3.1 Container & Runtime Requirements

* Docker / container runtime installed
* Ability to build custom Docker images
* Internal Docker network allowing container-to-container DNS resolution via links

### 3.2 Postgres – Airflow Metadata Database

Postgres runs as a **dedicated container** and hosts the Airflow metadata DB.

#### 3.3.1 Database & User Setup

Create a dedicated database and user for Airflow 3.x:

* **Database**: `airflow`
* **User**: `airflow`
* **Password**: (`AIRFLOW_POSTGRES_PASSWORD`)

#### 3.3.2 Required Permissions

The Airflow DB user must have:

* Full ownership of the `airflow` database
* Permissions to:

  * CREATE tables
  * ALTER tables
  * CREATE indexes
  * CREATE sequences
  * EXECUTE migrations

Recommended SQL (executed as postgres superuser):

* Create user
* Create database
* Grant ownership of database to `airflow`

#### 3.3.3 Connectivity

* Postgres must be reachable as `postgres:5432`
* Schema migrations are executed by Airflow during startup (`airflow db migrate`)

### 3.4 Redis – Celery Broker

Redis is used exclusively as the **Celery message broker**.

**Requirements**:

* Redis version: latest (2.8 protocol compatible)
* Authentication enabled

**Required Variables**:

* `REDIS_USER_NAME`
* `REDIS_USER_PASSWD`

Redis must be reachable at:

```
redis://<user>:<password>@redis:6379/0
```

### 3.5 Dagsync (GitRunner)

Dagsync is responsible for synchronizing DAGs into the Airflow environment.

**Requirements**:

* Container running and reachable as `gitrunner`
* Rsync enabled

**Required Variables**:

* `GITRUNNER_RSYNC=1`
* `GITRUNNER_RSYNC_SECRET`
* `PRESISTENT_STATE_DIRECTORY=/appz/home/airflow/.rsync_state`

### 3.6 Okta (Authentication Prerequisites)

Airflow 3.x integrates with Okta for authentication.

**Required Okta Setup**:

* Okta application created
* Redirect URI configured for Airflow webserver
* Client credentials

**Required Environment Variables (Webserver)**:

* `OKTA_API_BASE_URL`
* `OKTA_API_CLIENT_ID`
* `OKTA_API_CLIENT_SECRET`

### 3.7 Mandatory Airflow Secrets

The following **must exist before startup**:

* `AIRFLOW__API_AUTH__JWT_SECRET`
* `AIRFLOW_ADMIN_PASSWORD` 
* `AIRFLOW__CORE__FERNET_KEY` (static, consistent across all components)

## 4. Installation & Deployment Steps

### 4.1 Container Startup Order (MANDATORY)

Containers **must be started in the following order**:

1. **Postgres**
2. **Redis**
3. **Dagsync (GitRunner)**
4. **airflow-3 (Base Image – build only)**
5. **airflowsvr**
6. **airflowsch**
7. **airflowwkr**

### 4.2 Build Airflow 3 Base Image

* Build the custom `airflow-3` base image
* This image contains:

  * Python 3.12+
  * Airflow 3.x
  * Providers and shared dependencies

 The base image is **not run as a container**.

### 4.3 Start Airflow Webserver (airflowsvr)

* Uses the `airflow-3` base image
* Initializes:

  * Airflow metadata DB
  * Admin user
  * API services

Responsibilities:

* UI access
* REST API
* Authentication (Okta)

### 4.4 Start Airflow Scheduler (airflowsch)

* Connects to:

  * Postgres
  * Redis
  * Webserver API

Responsibilities:

* DAG parsing
* Task scheduling
* Triggering Celery tasks

### 4.5 Start Airflow Worker (airflowwkr)

* Connects to:

  * Redis broker
  * Postgres result backend
  * Webserver API

Responsibilities:

* Task execution
* Log generation

## 5. Post-Installation Validation

After all containers are running:

* Airflow UI is reachable via the new URL
* Scheduler heartbeats are healthy
* Workers are registered
* DAGs appear from Dagsync
* Test DAG executes successfully
