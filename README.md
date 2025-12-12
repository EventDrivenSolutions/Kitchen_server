kitchen_server – MQTT Sensor Monitoring Backend

kitchen_server is the server-side stack for an MQTT-based sensor monitoring project.
It hosts a local Mosquitto MQTT broker, ingests sensor data via Telegraf, stores it in InfluxDB 2, and exposes metrics to Grafana dashboards for real-time visualization and status monitoring.

Features & Requirements Mapping

This backend is designed to satisfy the following project requirements:

Local MQTT Server Hosting
A Mosquitto broker runs locally (default tcp/1883) and accepts MQTT client connections over the LAN (Wi-Fi or Ethernet).

MQTT–Grafana Integration for Request Monitoring
Telegraf subscribes to MQTT topics, parses JSON payloads, and writes metrics to InfluxDB 2. Grafana uses InfluxDB as a data source to visualize MQTT message activity and sensor values.

Reception & Visualization of Sensor Data
The server receives temperature, humidity, and pressure readings from the client and visualizes them over time in Grafana using bar/time-series panels.

Network Interface Identification
The client encodes its connection type (Wi-Fi vs Ethernet) in the MQTT payload. Telegraf stores this as a tag, and Grafana displays it in a dedicated status panel.

Connection Fault Detection & Indication
Each packet includes a functional flag. Grafana’s Stat panels show “Functional” vs “Non-functional”, making connection or device faults visible on the dashboard.

Architecture
High-Level Flow

Client (MQTT publisher)
  | MQTT (tcp/1883)
  v
Mosquitto broker (local)
  | MQTT subscribe (mqtt_consumer)
  v
Telegraf agent
  | InfluxDB line protocol (HTTP)
  v
InfluxDB 2 (mqtt_bucket)
  | Flux queries
  v
Grafana dashboards

Components

Client device – Publishes sensor readings as JSON over MQTT on the local network.

Mosquitto (MQTT broker) – Runs on the server machine, listens on 0.0.0.0:1883, and routes MQTT PUBLISH/SUBSCRIBE traffic.

Telegraf – Uses the mqtt_consumer input plugin to subscribe to sensor topics, parses JSON messages, and writes them to InfluxDB 2.

InfluxDB 2 – Stores time-series metrics in a bucket (e.g., mqtt_bucket) with tags and fields derived from the MQTT payload.

Grafana – Connects to InfluxDB 2 as a data source and renders dashboards with bar charts for temperature, humidity, and pressure, plus Stat panels for connection type and functional status.

Technologies

Mosquitto – Lightweight MQTT broker for publish/subscribe communication.

Telegraf – Metrics and logging agent with an mqtt_consumer input and InfluxDB 2 output.

InfluxDB 2 – Time-series database storing sensor metrics (bucket: mqtt_bucket).

Grafana – Visualization and dashboarding tool using InfluxDB as a data source.

Python + paho-mqtt (optional) – Example client and traffic generator.

Wireshark (optional) – Used to inspect MQTT packets on tcp/1883 during development.

Data Model
MQTT Topic

The server expects sensor data on MQTT topics of the form:

sensors/<device_id>/data

Example: sensors/kitchen_node_1/data

JSON Payload

The client publishes JSON payloads like:

{
"connection_type": 0,
"functional": 1,
"device_id": "kitchen_node_1",
"temp": 23.1,
"humidity": 34.3,
"pressure": 1013.6
}

Where:

connection_type – Encodes the network interface used by the client

0 → Wi-Fi

1 → Ethernet

functional – Device / station health flag

1 → Functional/OK

0 → Non-functional / fault

device_id – Logical identifier of the sensor node (e.g., kitchen_node_1)

temp – Temperature reading in °C

humidity – Relative humidity in %

pressure – Pressure reading (e.g., hPa or mbar)

InfluxDB Measurement

Telegraf maps the JSON payload into an InfluxDB measurement (default: mqtt_consumer):

Measurement:

mqtt_consumer

Tags (indexed):

device_id – e.g., "kitchen_node_1"

connection_type – "0" or "1" stored as a tag

Fields (numeric values):

temp – float

humidity – float

pressure – float

functional – integer (0 or 1)

This structure lets Grafana filter and group by device, connection type, and time.

Repository Structure

A suggested structure for this repo:

