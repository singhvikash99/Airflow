# Setting Up Airflow and Django with AWS RDS Database on Docker

## Overview

Welcome to our comprehensive guide on setting up Airflow and Django within Docker containers, leveraging AWS RDS for database storage. By following this guide, you'll have a robust development environment ready to handle task scheduling with Airflow and web application development with Django, all neatly containerized for easy deployment and management.

## Vision: Orchestrating and Automating Data Retrieval
The vision behind this setup extends beyond mere containerization. With Airflow orchestrating the scheduling of tasks and Django serving as the application framework, our aim is to create a seamless environment for orchestrating and automating data retrieval from the Django database hosted on AWS RDS. Imagine a scenario where data needs to be fetched from the Django database every week(my tutorial shows for every 5 mins) for real-time analysis or reporting. This setup ensures that the entire process is orchestrated and automated, reducing manual intervention and increasing efficiency.

By containerizing Airflow and Django within Docker, we ensure portability and consistency across different environments. Leveraging AWS RDS for database storage adds scalability and reliability to our infrastructure, allowing us to focus on building and deploying our applications without worrying about database management.

## Prerequisites

1. **Before we dive into the setup process, make sure you have the following installed on your system:**
- Docker
- AWS account with RDS (Relational Database Service) access
2. **AWS RDS:**
- Create a RDS of MySQL

3. **Setting up Django with RDS MySQL**
- Configure you Django Database setting
![image](https://github.com/singhvikash99/Airflow/assets/19836202/95ac92ba-fd7d-4a7d-89c7-02b9851053f3)

4. **Create Dockerfile**
Create a file named Dockerfile in the root directory of your Django project. This file will contain instructions for building the Docker image for your Django application.
   ```terminal
   FROM python:3.12
   ENV PYTHONUNBUFFERED 1
   WORKDIR /code
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   RUN python manage.py migrate
5. **Create Docker-compose-yaml**
The Docker Compose file orchestrates containerization of both Django and Airflow applications, defining their services, dependencies, and configurations. This unified setup simplifies deployment, ensuring seamless integration and consistent environments for development, testing, and production, enhancing scalability and efficiency in managing complex application architectures.
   ```terminal
   version: '3'
    x-airflow-common:
      &airflow-common
      image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.0.2}
      environment:
        &airflow-common-env
        # AIRFLOW__CORE__EXECUTOR: CeleryExecutor
        AIRFLOW__CORE__SQL_ALCHEMY_CONN: sqlite:////usr/local/airflow/airflow.db
        AIRFLOW__CORE__FERNET_KEY: ''
        AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
        AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
        AIRFLOW__API__AUTH_BACKEND: 'airflow.api.auth.backend.basic_auth'
      volumes:
        - ./dags:/opt/airflow/dags
        - ./logs:/opt/airflow/logs
        - ./plugins:/opt/airflow/plugins
        - ./airflow_db:/usr/local/airflow   # Mount volume for SQLite database file
      user: "${AIRFLOW_UID:-50000}:${AIRFLOW_GID:-50000}"
    
    services:
      django-app:
        build:
          context: .
          dockerfile: Dockerfile
        command: >
          bash -c "python manage.py runserver 0.0.0.0:8000"
        image: django-app
        container_name: django-app
        ports:
          - '8000:8000'
        restart: always
    
      airflow-webserver:
        <<: *airflow-common
        command: webserver
        ports:
          - 8080:8080
        healthcheck:
          test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
          interval: 10s
          timeout: 10s
          retries: 5
        restart: always
    
      airflow-scheduler:
        <<: *airflow-common
        command: scheduler
        restart: always
    
      airflow-init:
        <<: *airflow-common
        command: version
        environment:
          <<: *airflow-common-env
          _AIRFLOW_DB_UPGRADE: 'true'
          _AIRFLOW_WWW_USER_CREATE: 'true'
          _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
          _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
          _AIRFLOW_WWW_USER_ROLE: ${_AIRFLOW_WWW_USER_ROLE:-Admin}  # Set the admin role

6. **Build the Docker Image**
   ```terminal
   docker-compose up --build
   
7. **Create a DAG**
With this setup there will created a dag folder in which you can keep you dag file like:
   ```terminal
    from airflow import DAG
    from airflow.operators.python_operator import PythonOperator
    from datetime import datetime, timedelta
    import mysql.connector
    import logging
    
    
    def fetch_data():
        """
        Function to fetch data from a MySQL database and log the fetched rows.
        """
        try:
            # Connect to the MySQL database
            conn = mysql.connector.connect(
                host="your_rds_endpoint",
                user="your_database_user",
                password="your_database_password",
                database="your_database_name"
            )
            cursor = conn.cursor()
    
            # Fetch data from the database table
            cursor.execute("SELECT * FROM myapp_mydata")
            rows = cursor.fetchall()
    
            # Process fetched data
            for row in rows:
                print(row)
                return logging.info(row)
    
        except Exception as e:
            print("Error occurred while fetching data:", str(e))
    
        finally:
            # Close cursor and connection
            cursor.close()
            conn.close()
    
    
    def print_task_complete():
        """
        Function to print a message indicating that all tasks are completed.
        """
        print("All tasks completed")
    
    
    # Define the default arguments for the DAG
    default_args = {
        'owner': 'airflow',
        'depends_on_past': False,
        'start_date': datetime(2024, 4, 28),
        'email_on_failure': False,
        'email_on_retry': False,
        'retries': 1
    }
    
    # Define the DAG
    dag = DAG(
        'fetch_data_and_print',
        default_args=default_args,
        description='A simple DAG to fetch data from a database table and print task complete',
        schedule_interval="*/5 * * * *",  # Run every 5 minutes
    )
    
    # Define the tasks
    fetch_data_task = PythonOperator(
        task_id='fetch_data_task',
        python_callable=fetch_data,
        dag=dag,
    )
    
    print_task_complete_task = PythonOperator(
        task_id='print_task_complete_task',
        python_callable=print_task_complete,
        dag=dag,
    )
    
    # Set task dependencies
    fetch_data_task >> print_task_complete_task

8. **Run Airflow and Django**
   ```terminal
   http://localhost:8000 #for Django
   http://localhost:8080 #for Airflow
9. **Airflow Credential**
With this setup the deafult username and password will be:
   ```terminal
   username: airflow
   password: airflow

10. **Airflow UI**
Aiflow UI will look like this:
![image](https://github.com/singhvikash99/Airflow/assets/19836202/da2e52b4-0d3c-4180-a742-aba061a90ee2)
You can check if task succesfully ran of failed:
![image](https://github.com/singhvikash99/Airflow/assets/19836202/f5da3c37-dd08-4410-b07e-60daeaf6f55f)
check logs by clicking on task_name graph and clicking on logs
![image](https://github.com/singhvikash99/Airflow/assets/19836202/bec5755a-2640-4a8e-a3c1-1839bd75c510)




