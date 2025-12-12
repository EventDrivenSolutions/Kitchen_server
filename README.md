# MQTT Sensor Monitoring with Grafana

This project implements a small, end-to-end IoT monitoring pipeline using MQTT, Telegraf, InfluxDB 2, and Grafana.  
A client simulates an environmental sensor and publishes JSON payloads over MQTT to a locally hosted Mosquitto broker.  
Telegraf consumes these MQTT messages, writes them to InfluxDB, and Grafana visualizes the data and connection status.

---

## Features & Requirements Mapping

1. **Local MQTT Server Hosting**  
   - Mosquitto MQTT broker running locally on TCP port `1883`  
   - Accepts client connections over Wi-Fi or Ethernet on the same LAN  

2. **MQTT–Grafana Integration for Request Monitoring**  
   - Telegraf `mqtt_consumer` input subscribes to sensor topics  
   - Metrics written to InfluxDB 2 (`mqtt_bucket`)  
   - Grafana connects to InfluxDB and exposes MQTT traffic as dashboards  

3. **Reception & Visualization of Sensor Data**  
   - Client sends JSON payloads including temperature, humidity, and pressure  
   - Data stored in InfluxDB and shown in Grafana as bar charts over time  
   - Panels per metric (temp, humidity, pressure) plus an integrated overview panel  

4. **Network Interface Identification**  
   - Client encodes connection type in each MQTT payload (`connection_type` tag)  
   - Grafana Stat panel displays current connection type (e.g., Wi-Fi vs Ethernet)  

5. **Connection Fault Detection & Indication**  
   - Client sends a `functional` flag in each packet  
   - Grafana Stat panel shows “Functional” vs “Non-functional” based on latest value  
   - Can be extended with time-based logic for missing data / connection loss  

---

## Architecture

```text
Client (MQTT publisher)
    |
    |  MQTT (tcp/1883)
    v
Mosquitto broker (local)
    |
    |  MQTT subscribe (mqtt_consumer)
    v
Telegraf agent
    |
    |  InfluxDB line protocol (HTTP)
    v
InfluxDB 2 (mqtt_bucket)
    |
    |  Flux queries
    v
Grafana dashboards

