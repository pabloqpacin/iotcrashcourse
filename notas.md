# IoT dockerized-microservices' arquitecture

> [Curso IoT 0 a 100!: Docker, Compose, InfluxDB, Grafana, MQTT, Sensor virtual Python y Telegraf!](https://www.youtube.com/watch?v=CMXUXdLzrPE)

- [IoT dockerized-microservices' arquitecture](#iot-dockerized-microservices-arquitecture)
  - [NOTAS](#notas)
  - [STACK](#stack)
    - [-mock sensor-](#-mock-sensor-)
    - [-docker compose-](#-docker-compose-)
    - [EMQX](#emqx)
    - [Telegraf\_IN -- tail \& mqtt](#telegraf_in----tail--mqtt)
    - [InfluxDB](#influxdb)
    - [Telegraf\_OUT -- mqtt\_consumer \& influxdb\_v2](#telegraf_out----mqtt_consumer--influxdb_v2)
    - [Grafana](#grafana)


## NOTAS

- Preguntas / Comentarios
  - [ ] DC version: siempre la última?
  - Destino final (TEMA INFLUXDB?)? Python data analysis? ~~as per 01:04 en el vid de 16 minutos de abajo en https://www.influxdata.com/time-series-platform/telegraf/~~
  - alertas/eventos
- NOTAS
  - OJO `docker build` for mock -- no image
  - Evitar usar `latest`
  - [ ] Seguridad, permisos, ...
  - [ ] Docker secrets... (Grafana)
  - Telegraf: tags, plugins, ...
  - The default unencrypted MQTT port is 1883. The encrypted port is 8883.[20]
- Research
  - [Edge computing](https://en.wikipedia.org/wiki/Edge_computing)
  - [Message broker](https://en.wikipedia.org/wiki/Message_broker)
  - [MQTT](https://en.wikipedia.org/wiki/MQTT)


## STACK

### -mock sensor-

Implementación en python para enviar DATA: registros de temperatura

<details>

```md
**Level 1 - Simple Explanation:**
This code checks if the Python script is being run directly (not imported as a module into another script). If it is being run directly, it calls the `main()` function.

**Level 2 - Intermediate Explanation:**
The `if __name__ == '__main__':` block is a common Python idiom used to determine whether the Python script is the main program being executed or if it is being imported as a module into another script. When the script is executed directly, the special variable `__name__` is set to `'__main__'`. So, the condition `if __name__ == '__main__':` checks if the script is the main program. If true, it proceeds to call the `main()` function.

**Level 3 - Advanced Explanation:**
In Python, the `__name__` variable is a built-in variable that represents the name of the current module. When a Python script is executed, the interpreter assigns the value `'__main__'` to `__name__` if the script is the main program being run. If the script is imported as a module into another script, `__name__` is set to the name of the module.
The `if __name__ == '__main__':` construct takes advantage of this behavior. It ensures that the code block beneath it is only executed when the script is run directly, not when it is imported as a module. This is useful for separating reusable code (functions, classes) from code that should only run when the script is executed independently. In this specific case, when the script is run directly, it calls the `main()` function.
```
```bash
# Python Dictionary == Bash Ass. Array

# Declare an associative array
declare -A sensor_data

# Assign values
sensor_data["device_id"]="e2e78334"
sensor_data["client_id"]="c03d5155"
sensor_data["sensor_type"]="Temperature"
sensor_data["temperature"]=25
sensor_data["timestamp"]=$(date +%s)

# Access values
echo ${sensor_data["temperature"]}
```
```py
output_file.write(f'{json.dumps(sensor_data)}\n')         # OJO
```

</details>

### -docker compose-


```yaml
mocksensor:
    build: mocksensor
    volumes:
      - ./data/mocksensor:/tmp/
```

```bash
docker compose up -d
tail -f data/mocksensor/out*
```

### EMQX

Broker de mensajería MQTT

```yaml
emqx:
    user: root              # OJO
    image: "emqx:4.4.3"
    ports:
      - "18083:18083" 
    volumes:
      - ./data/emqx/data:/opt/emqx/data
      - ./data/emqx/log:/opt/emqx/log
    environment:
      - EMQX_DASHBOARD__DEFAULT_USER__LOGIN=pabloqpacin
      - EMQX_DASHBOARD__DEFAULT_USER__PASSWORD=changeme
```

<details>
<summary>10x Permissions</summary>

```md
**ERROR**
Faltan permisos para container user `emqx` sobre `/data/emqx/data` (actualmente para `root` localmente).
Habría que habilitar tales permisos...
*apaño*: `user: root`
```

```md
**SOLUTION** ~~ChatGPT~~

1. EMQX Dockerfile...

# Add these lines to the Dockerfile
USER emqx
RUN chown -R emqx:emqx /opt/emqx/data /opt/emqx/log


2. Docker volumes

    volumes:
      - emqx-data:/opt/emqx/data
      - emqx-log:/opt/emqx/log

volumes:
  emqx-data:
  emqx-log:

docker voluem ls
docker volume inspect emqx-data
```

</details>


```bash
docker compose up

xdg-open http://localhost:18083
```

PANEL ADMINISTRACIÓN WEB --> SUPER

```bash
# ojete
docker network ls
docker inspect iotcrashcourse_default
```

### Telegraf_IN -- tail & mqtt

> https://www.influxdata.com/time-series-platform/telegraf/

Capturar datos y reenviarlos

```yaml
  telegrafinput:
    image: "telegraf:1.22.4"
    volumes:
      - ./data/telegrafinput/telegraf.conf:/etc/telegraf/telegraf.conf
      - ./data/mocksensor:/tmp/mocksensor
```
```bash
sudo mkdir ./data/telegrafinput
sudo chmod 777 ./data/telegrafinput
sudo nvim data/telegrafinput/telegraf.conf
```

> - https://github.com/influxdata/telegraf/tree/master/plugins/inputs/tail

```conf
[global_tags]
  project = "iotcrashcourse"

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = "telegrafinput"      # OJO -- si no, hostname generado por DOcker random
  omit_hostname = false

[[inputs.tail]]
  files = ["/tmp/mocksensor/ouput_mock_sensor.json"]
  data_format = "json"

  tag_keys = [
    "device_id",
    "client_id"
  ]

  json_name_key = "sensor_type"
  json_time_key = "timestamp"       # OJO -- mejor registrar en el propio sensor/mock que aquí
  json_time_format = "unix"


[[outputs.mqtt]]
  servers = ["emqx:1883"]
  topic_prefix = "telegraf"
  qos = 2
  data_format = "influx"
  retain = true

```

```bash
docker compose up -d
docker compose down
```


### InfluxDB

<!--
> - https://docs.influxdata.com/
> - https://www.influxdata.com/university/

```python
# https://www.influxdata.com/

from influxdb_client_3 import InfluxDBClient3
import pandas
import os

database = os.getenv('INFLUX_DATABASE')
token = os.getenv('INFLUX_TOKEN')
host="https://us-east-1-1.aws.cloud2.influxdata.com"

def querySQL():
  client = InfluxDBClient3(host, database=database, token=token)
  table = client.query(
    '''SELECT
        room,
        DATE_BIN(INTERVAL '1 day', time) AS _time,
        AVG(temp) AS temp,
        AVG(hum) AS hum,
        AVG(co) AS co
      FROM home
      WHERE time >= now() - INTERVAL '90 days'
      GROUP BY room, _time
      ORDER BY _time'''
  )

  print(table.to_pandas().to_markdown())

  client.close()
  
querySQL()
```
-->

```yaml
  influxdb:
    image: "influxdb:2.2.0-alpine"
    ports: 
      - "8086:8086"
    volumes:
      - ./data/influxdb/data:/var/lib/influxdb2
      - ./data/influxdb/config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=manolo
      - DOCKER_INFLUXDB_INIT_PASSWORD=iotcrashcourse
      - DOCKER_INFLUXDB_INIT_ORG=manolotech
      - DOCKER_INFLUXDB_INIT_BUCKET=iotdata
      - DOCKER_INFLUXDB_INIT_RETENTION=1w
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=manolotech-iotdata-auth-token  # ojo

```
```md
Certainly! Let me elaborate on the concepts of "bucket," "retention policy," and "admin token" in the context of InfluxDB:

1. **Bucket:**
   - In InfluxDB, a "bucket" is a container for time-series data. It is essentially a logical grouping or storage compartment where data is stored.
   - Each bucket can be associated with specific data characteristics, such as data retention policy, shard duration, and more.
   - When you write data into InfluxDB, you specify the target bucket for that data. Queries and data manipulations are often scoped to a particular bucket.

2. **Retention Policy:**
   - A "retention policy" in InfluxDB defines how long data is retained in a bucket before it is automatically deleted.
   - It allows you to control the duration for which data points are kept in the database. Once data reaches the expiration time defined by the retention policy, it is eligible for deletion.
   - The retention policy is specified in a duration format, such as "1w" for 1 week or "30d" for 30 days.

3. **Admin Token:**
   - An "admin token" is a form of authentication token with administrative privileges.
   - In the context of InfluxDB, the admin token is used for administrative operations, such as managing users, organizations, and other administrative tasks.
   - It provides a secure way to perform administrative actions without requiring the username and password of an admin user.
   - The specific token value, such as "manolotech-iotdata-auth-token," is unique to your deployment and serves as a secure means of authenticating and authorizing administrative operations.

These concepts are specific to InfluxDB and are not directly related to AWS. They are fundamental to working with time-series data in InfluxDB, providing a way to organize, manage, and control access to data within the database. Understanding these concepts allows you to configure InfluxDB to suit your data storage and retention needs.
```

```bash
docker compose up

xdg-open http://localhost:8086

docker compose down
```


### Telegraf_OUT -- mqtt_consumer & influxdb_v2

<!-- Capturar datos y reenviarlos -->

```bash
sudo cp -r data/telegrafinput data/telegrafoutput
sudo nvim data/telegrafoutput/telegraf.conf
```

```conf
[global_tags]
  project = "iotcrashcourse"

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"
  precision = "0s"
  hostname = "telegrafoutput"                 # ojo
  omit_hostname = false

[[inputs.mqtt_consumer]]
  servers = ["emqx:1883"]
  topics = [
    "telegraf/telegrafinput/#"
  ]
  qos = 2

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "manolotech-iotdata-auth-token"
  organization = "manolotech"
  bucket = "iotdata"
```

...

```yaml
  telegrafoutput:
    image: "telegraf:1.22.4"
    deploy:
      # restart_policy:
      #   condition: on-failure
      #   delay: 10s
      #   max_attempts: 20
    volumes:
      - ./data/telegrafoutput/telegraf.conf:/etc/telegraf/telegraf.conf
```

> - [ ] Script verify `emqx` running then `telegrafoutput` (52:00-52:) ~~`depends-on` in compose ain't a fix (up != ready)

...

```bash
docker compose up

# EMQX > Clients, Topics
xdg-open localhost:18083

# Influx >
xdg-open localhost:8086
```

<!-- > - [ ] problem with influx/telegraph `auth token`... -->

### Grafana

```yaml
  grafana:
    user: root                                                          # OJO -- bad
    image: "grafana/grafana:8.5.3"
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/admin_password    # OJO
    volumes:
      - ./data/grafana:/var/lib/grafana
    secrets:
      - source: grafana_admin_password
        target: /run/secrets/admin_password
secrets:
  grafana_admin_password:
    file: ./secrets/grafana_admin_password
```

```bash
# do secrets...

docker compose up

xdg-open localhost:3000
```
```yaml
# Grafana
Configuration:
  Add Data Source: InfluxDB
    Query Language: Flux
    URL: http://influxdb:8086       # NOTA !! En Docker Compose, lo servicios se ven entre ellos y tal
    InfluxDB Details:
      Organization: manolotech
      Token: manolotech-iotdata-auth-token
      Default BUcket: iotdata

Create: Dashboard
  New panel:

  # from(bucket: "iotdata")
  # |> range(start: v.timeRangeStart, stop:v.timeRangeStop)
  # |> aggregateWindow(every: 1m,fn: mean)
  # |> filter(fn: (r) =>
  #   r._measurement == "Temperature" and 
  #   r.device_id == "e2e78334"
  # )

```

