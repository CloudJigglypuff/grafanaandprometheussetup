# Readme

The guide describes how to make a monitoring setup with Grafana and Prometheus Node Exporter using Docker containers.

1. Install Docker.<br>
a) Windows/Mac: https://www.docker.com/products/docker-desktop/<br>
b) Linux: https://docs.docker.com/desktop/install/linux-install/
2. Afterward, create two files in the desired folder, and paste the configuration into the corresponding files:<br><br>
**`docker-compose.yml`**
```
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data: {}

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - 9090:9090
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:7.5.7
    container_name: grafana
    ports:
      - 3000:3000
    restart: unless-stopped
    volumes:
      - ./grafana/provisioning/datasources:/etc/grafana/provisioning/datasources
      - ./grafana:/var/lib/grafana
```
**`prometheus.yml`**
```
global:
  scrape_interval: 10s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```
3. Run `docker-compose up -d` in the folder where the files were created;
4. Open http://localhost:3000 to log into Grafana. Use the default admin credentials:<br>
username: admin<br>
password: admin
5. Go to "Configuration" and click on "Add data source";
6. Find "Prometheus" and click on "Select";
7. Since localhost is not recognized as server host when running both Grafana and Prometheus, set `http://host.docker.internal:9090` in the "URL" field, and then click on "Save & Test";
8. Now we can create our monitoring panels. Go back to the main page, click on '+' (Create) in the left bar and select "Dashboard";
9. Click on "Add an empty panel";
10. Select "Prometheus" as data source (if not selected):
![Untitled](https://github.com/CloudJigglypuff/grafanaandprometheussetup/assets/76413235/55c214cb-3990-42df-917b-09cbd106c2ff)
1. **CPU.** In this step, we will set up a panel to monitor CPU utilization:
   1. Enter the following query:<br>`avg without (cpu)(irate(node_cpu_seconds_total{mode!="idle"}[5m]))`<br><br>
    `irate` calculates the per-second instant rate of increase of the time series in the range vector.<br>
    `avg without (cpu)` is used to see the overall CPU utilization across all CPUs, and removes the "cpu" label.<br>
    `mode!="idle"` is used to ignore iddle time.
   2. Add `{{mode}}` to the "Legend" field under the query.
   3. To display correct %, go to the "Display" menu on the right, find "Axes", then locate the "Unit" field under "Left Y", and specify: **Percent (0.0-1.0)**:
    ![Untitled2](https://github.com/CloudJigglypuff/grafanaandprometheussetup/assets/76413235/2f6b7199-2281-4d27-b0ce-96efb3f3f5e6)
  3. Click on "Apply":
  ![Pasted image 20231011022109](https://github.com/CloudJigglypuff/grafanaandprometheussetup/assets/76413235/d7fb7080-c68a-4831-b040-1a38ce635dfc)
  4. As a result, you will see your Dashboard with the newly created CPU panel:
	![Pasted image 20231011022500](https://github.com/CloudJigglypuff/grafanaandprometheussetup/assets/76413235/8e398328-82d5-482f-942f-3914dfac9914)
3. **Memory Usage.** Let's add a panel to monitor memory usage. 
	1. Click on "Add panel"; 
	2. Click on "Add an empty panel" and add the following queries:
	   - 1st query (A):<br>`node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Cached_bytes - node_memory_Buffers_bytes`
	   - 2nd query (B):<br>`node_memory_Buffers_bytes`
	   - 3rd query (C):<br>`node_memory_Cached_bytes`
	   - 4th query (D):<br>`node_memory_MemFree_bytes`
  3. Add `Used`, `Buffers`, `Cached`, and `Free` to the corresponding "Legend" fields;
	3. To see correct values in the graph, enter "Bytes(IEC)" in the "Unit" field;
  4. Add `{{device}}` to the "Legend" field under the query.
	5. Click on "Apply".
5. **Used Disk Space.** Now we will add a panel to monitor used disk space:
	1. Click on "Add panel"; 
	2. Click on "Add an empty panel" and add the following query:<br>`node_filesystem_size_bytes - node_filesystem_avail_bytes {device!="tmpfs", device!="none"}`
	3. To see correct values in the graph, enter "Bytes(IEC)" in the "Unit" field;
  4. Add `{{device}}` to the "Legend" field under the query.
  5. Click on "Apply"
5. **Available Disk Space.** Here we will monitor available disk space.
	1. Click on "Add panel";
	2. Click on "Add an empty panel" and add the following query:<br>`node_filesystem_avail_bytes{device!="tmpfs", device!="none"}`
  3. Add `{{device}}` to the "Legend" field under the query.
	4. To see correct values in the graph, enter "Bytes(IEC)" in the "Unit" field;
6. **Network Traffic.** In the last panel, we will monitor network traffic.
	1. Click on "Add panel";
	2. Click on "Add an empty panel" and the following queries:
	   - 1st query (A):<br>`irate(node_network_receive_bytes_total[5m])`
	   - 2nd query (B):<br>`irate(node_network_transmit_bytes_total[5m])`
  3. Add `{{device}} receive` and `{{device}} transmit` to the corresponding "Legend" fields;
	4. To see correct values in the graph, enter "kilobytes/sec" in the "Unit" field;
  5. Click on "Apply"
7. Now you can save the last panel and save the Dashboard. Here is a screenshot of the final result:
![Pasted image 20231011221935](https://github.com/CloudJigglypuff/grafanaandprometheussetup/assets/76413235/3ec5a74c-60a3-4983-a635-4a7a2e6219ce)

