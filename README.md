# weatherapi-etl-pipeline
Automated Weather Data ETL Pipeline Using Apache Airflow, Python, and AWS S3


## Introduction

In today's data-driven world, the ability to extract, transform, and load (ETL) data efficiently is crucial for informed decision-making. This project demonstrates an ETL pipeline for weather data using Apache Airflow, Python, and AWS. The pipeline extracts real-time weather data from an API, transforms the data into a structured format, and loads it into an Amazon S3 bucket for further analysis and use.


## Objective

The objective of this project is to create an automated ETL pipeline that fetches weather data from an API, transforms it into a user-friendly format, and stores it in AWS S3 for further analysis. The pipeline will be scheduled to run daily, ensuring that the data remains up-to-date.


## Tools and Technologies

Apache Airflow: To orchestrate and schedule the ETL workflow.
Python: For data extraction, transformation, and loading.
AWS S3: For storing the transformed data.
Pandas: For data manipulation and transformation.
OpenWeatherMap API: For extracting real-time weather data.


## Methodology

4.1. Setting Up the Environment

1. Installing Apache Airflow
To install Apache Airflow, follow these steps:

1 - Create a Python virtual environment:

```python
python3 -m venv airflow_env
source airflow_env/bin/activate
```


2- Install Apache Airflow using pip:

```python
pip install apache-airflow

```


3- Initialize the Airflow database:

```python
airflow db init

```

4 - Start the Airflow web server and scheduler:

```python
airflow webserver --port 8080
airflow scheduler
```


2. Configuring AWS S3
Create an S3 bucket in your AWS account.
Generate AWS access keys (Access Key ID and Secret Access Key) for programmatic access to the S3 bucket.
3. ETL Pipeline Design
The ETL pipeline will consist of the following tasks:



4.2. ETL Pipeline Design
1.	Extract:
o	Use the weather API to fetch daily data for a city
o	Schedule the data extraction using an Airflow DAG (Directed Acyclic Graph).
2.	Transform:
o	Clean and preprocess the extracted data.
o	Convert the data into a suitable format for analysis (e.g., CSV, JSON).
3.	Load:
o	Upload the transformed data to the AWS S3 bucket.
o	Ensure data is correctly stored and accessible in the S3 bucket.

HttpSensor: To check the availability of the weather API.
SimpleHttpOperator: To extract weather data from the API.
PythonOperator: To transform and load the extracted data into an S3 bucket.
4. Implementation
Importing Libraries

```python
import datetime
from airflow import DAG
from datetime import timedelta, datetime
from airflow.providers.http.sensors.http import HttpSensor
import json
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.operators.python import PythonOperator
import pandas as pd

```


Helper Functions


```python

def kelvin_to_fahrenheit(temp_in_kelvin):
    temp_in_fahrenheit = (temp_in_kelvin - 273.15) * (9/5) + 32
    return temp_in_fahrenheit

def transform_load_data(task_instance):
    data = task_instance.xcom_pull(task_ids="extract_weather_data")
    city = data["name"]
    weather_description = data["weather"][0]['description']
    temp_farenheit = kelvin_to_fahrenheit(data["main"]["temp"])
    feels_like_farenheit = kelvin_to_fahrenheit(data["main"]["feels_like"])
    min_temp_farenheit = kelvin_to_fahrenheit(data["main"]["temp_min"])
    max_temp_farenheit = kelvin_to_fahrenheit(data["main"]["temp_max"])
    pressure = data["main"]["pressure"]
    humidity = data["main"]["humidity"]
    wind_speed = data["wind"]["speed"]
    
    time_of_record = datetime.fromtimestamp(data['dt'] + data['timezone'])
    sunrise_time = datetime.fromtimestamp(data['sys']['sunrise'] + data['timezone'])
    sunset_time = datetime.fromtimestamp(data['sys']['sunset'] + data['timezone'])

    transformed_data = {"City": city,
                        "Description": weather_description,
                        "Temperature (F)": temp_farenheit,
                        "Feels Like (F)": feels_like_farenheit,
                        "Minimun Temp (F)": min_temp_farenheit,
                        "Maximum Temp (F)": max_temp_farenheit,
                        "Pressure": pressure,
                        "Humidty": humidity,
                        "Wind Speed": wind_speed,
                        "Time of Record": time_of_record,
                        "Sunrise (Local Time)":sunrise_time,
                        "Sunset (Local Time)": sunset_time                        
                        }
    
    transformed_data_list = [transformed_data]
    df_data = pd.DataFrame(transformed_data_list)

    aws_credentials = {"key": "", "secret": "", "token": ""}

    now = datetime.now()
    dt_string = now.strftime("%d%m%Y%H%M%S")

    filename = f"s3://bucketname/{dt_string}.csv"
    df_data.to_csv(filename, index=False, storage_options=aws_credentials)
    print(f"Data saved to {filename}")



```



DAG Definition

```python
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 8),
    'email': ['myemail@domain.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=2)
}

with DAG('weather_dag',
        default_args=default_args,
        schedule_interval='@daily',
        catchup=False) as dag:
    
    is_weather_api_ready = HttpSensor(
        task_id='is_weather_api_ready',
        http_conn_id='weathermap_api',
        endpoint='/data/2.5/weather?q=Portland&APPID='
    )

    extract_weather_data = SimpleHttpOperator(
        task_id='extract_weather_data',
        http_conn_id='weathermap_api',
        endpoint='/data/2.5/weather?q=Portland&APPID=',
        method='GET',
        response_filter=lambda r: json.loads(r.text),
        log_response=True
    )

    transform_load_weather_data = PythonOperator(
        task_id='transform_load_weather_data',
        python_callable=transform_load_data
    )

    is_weather_api_ready >> extract_weather_data >> transform_load_weather_data

```


Discussion
Data Extraction
The pipeline begins by checking the availability of the weather API using an HttpSensor. If the API is available, the SimpleHttpOperator extracts weather data for a specified city (e.g., Portland).

Data Transformation
The extracted data is then processed in the transform_load_data function. This involves converting temperatures from Kelvin to Fahrenheit and structuring the data into a pandas DataFrame.

Data Loading
The transformed data is saved as a CSV file in an Amazon S3 bucket. The filename is timestamped to ensure uniqueness.

Automation
The entire pipeline is scheduled to run daily, ensuring that the latest weather data is continuously collected, transformed, and stored.

Conclusion
This project showcases the integration of Apache Airflow, Python, and AWS for building an automated ETL pipeline. By leveraging these technologies, we can efficiently process and store real-time weather data, providing a foundation for further analysis and application development.

Future Work
Error Handling: Implement robust error handling to manage potential issues during data extraction and loading.
Scalability: Explore the use of AWS Lambda for serverless computing to enhance scalability.
Data Analysis: Develop additional scripts to analyze the stored weather data and generate insights.
This ETL pipeline serves as a versatile and extendable framework, adaptable to various data sources and analytical requirements.

