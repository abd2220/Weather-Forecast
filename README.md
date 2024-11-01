# Weather Forecast Data Streaming Project using Big Data tools

## Project Overview
This project demonstrates real-time weather forecast data streaming as part of a Big Data Engineering pipeline. Weather data is ingested, processed, and visualized to provide actionable insights. The project simulates real-world data processing scenarios, focusing on data engineering tools and techniques suitable for high-volume data streams.

>Note: This pipeline was designed in CentOS 6.5.

## Project Objectives
- Stream real-time weather forecast data to a Kafka topic.
- Process the streaming data using PySpark to transform it for storage and analysis.
- Use InfluxDB for time-series data storage and HDFS for scalable storage.
- Visualize processed data in Grafana to monitor forecast trends.

## Key Components
- **Data Source**: [Weather forecast data](https://www.weather.gov/documentation/services-web-api) streamed using Kafka.
- **Message Broker**: Apache Kafka for ingesting and streaming data.
- **Data Processing**: PySpark transforms the raw forecast data and prepares it for storage.
- **Storage**:
  - **InfluxDB** for time-series data storage and analysis.
  - **HDFS** for scalable storage of processed data using Apache Flume to transfer from Kafka.
- **Visualization**: Grafana for monitoring forecast trends.

## Architecture Diagram
> *(Optional) Add an architecture diagram to illustrate the data flow between each component in your pipeline.*

## Tools and Technologies
- **Kafka**: For real-time data streaming.
- **Apache Flume**: To transfer data from Kafka to HDFS.
- **PySpark**: For processing and transforming streaming data.
- **InfluxDB**: A time-series database for storing weather data.
- **Grafana**: Visualization tool for monitoring trends.
- **Jupyter Notebook**: Documentation and interactive development environment.

## Project Workflow
1. **Data Streaming**:
   - Weather forecast data is streamed from a source (using Python's KafkaProducer) and published to a Kafka topic.

2. **Data Ingestion to HDFS**:
   - Apache Flume is used to efficiently transfer data from Kafka to HDFS, providing a backup storage layer.

3. **Data Processing**:
   - Data is consumed from Kafka using PySpark.
   - Transforms are applied to pivot the data by day and time (morning/night), removing irrelevant entries (e.g., 'Overnight').

4. **Storage and Visualization**:
   - Processed data is saved to InfluxDB, and real-time visualizations are configured in Grafana.

## Metadata used:
- Location: Latitude and longtitude are set on Washington, D.C.
- short_forecast: A brief description of the forecast (e.g., "Partly Cloudy").
- Temperature: The forecasted temperature.
- Temperature unit: The unit of temperature (Fahrenheit).
- Wind speed: The forecasted wind speed.
- Start-time: The start time of the forecast period which is every 7 days.

## Usage:

### _Step #1: Run the following commands in separate terminals_
Start HDFS and YARN:
```sh
start-dfs.sh
```
Start Zookeeper Server:
```sh
cd $KAFKA_HOME
```
```sh
bin/zookeeper-server-start.sh config/zookeeper.properties 
```
Start Kafka Server:
```sh
cd $KAFKA_HOME
```
```sh
bin/kafka-server-start.sh config/server.properties 
```
(Optional) Checking ecosystem running successfully:
```sh
jps
```
#### Choose either:

<details>

<summary>Weather API -> Kafka -> Flume -> HDFS</summary>

### You can add a header

You can add text within a collapsed section. 

You can add an image or a code block, too.

```ruby
   puts "Hello World"
```
</details>



<details>
<summary>Weather API -> Kafka -> PySpark -> influxDB -> Grafana</summary>

### _Step #2: Setup influxDB:_
1. Create an account.
2. Name your organization (Will be used later in the code).
3. Create a new bucket.
   
![creating bucket](https://github.com/user-attachments/assets/7fe1f32b-4672-4794-9a31-6da557b46ccf)
4. Create a new API Token (Will be used later in the code).

![InfluxDB API token creation](https://github.com/user-attachments/assets/1a42d8c2-a994-41d2-870c-9d03758e620e)

### _Step #3: Modify and run the following code:_
#### Ensure following packages are installed.
```python
!pip install influxdb_client # install the influxdb_client to enable streaming to influxDB
```
```python
!pip install certifi # provides Mozilla's trusted SSL/TLS certificates for secure HTTPS connections in Python
```

#### Code implementing stream: Source -> Kafka -> PySpark -> InfluxDB


```python
import requests
import json
import time
from kafka import KafkaProducer, KafkaConsumer
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType
from pyspark.sql import Row
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS
import certifi
import threading

# Kafka configuration
KAFKA_BROKER = 'localhost:9092'
TOPIC = 'weatherlogs'

# InfluxDB configuration
token = "YOUR_TOKEN_HERE"
org = "YOUR_ORGANIZATION_HERE"
bucket = "YOUR_BUCKET_HERE"
url = "YOUR_URL_INSTANCE_HERE"

# Kafka producer setup
producer = KafkaProducer(
    bootstrap_servers=[KAFKA_BROKER],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Spark session initialization
spark = SparkSession.builder \
    .appName("Weather") \
    .getOrCreate()

# Schema for incoming data
schema = StructType([
    StructField("location", StringType(), True),
    StructField("short_forecast", StringType(), True),
    StructField("temperature", StringType(), True),
    StructField("temperature_unit", StringType(), True),
    StructField("wind_speed", StringType(), True),
    StructField("start_time", StringType(), True)
])

# InfluxDB client setup
client = InfluxDBClient(url=url, token=token, org=org, ssl_ca_cert=certifi.where())
write_api = client.write_api(write_options=SYNCHRONOUS)

latitude, longitude = 38.8977, -77.0365  # Washington, D.C. coordinates

def stream_forecast():
    while True:
        try:
            api_url = f"https://api.weather.gov/points/{latitude},{longitude}"
            response = requests.get(api_url)
            response.raise_for_status()
            
            data = response.json()
            forecast_url = data['properties']['forecast']

            forecast_response = requests.get(forecast_url)
            forecast_response.raise_for_status()
            
            forecast_data = forecast_response.json()
            for period in forecast_data['properties']['periods']:
                forecast_info = {
                    "location": "Washington, D.C.",
                    "short_forecast": period['shortForecast'],
                    "temperature": period['temperature'],
                    "temperature_unit": period['temperatureUnit'],
                    "wind_speed": period['windSpeed'],
                    "start_time": period['startTime']
                }
                producer.send(TOPIC, value=forecast_info)
            time.sleep(604800)  # Stream every 7 days
        except requests.RequestException as e:
            print("Failed to fetch data:", e)
            time.sleep(3600)  # Retry after 1 hour on failure

def consume_and_store():
    consumer = KafkaConsumer(
        TOPIC,
        bootstrap_servers=KAFKA_BROKER,
        auto_offset_reset='latest',
        value_deserializer=lambda x: json.loads(x.decode('utf-8'))
    )
    for message in consumer:
        data = message.value
        if isinstance(data, dict):
            row = Row(
                location=data['location'],
                short_forecast=data['short_forecast'],
                temperature=str(data['temperature']),
                temperature_unit=data['temperature_unit'],
                wind_speed=data['wind_speed'],
                start_time=data['start_time']
            )
            df = spark.createDataFrame([row], schema=schema)
            point = (
                Point("weather.forecast")
                .tag("location", row['location'])
                .tag("short_forecast", row['short_forecast'])
                .field("temperature", row['temperature'])
                .field("wind_speed", row['wind_speed'])
                .time(row['start_time'])
            )
            write_api.write(bucket=bucket, org=org, record=point)
            print(f"Data written to InfluxDB: {row['start_time']}, {row['temperature']}°{row['temperature_unit']} , {row['short_forecast']}")

if __name__ == "__main__":
    try:
        threading.Thread(target=stream_forecast).start()
        consume_and_store()
    except KeyboardInterrupt:
        print("Streaming stopped.")
    finally:
        producer.close()
        client.close()
```
### _Step #4: Show the data streamed in a table:_
1. Go to Data Explorer (the graph-like icon on the left).
2. Choose your bucket.
3. Choose your Measurment [which is -> Point("weather.forecast")].
4. Then query using sql to show the time-series output.
![Influx interface](https://github.com/user-attachments/assets/b48790fa-27f7-46ab-87d2-afb77e73f6ff)

### _Step #5: Visualize the data streamed:_
1. Go to Grafana.
2. Create a new account.
3. Then go to Connections -> Add new connection.
4. Choose influxDB
5. Enter the relevant data needed to connect to influxDB. [Tutorial](https://www.youtube.com/watch?v=rSsouoNsNDs)
6. Create a new dashboard then write the same query as in image:
![Grafana Visualization](https://github.com/user-attachments/assets/0c719136-df9e-4d15-8ae1-981ce1c90f69)

>After running the query, a short line is plotted. Therefore, click on the minimize icon multiple times ensuring all 7 days are plotted. (Shown in previous image)
</details>

## Future Improvements
- Enhance data cleaning and transformation processes.
- Add support for additional weather parameters.
- Explore alternative storage options for scalability.

