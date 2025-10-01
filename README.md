# airflow-local-dev

Airflow Docker configuration for local development

---

## Project Structure

This repository provides a ready-to-use local Airflow environment using Docker Compose, with custom configuration and dependencies.

### Key Files and Directories

- **Dockerfile**  
  Customizes the Airflow image with additional dependencies and Java support.

- **docker-compose.yaml**  
  Defines the multi-container Airflow stack (scheduler, API server, DAG processor, triggerer, PostgreSQL, etc).

- **requirements.txt**  
  Python dependencies and Airflow providers to be installed in the custom image.

- **config/airflow.cfg**  
  Main Airflow configuration file, mounted into the container.  
  **This file has been customized—see below for details.**

- **.env**  
  Environment variables for Docker Compose (e.g., `AIRFLOW_UID`).

---

## How to Use

### 1. Build the Custom Airflow Image

The `Dockerfile` is used to build a custom Airflow image with extra dependencies and Java support.

**File:** `Dockerfile`  
**Location:** Project root

**Key contents:**
```dockerfile
FROM apache/airflow:3.1.0
USER root
COPY requirements.txt /
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
         openjdk-17-jre-headless \
  && apt-get autoremove -yqq --purge \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*
USER airflow
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
RUN pip install --no-cache-dir "apache-airflow==3.1.0" -r /requirements.txt
```
*This installs Java 17 and all Python requirements.*

---

### 2. Configure Environment Variables

**File:** `.env`  
**Location:** Project root

**Example:**
```env
AIRFLOW_UID=1000
```
*Set the user ID for file permissions inside containers.*

---

### 3. Add Python Dependencies

**File:** `requirements.txt`  
**Location:** Project root

**Example:**
```text
pandas
sqlalchemy
psycopg2-binary
lxml
beautifulsoup4
apache-airflow-providers-postgres
apache-airflow-providers-http
apache-airflow-providers-ftp
apache-airflow-providers-github
```
*Add or remove dependencies as needed for your DAGs and operators.*

---

### 4. Airflow Configuration

**File:** `config/airflow.cfg`  
**Location:** `config/` directory

This file is a full Airflow configuration and has been customized for this project.  
**If you need to make further changes, edit this file directly.**

#### Notable Customizations

Below are the main changes made to the default `airflow.cfg` for this local development setup:

```ini
[core]
executor = LocalExecutor
dags_folder = /opt/airflow/dags
plugins_folder = /opt/airflow/plugins
simple_auth_manager_users = admin:admin
auth_manager = airflow.api_fastapi.auth.managers.simple.simple_auth_manager.SimpleAuthManager
parallelism = 32
max_active_tasks_per_dag = 16
max_active_runs_per_dag = 16
load_examples = True

[database]
sql_alchemy_conn = sqlite:////opt/airflow/airflow.db

[logging]
base_log_folder = /opt/airflow/logs
logging_level = INFO
colored_console_log = True

[dag_processor]
dag_bundle_config_list = [
  {
    "name": "git-dags",
    "classpath": "airflow.providers.git.bundles.git.GitDagBundle",
    "kwargs": {
      "tracking_ref": "main",
      "git_conn_id": "local-dags-test",
      "repo_url": "https://github.com/<you_account>/<your_repository>.git"
    }
  },
  {
    "name": "dags-folder",
    "classpath": "airflow.dag_processing.bundles.local.LocalDagBundle",
    "kwargs": {}
  }
]
```

**Key changes and why:**
- **LocalExecutor** is used for parallel task execution in local development.
- **SQLite** is set as the metadata database for simplicity (not for production).
- **Simple Auth Manager** is enabled with a default admin user (`admin:admin`) for easy access.
- **Parallelism and concurrency** values are increased for better local performance.
- **DAG bundle config** includes a Git-based DAG source and the local `dags/` folder.
- **Logging** is set to INFO level and logs are stored in `/opt/airflow/logs`.
- **Fernet key** is set for connection encryption.

**To change any of these settings:**  
Open `config/airflow.cfg`, search for the relevant section and key, and edit the value.  
For example, to use a different executor or database, change the `executor` or `sql_alchemy_conn` values.

---

### 5. Docker Compose Setup

**File:** `docker-compose.yaml`  
**Location:** Project root

- Uses the custom image built from the `Dockerfile`.
- Mounts local `dags/`, `logs/`, `plugins/`, and `config/` directories into the containers.
- Sets up all Airflow services and a PostgreSQL database.
- The `airflow-init` service initializes the environment and creates the admin user.

**Key changes to note:**
- The `build: .` line uses your custom Dockerfile.
- The `AIRFLOW_CONFIG` environment variable points to the mounted config file.
- The `postgres` service is configured for Airflow metadata storage.

---

## Running the Environment

1. **Build and start the stack:**
   ```sh
   docker compose up --build
   ```
   or, if using legacy Compose:
   ```sh
   docker-compose up --build
   ```

2. **Access the Airflow UI:**  
   Open [http://localhost:8080](http://localhost:8080) in your browser.  
   Default credentials:  
   - Username: `airflow`
   - Password: `airflow`

3. **Place your DAGs:**  
   Add your DAG files to the `dags/` directory.

---

## Customization

- **To add more Python dependencies:**  
  Edit `requirements.txt` and rebuild the image.

- **To change Airflow settings:**  
  Edit `config/airflow.cfg` and restart the containers.

- **To change the Airflow version:**  
  Update the `FROM apache/airflow:3.1.0` line in the `Dockerfile` and the `AIRFLOW_IMAGE_NAME` in `docker-compose.yaml` if needed.

---

## Notes

- This setup is for local development only.  
- Do **not** use in production.
- For more details, see the official [Airflow Docker Compose documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html).

---
