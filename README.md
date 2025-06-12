# 🚀 NASA APOD ETL Pipeline with Apache Airflow, PostgreSQL & AWS Deployment

## 🧠 Project Overview

This project builds a modern ETL (Extract, Transform, Load) pipeline using **Apache Airflow**, **PostgreSQL**, **Docker**, and **AWS**. It automates the retrieval of data from NASA’s Astronomy Picture of the Day (APOD) API, transforms the data, and loads it into a PostgreSQL database.

The pipeline is orchestrated with Apache Airflow and containerized using Docker. It can be deployed both locally and on AWS infrastructure using Amazon EC2 for scalable, cloud-based data processing.

---

## 🧩 Key Components

- **Apache Airflow** – Orchestration tool for task scheduling and dependency management.
- **NASA APOD API** – Provides daily astronomy content.
- **PostgreSQL** – Serves as the persistent data store.
- **Docker Compose** – Container management tool for local development and testing.
- **AWS EC2** – Used to deploy the Dockerized application in a cloud environment.

---

## 🎯 Objectives

1. **Extract**: Fetch data from NASA’s APOD API on a daily schedule.
2. **Transform**: Parse and clean the API response.
3. **Load**: Insert structured data into a PostgreSQL database.
4. **Deploy**: Run the pipeline both locally and in the cloud using AWS.

---

## 🏗️ Architecture & Workflow

The workflow is defined via an Airflow DAG (`nasa_apod_dag.py`) consisting of:

- Table Creation
- Data Extraction via `HttpHook`
- Data Transformation
- Data Loading into PostgreSQL
---
## Output
Below is a screenshot from Airflow’s web UI showing a successful run of the DAG (Directed Acyclic Graph). Each task has executed without error, indicating that the end-to-end pipeline — from fetching data to storing the results — completed as expected.

![Screenshot 2025-06-10 212147](https://github.com/user-attachments/assets/51b49ae7-ada3-49bd-b99d-0dbcb8f40f3d)

This DAG ensures:

- Daily execution of the pipeline

- Automatic retries upon failure

- Modularity for easy integration with cloud storage or databases in future version

- The output image and data can be viewed or used for further analysis, visualization, or archival purposes.




### DAG: `dags/nasa_apod_dag.py`

```bash
from airflow.decorators import dag, task
from airflow.providers.http.hooks.http import HttpHook
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime

@dag(
    dag_id="nasa_apod_postgres",
    schedule="@daily",
    start_date=datetime(2025, 6, 11),
    catchup=False,
    tags=["example"]
)
def nasa_apod_dag():

    @task()
    def create_table():
        postgres_hook = PostgresHook(postgres_conn_id="my_postgres_connection")
        postgres_hook.run("""
            CREATE TABLE IF NOT EXISTS apod_data (
                id SERIAL PRIMARY KEY,
                title VARCHAR(255),
                explanation TEXT,
                url TEXT,
                date DATE,
                media_type VARCHAR(50)
            );
        """)

    @task()
    def extract_apod():
        hook = HttpHook(method="GET", http_conn_id="nasa_api")
        response = hook.run(
            endpoint="planetary/apod",
            data={"api_key": hook.get_connection("nasa_api").extra_dejson.get("api_key")}
        )
        return response.json()

    @task()
    def transform_apod_data(response):
        return {
            'title': response.get('title', ''),
            'explanation': response.get('explanation', ''),
            'url': response.get('url', ''),
            'date': response.get('date', ''),
            'media_type': response.get('media_type', '')
        }

    @task()
    def load_data_to_postgres(apod_data):
        postgres_hook = PostgresHook(postgres_conn_id='my_postgres_connection')
        insert_query = """
            INSERT INTO apod_data (title, explanation, url, date, media_type)
            VALUES (%s, %s, %s, %s, %s);
        """
        postgres_hook.run(insert_query, parameters=(
            apod_data['title'],
            apod_data['explanation'],
            apod_data['url'],
            apod_data['date'],
            apod_data['media_type']
        ))

    table = create_table()
    raw_data = extract_apod()
    transformed = transform_apod_data(raw_data)
    load = load_data_to_postgres(transformed)

    table >> raw_data >> transformed >> load

dag = nasa_apod_dag()
```

---

## 🐳 Docker Compose Setup

### `docker-compose.yml`

```bash
version: "3"
services:
  postgres:
    image: postgres:13
    container_name: postgres_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - airflow_network

networks:
  airflow_network:
    external: false

volumes:
  postgres_data:
```

---

## 🧪 Getting Started

### ✅ Prerequisites
- Docker & Docker Compose installed.

- NASA API key [get one here](https://api.nasa.gov/).

**🛠️ Steps**
** 1. Clone the repository

```bash
git clone https://github.com/jcboy101/NASA-APOD-ETL-Pipeline.git
cd CD NASA-APOD-ETL-Pipeline
```
**2. Configure Airflow connections**

Inside Airflow UI (at localhost:8080 after setup):

- Create an HTTP Connection with ID nasa_api and set the api_key in the "Extra" field as:

```json
{ "api_key": "YOUR_NASA_API_KEY" }
```
- Create a Postgres Connection with ID my_postgres_connection.

**3. Start services**

```bash
docker-compose up -d
```
**4.Run the DAG**

- Access Airflow UI at http://localhost:8080.

-Trigger the nasa_apod_postgres DAG manually or wait for the scheduled run.



## 🌐 AWS Deployment Steps

1. **Launch EC2 Instance**: Use an Amazon Linux 2 or Ubuntu instance with ports 22, 5432, and 8080 open.
2. **Install Docker & Docker Compose**:
   ```bash
   sudo apt update && sudo apt install docker.io docker-compose -y
   sudo usermod -aG docker $USER
   ```
3. **Clone Repo & Set Up Project**:
   ```bash
   git clone https://github.com/your-username/nasa-airflow-etl.git
   cd nasa-airflow-etl
   docker-compose up -d
   ```
4. **Access Airflow UI**: Visit `http://<EC2-PUBLIC-IP>:8080` from your browser.

---

## 🧪 Example Output

A sample inserted row in Postgres:

| title | explanation | url | date | media_type |
|-------|-------------|-----|------|------------|
| Solar Flare | A strong flare erupting from... | https://apod.nasa.gov/... | 2025-06-10 | image |

---

Happy Data Engineering! 
