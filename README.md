# Graphana + Loki + Promtail - Local setup
## Prerequisites

	•	Docker and Docker Compose installed on your system.
	•	Basic knowledge of Docker and containerized applications.
	•	Your application code and Dockerfile are ready.

### Step 1: Set Up Your docker-compose.yml File

Create a docker-compose.yml file in the root directory of your project with the following content:

```yaml
version: '3.8'

services:
  # Your application service
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD=example
      - MONGO_URI=mongodb://root:example@mongo:27017/gateway-caas?authSource=admin
    depends_on:
      - redis
      - mongo

  # Redis service
  redis:
    image: redis:7.4.1
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass example
    volumes:
      - redis_data:/data

  # MongoDB service
  mongo:
    image: mongo:8.0
    ports:
      - "27017:27017"
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=example
    volumes:
      - mongo_data:/data/db

  # Loki service for log aggregation
  loki:
    image: grafana/loki:2.9.1
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml

  # Promtail service for log collection
  promtail:
    image: grafana/promtail:2.9.1
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml

  # Grafana service for visualization
  grafana:
    image: grafana/grafana:10.1.0
    ports:
      - "3001:3000"
    depends_on:
      - loki
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  redis_data:
  mongo_data:
  loki_data:
  grafana_data:
```

Notes:

	•	Replace the api service with your application’s configuration.
	•	Ensure the versions of images match your requirements.

### Step 2: Create Configuration Files

1. loki-config.yaml

Create a file named loki-config.yaml in the same directory as your docker-compose.yml:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9095

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 5m
  max_chunk_age: 1h
  chunk_retain_period: 30s
  wal:
    enabled: true
    dir: /tmp/loki/wal

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    shared_store: filesystem
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h
  filesystem:
    directory: /tmp/loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

compactor:
  working_directory: /tmp/loki/compactor
  shared_store: filesystem

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

```

2. promtail-config.yaml

Create a file named promtail-config.yaml:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      # Keep only containers with the label com.docker.compose.service=api
      - source_labels: [__meta_docker_container_name]
        regex: ".*api.*"
        action: keep

      # Add metadata labels for service name, container name, container id, and container image
      - source_labels: [__meta_docker_container_name]
        target_label: container
        regex: '/?(.*)'
      - source_labels: [__meta_docker_container_id]
        target_label: container_id
      - source_labels: [__meta_docker_container_image]
        target_label: container_image
      - source_labels: [__meta_docker_container_label_com_docker_compose_service]
        target_label: compose_service

      # Add static job label
      - target_label: job
        replacement: api_logs

      # Setup logs route
      - source_labels: [__meta_docker_container_id]
        target_label: __path__
        replacement: /var/lib/docker/containers/${1}/*-json.log
```

Notes:

	•	This configuration tells Promtail to:
	•	Monitor Docker containers whose names contain api.
	•	Parse logs as JSON and extract level and useCase as labels.
	•	Send logs to Loki at http://loki:3100.

### Step 3: Start the Services with Docker Compose

Open a terminal in your project’s directory and run:

docker-compose up -d

This command will build and start all the services defined in your docker-compose.yml file.

### Step 4: Access Grafana

Once the services are running:

	1.	Open your web browser and navigate to http://localhost:3001.
	2.	Log in with the default credentials:
	•	Username: admin
	•	Password: admin
	3.	You will be prompted to change the password. Choose a secure password.

### Step 5: Configure Loki as a Data Source in Grafana

	1.	In Grafana’s left-hand menu, click on the Configuration (gear icon) and select Data Sources.
	2.	Click on Add data source.
	3.	Select Loki from the list.
	4.	Configure the data source:
	•	Name: Loki (or any name you prefer).
	•	URL: http://loki:3100
	•	Leave other settings as default.
	5.	Click Save & Test. You should see a message saying Data source is working.

### Step 6: Create a Dashboard to View Logs

1. Create a New Dashboard

	•	Click on the Create (plus icon) in the left-hand menu and select Dashboard.

2. Add a New Panel

	•	Click on Add new panel.

3. Configure the Panel

	•	In the Query editor at the bottom, select your Loki data source.
	•	Enter the following query to retrieve logs from your api service:
```
{job="api_logs"}
```

	•	This query fetches logs labeled with `job="api_logs"`.

4. Customize the Visualization

	•	In the Visualization tab, select Logs to display logs in a readable format.
	•	Enable Show Labels to display labels like level and useCase.

5. Apply and Save the Dashboard

	•	Click Apply to save the panel.
	•	Click Save Dashboard (disk icon at the top right) and give your dashboard a name.

### Step 7: View and Analyze Logs

	•	Interact with your application to generate logs.
	•	In Grafana, you should see the logs appearing in real-time.
	•	Use the query editor to filter logs. For example:
	•	Filter by Level:
```
{job="api_logs", level="INFO"}
```

	•	Filter by Use Case:
```
{job="api_logs", useCase="ValidateApiKeyProject"}
```

	•	You can also parse JSON fields in the logs:
```
{job="api_logs"} | json
```

	•	This allows you to access all JSON fields in your logs for advanced querying.

## Additional Tips

1. Handling High Cardinality

	•	Be cautious when adding labels in Promtail. Only include labels with low cardinality (few unique values) to avoid performance issues.

2. Parsing Logs in Grafana

	•	Use LogQL operators like | json to parse and filter logs based on their content without adding extra labels.

3. Troubleshooting

	•	If logs are not appearing:
	•	Check Promtail and Loki logs for errors:
```
docker-compose logs promtail
docker-compose logs loki
```

	•	Ensure that the container names match the regex in promtail-config.yaml.

	•	Permissions Issues:
	•	If you’re on Windows or macOS, adjust the volume mounts in docker-compose.yml accordingly, as file paths may differ.

Conclusion

By following these steps, you have:

	•	Set up your application and necessary services with Docker Compose.
	•	Configured Loki and Promtail for log aggregation and collection.
	•	Visualized your logs in Grafana, enabling you to monitor and analyze your application’s behavior effectively.

Optional: Scaling and Customization

	•	Add More Services: Extend your docker-compose.yml to include additional services as needed.
	•	Customize Dashboards: Explore Grafana’s features to create customized dashboards and alerts.
	•	Security Considerations: Implement authentication and authorization for Grafana and other services in a production environment.

If you encounter any issues or have questions about specific steps, feel free to ask for further assistance!