.
├── README.md – This file
├── LICENSE – Project license (e.g., MIT)
├── config
│ ├── mosquitto.conf – Local Mosquitto configuration
│ └── telegraf.conf – Telegraf MQTT → InfluxDB configuration
├── scripts
│ ├── mqtt_flood_local.sh – Synthetic MQTT traffic generator (optional)
│ └── mqtt_client_example.py – Example MQTT client using paho-mqtt (optional)
├── grafana
│ └── kitchen_dashboard.json – Exported Grafana dashboard definition
└── docs
├── architecture-diagram.png – Architecture diagram (optional)
└── slides.pdf – Project presentation (optional)

Not all files are required to run the system, but this layout keeps configuration, code, and documentation clearly separated.

Setup & Installation

These steps assume macOS with Homebrew. On Linux, commands are similar but package management paths will differ.

1. Clone the repository

git clone https://github.com/EventDrivenSolutions/kitchen_server.git

cd kitchen_server

2. Install prerequisites

Install Mosquitto, Telegraf, InfluxDB 2, and Grafana. On macOS with Homebrew:

brew install mosquitto telegraf
brew install --cask influxdb influxdb-cli grafana

(Adjust package names depending on your environment / existing installations.)

3. Configure Mosquitto

Copy the project’s Mosquitto config into the system location (paths may vary):

sudo cp config/mosquitto.conf /opt/homebrew/etc/mosquitto/mosquitto.conf
brew services restart mosquitto

Key settings in mosquitto.conf:

listener 1883
bind_address 0.0.0.0
allow_anonymous true (for lab/demo use only)

This starts a local MQTT broker listening on 0.0.0.0:1883.

4. Configure Telegraf

Copy the project Telegraf config:

sudo cp config/telegraf.conf /opt/homebrew/etc/telegraf.conf

Edit /opt/homebrew/etc/telegraf.conf and update the InfluxDB output:

[[outputs.influxdb_v2]]
urls = ["http://localhost:8086"]
token = "YOUR_INFLUXDB_TOKEN"
organization = "YOUR_INFLUX_ORG"
bucket = "mqtt_bucket"

Then restart Telegraf:

brew services restart telegraf

5. InfluxDB 2 setup

Start InfluxDB 2 (via app, Docker, or service).

Create:

An organization (e.g., enpm818m_lab).

A bucket named mqtt_bucket.

Generate a read/write API token and put it into the Telegraf config.

6. Grafana setup

Start Grafana (often via Homebrew service or app):

brew services start grafana

Open Grafana at: http://localhost:3000

Add an InfluxDB 2 data source:

URL: http://localhost:8086

Organization: your InfluxDB org

Token: same token used by Telegraf

Default bucket: mqtt_bucket

Import the provided dashboard:

Go to “Dashboards → Import”

Upload grafana/kitchen_dashboard.json

Select the InfluxDB data source you just created.

Running the System

Assuming services are configured and installed:

Start Mosquitto:

brew services start mosquitto

Start Telegraf:

brew services start telegraf

Ensure InfluxDB 2 and Grafana are running:

InfluxDB 2 UI: http://localhost:8086

Grafana UI: http://localhost:3000

Start the client (or synthetic generator):

Either run your real sensor client on the same LAN,

Or use the example client / flood script (see next section).

Open the Grafana dashboard:

Navigate to the imported “Kitchen Server” / MQTT dashboard.

You should see:

Bar charts for temperature, humidity, and pressure.

Stat panel for connection_type (Wi-Fi vs Ethernet).

Stat panel for functional status (Functional vs Non-functional).

Testing with Synthetic Traffic

When you don’t have the real client running, you can still generate MQTT traffic from localhost to exercise the pipeline.

Option A – Shell script: mqtt_flood_local.sh

cd scripts
chmod +x mqtt_flood_local.sh
./mqtt_flood_local.sh

This script publishes a burst of JSON packets with randomized sensor values to the configured MQTT topic. As long as Mosquitto and Telegraf are running, these packets will flow into InfluxDB and be visible in Grafana.

Option B – Python example client: mqtt_client_example.py

If you have Python and paho-mqtt installed:

pip install paho-mqtt

cd scripts
python mqtt_client_example.py

This script simulates a single device (kitchen_node_1) sending random values for temp, humidity, pressure, connection_type, and functional once per second. It’s useful for continuous, realistic-looking test data during demos.

License

This project is licensed under the MIT License.
See the LICENSE file for details.

Acknowledgements

Developed as part of an academic project on networking and cloud monitoring.

Uses the excellent open source work of:

The Mosquitto MQTT broker project

InfluxData (Telegraf & InfluxDB)

Grafana Labs (Grafana)

Many thanks to the maintainers of these tools and the broader open-source community.
