Nginx Monitoring with Prometheus & Grafana (WSL Guide)

Prerequisites
Make sure Docker is installed and running in WSL.
bashdocker --version
docker compose version

If not installed, get Docker Desktop for Windows and enable the WSL 2 backend from its settings. Docker Desktop runs the Docker engine, and WSL talks to it directly.


Step 1 — Create a Project Folder
bashmkdir ~/nginx-monitoring && cd ~/nginx-monitoring

Creates a dedicated folder for all your config files and moves into it. All files you create next will live here.


Step 2 — Create the Prometheus Config File
bashnano prometheus.yml
Paste this inside:
yamlglobal:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "nginx"
    static_configs:
      - targets: ["nginx-exporter:9113"]
Save with Ctrl+X → Y → Enter

This tells Prometheus to collect (scrape) metrics every 15 seconds — from itself and from the nginx-exporter container. The exporter is what reads Nginx's internal stats and converts them into a format Prometheus understands.


Step 3 — Create the Nginx Config File
bashnano nginx.conf
Paste this inside:
nginxevents {}

http {
  server {
    listen 80;

    location /nginx_status {
      stub_status on;
      allow all;
    }
  }
}
Save with Ctrl+X → Y → Enter

Enables the /nginx_status page on Nginx, which shows live stats like active connections and total requests. The nginx-exporter reads this page and feeds the data to Prometheus.


Step 4 — Create the Docker Compose File
bashnano docker-compose.yml
Paste this inside:
yamlversion: '3'

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"

  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter
    container_name: nginx-exporter
    command:
      - -nginx.scrape-uri=http://nginx/nginx_status
    ports:
      - "9113:9113"
    depends_on:
      - nginx
Save with Ctrl+X → Y → Enter

This single file defines all 4 containers:

prometheus — collects and stores metrics
grafana — displays metrics as graphs
nginx — the web server being monitored
nginx-exporter — reads Nginx stats and exposes them to Prometheus



Step 5 — Start All Containers
bashdocker compose up -d

-d means "detached" — runs everything in the background. First run will pull Docker images, which takes 1–2 minutes. After that it's instant.


Step 6 — Open in Your Browser
ServiceURLLoginPrometheushttp://localhost:9090none neededGrafanahttp://localhost:3000admin / adminNginxhttp://localhost:80none needed

Step 7 — Connect Grafana to Prometheus
In Grafana:

Left sidebar → Connections → Data Sources
Click Add data source
Choose Prometheus
Set URL to: http://prometheus:9090
Click Save & Test — should say "Successfully queried"


Grafana needs to know where to pull data from. You're pointing it at the Prometheus container using its internal Docker network name (prometheus), not localhost.


Step 8 — Generate Nginx Traffic (to see metrics)
bashfor i in {1..50}; do curl http://localhost & done

Sends 50 HTTP requests to Nginx in parallel. The & runs each curl in the background so they all fire at once. This gives Prometheus real traffic data to collect, so your Grafana graphs actually show something instead of flat lines.


Step 9 — Import a Grafana Dashboard

Left sidebar → Dashboards → New → Import
In the "Import via grafana.com" box, type 12708 and click Load
Select your Prometheus data source from the dropdown
Click Import


Dashboard ID 12708 is an official Nginx dashboard — it auto-creates panels for active connections, request rates, and more.

Alternatively, you can add custom panels manually with these PromQL queries:

Active connections: nginx_connections_active
Total requests: nginx_http_requests_total


Step 10 — Stop Everything When Done
bashdocker compose down

Stops and removes all running containers. Your config files (prometheus.yml, nginx.conf, docker-compose.yml) stay intact — just run docker compose up -d again to bring it all back.


Quick Reference
ActionCommandStart all containersdocker compose up -dStop all containersdocker compose downCheck container statusdocker compose psView logsdocker compose logs -fGenerate test trafficfor i in {1..50}; do curl http://localhost & done
