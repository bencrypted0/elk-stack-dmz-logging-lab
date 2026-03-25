# ELK Stack Performance Monitoring Lab

![](assets/elk_banner.png)

Security is only as good as your visibility. You can have the most hardened firewall rules and the most carefully segmented network, but if you have no insight into what's actually flowing through it, you're flying blind. That's the gap a SIEM fills — and in this guide, we're going to fill it ourselves, from scratch.

We're going to build a fully functional log monitoring pipeline using the ELK Stack — Elasticsearch, Logstash, and Kibana — deployed across a segmented lab network with pfSense as the firewall and an intentionally vulnerable web application as our monitoring target. No managed services, no shortcuts. Every component is configured by hand so you understand exactly what each piece does and why it's there.

By the end of this guide you'll have a three-zone network routing live log data from a DMZ web server into a centralized Kibana dashboard — the same architectural pattern used in real SOC environments.

## Setting Up pfSense as Our Network Gateway

Before any log data can flow into our ELK stack, we need a network that actually separates and routes that data correctly. pfSense is going to serve as our firewall and router, and giving us full visibility into traffic between zones.

>  To set up pfSense and DMZ , follow this guide first: **[pfSense & DMZ Setup](https://medium.com/@bennetsharwin001/how-to-simulate-a-dmz-demilitarized-zone-network-with-virtualbox-and-pfsense-firewall-b36aea158d3c)** 

---

### A Key Difference From the Standard Setup

If you've followed a typical pfSense + DMZ guide, you'll have seen a three-zone layout: WAN, DMZ, and an Internal LAN. We're going to make **one deliberate change** to that architecture.

Instead of a general-purpose Internal LAN, we're introducing a dedicated **Logging Network**. This is an isolated zone whose sole purpose is to receive log data forwarded from the DMZ and hand it off to Logstash. Keeping logs on their own network segment means:

- Log traffic never competes with or gets mixed into production traffic
- If a DMZ host is compromised, it can't pivot directly into your analysis infrastructure
- It mirrors how real SOC environments isolate their log collection pipeline

Think of it as a one-way mirror — the DMZ can send data _into_ the logging network, but that's where the relationship ends.

---

### Renaming the Interface

Assuming you already have devices in your DMZ and your existing internal interface configured, the first step is a simple but important one — renaming the `Internal LAN` interface to `Logging` to reflect its new purpose.

1. Log in to your **pfSense web interface**
2. Navigate to **Interfaces → [Your Internal LAN interface]**
3. Change the **Description** field from `LAN` to `Logging`
4. Hit **Save**, then **Apply Changes**

Now you should have something like this:

![](assets/How%20to%20Set%20Up%20ELK%20Stack%20for%20performance%20Monitoring-20260320151631715.png)

This is a small change, but naming your interfaces accurately pays off the moment you're staring at firewall rules at 2am trying to figure out why logs aren't arriving.

---

## Deploying the Target: DVWA

A SOC without something to monitor is just infrastructure. We need a target — something running in our DMZ that generates realistic, attack-worthy traffic. For this lab, that target is **DVWA**.

DVWA is an intentionally vulnerable web application built specifically for security training. It's riddled with real-world vulnerabilities — SQL injection, XSS, broken authentication, and more — which makes it perfect for our purposes. We're not just setting up a dummy service; we're deploying something we can actively attack, so we have meaningful data flowing into our SIEM.

---

### The Architecture: Why Nginx Lives on the Host

Rather than running everything in Docker, we're making a deliberate split:

- **DVWA** runs inside a Docker container, exposed on port `3000`
- **Nginx** is installed directly on the host system, not in Docker

The reason comes down to log access. When Nginx runs inside a container, Filebeat has to reach into that container to tail the log files — adding complexity and a layer of indirection. When Nginx is installed on the host, its logs land at `/var/log/nginx/` like any standard system service, and Filebeat can watch them directly with no extra configuration. It's a cleaner, more reliable setup that mirrors how a real environment would be instrumented.

Nginx acts as the reverse proxy, sitting on port `80` and forwarding all traffic to DVWA on port `3000`. Every request that hits the server gets logged by Nginx — that's the data Filebeat will be shipping to Logstash.

---

### Step 1 — Run DVWA in Docker

Create a `docker-compose.yml` on the DMZ server:

```yaml
services:
  dvwa:
    image: ghcr.io/digininja/dvwa:latest
    container_name: dvwa
    restart: always
    ports:
      - "127.0.0.1:3000:80"
```

DVWA's container exposes port `80` internally — we map it to `3000` on the host so the Nginx config needs no changes from the DVWA setup. The bind to `127.0.0.1:3000` means DVWA is only reachable locally on the host, not directly from the network. Nginx remains the only intended entry point.

Bring it up:

```bash
docker compose up -d
```

---

### Step 2 — Install Nginx on the Host

```bash
sudo apt update && sudo apt install nginx -y
```

---

### Step 3 — Configure Nginx as a Reverse Proxy

Replace the default config at `/etc/nginx/sites-available/default` with the following, or create a new site config:

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    }
}
```

`server_name _;` is a catch-all — it matches any hostname, which is fine for a lab. The `X-Real-IP` and `X-Forwarded-For` headers are important: they ensure the actual client IP is preserved in the log, rather than just showing `127.0.0.1` for every request.

Test the config and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### What This Gives Us

Once both are running, any request hitting `http://<DMZ-IP>:80` lands on DVWA via Nginx. Every one of those requests gets written to `/var/log/nginx/access.log` on the host — exactly where Filebeat will be watching. The log pipeline is ready to be connected.

---

## Setting Up ELK in the Logging Network

If you've followed the previous steps, you should have a VM sitting on the Logging Network — the isolated zone we renamed from LAN earlier. That VM is going to be the home of our entire ELK stack.

### Why Docker Compose?

We're deploying ELK using Docker Compose rather than installing each component directly on the host. The reasons are practical:

- **Speed:** The entire stack spins up with a single command — `docker compose up -d`
- **Isolation:** Each component runs in its own container, with no dependency conflicts on the host OS
- **Reproducibility:** If something breaks or you want to start fresh, `docker compose down -v` tears everything down cleanly and you can rebuild in minutes
- **Portability:** The same `docker-compose.yml` works on any machine, making it easy to share and version-control

---

### The Docker Compose File

This is the core of the deployment. Three services — Elasticsearch, Logstash, and Kibana — all on a shared internal Docker network called `elk`.

```yaml
version: '3'
services:
  elasticsearch:
    image: elasticsearch:7.9.1
    container_name: elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - test_data:/usr/share/elasticsearch/data/
      - ./elk-config/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      - discovery.type=single-node
      - http.host=0.0.0.0
      - transport.host=0.0.0.0
      - xpack.security.enabled=false
      - xpack.monitoring.enabled=false
      - cluster.name=elasticsearch
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    mem_limit: 1GB
    networks:
      - elk

  logstash:
    image: logstash:7.9.1
    container_name: logstash
    ports:
      - "5044:5044"
      - "9600:9600"
      - "5140:5140/udp"
    volumes:
      - ./elk-config/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./elk-config/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./elk-config/logstash/patterns:/usr/share/logstash/patterns
      - ls_data:/usr/share/logstash/data
    environment:
      - LS_JAVA_OPTS=-Xms512m -Xmx512m
    mem_limit: 1GB
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:7.9.1
    container_name: kibana
    ports:
      - "5601:5601"
    volumes:
      - ./elk-config/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml
      - kb_data:/usr/share/kibana/data
    mem_limit: 1GB
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  test_data:
  ls_data:
  kb_data:
```

A few things worth understanding before you just run it:

- **All three services are pinned to version 7.9.1.** Mixing versions across Elasticsearch, Logstash, and Kibana is a common source of silent failures — always keep them in sync.
- **`discovery.type=single-node`** tells Elasticsearch not to look for cluster peers. This is essential for a single-VM lab setup — without it, the node will sit waiting to form a cluster and never become healthy.
- **`xpack.security.enabled=false`** disables authentication for simplicity in the lab environment. In a production deployment, you would never do this.
- **`ES_JAVA_OPTS=-Xms512m -Xmx512m`** and **`mem_limit: 1GB`** keep each container's memory footprint bounded. ELK is memory-hungry — without these limits, Elasticsearch will happily consume all available RAM.
- **`depends_on`** ensures Logstash and Kibana wait for Elasticsearch to start first, since both need it to be reachable before they initialize.

---

### The Configuration Files

Each service gets its own config file, mounted into the container via the `volumes` block. Here's what each one does:

**`elasticsearch.yml`**

```yaml
cluster.name: "elasticsearch"
network.host: localhost
```

Minimal config — just sets the cluster name and binds to localhost inside the container. The `docker-compose.yml` environment variables handle the rest.

---

**`logstash.conf`**

```
input {
  beats {
    port => 5044
  }
}

filter {
}

output {
  elasticsearch {
    hosts => "http://elasticsearch:9200"
    index => "filebeat-test%{+YYYY.MM.DD}"
    user => "elastic"
    password => "password"
  }
}
```

This is the Logstash pipeline. Right now the `filter {}` block is empty — we're getting the plumbing connected first. In the next section we'll add Grok patterns here to parse the Nginx and pfSense logs into structured fields. Notice that the output references `elasticsearch` by container name — Docker's internal DNS resolves this automatically within the `elk` network.

---

**`logstash.yml`**

```yaml
http.host: 0.0.0.0
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
```

Binds the Logstash HTTP API to all interfaces (so we can hit the monitoring endpoint) and points the monitoring module at our Elasticsearch container.

---

**`kibana.yml`**

```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true
```

Tells Kibana where to find Elasticsearch and enables the container monitoring UI. `server.host: "0"` binds Kibana to all interfaces so it's reachable from your browser outside the container.

---

### Bringing It Up

With all config files in place under `./elk-config/`, run:

```bash
docker compose up -d
```

Give it 30–60 seconds for Elasticsearch to fully initialize, then navigate to:

```
http://<logging-vm-ip>:5601
```

You should be greeted by the Kibana home dashboard — the analyst's interface for everything we're about to build. If Kibana shows a connection error on first load, wait another 30 seconds and refresh; Elasticsearch sometimes takes a moment longer than Kibana expects.

---

Here's the content for that section — kept tight:

---

## Verifying the Stack is Healthy

Before touching any configuration, let's confirm all three services are up and talking to each other. Run through these checks top to bottom.

### **1 — All containers running**

```bash
docker ps
```

You should see `elasticsearch`, `logstash`, and `kibana` all with status `Up`. If any show `Exited`, check its logs with `docker logs <container_name>`.

### **2 — Elasticsearch is responding**

```bash
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

The first command should return a JSON response with `"You Know, for Search"`. The second gives you cluster health — `green` or `yellow` are both fine for a single-node setup. `red` means something is wrong.

### **3 — Kibana is reachable**

Open `http://localhost:5601` in a browser on the Logging VM. If the Kibana UI loads, it has successfully connected to Elasticsearch on its own.

### **4 — Logstash is connected**

```bash
curl http://localhost:9600?pretty
docker logs logstash
```

The API should return `"status": "green"`. In the logs, look for `Successfully started Logstash API endpoint` and `Pipeline started successfully`. If you see `Connection refused`, there's an issue in your `logstash.conf` pipeline.

### **5 — Config files are mounted**

```bash
ls ./elk-config/elasticsearch/
ls ./elk-config/logstash/
ls ./elk-config/kibana/
```

If any directory is empty or missing, the container will either fail silently or fall back to defaults — which will cause subtle issues later.
Once all five checks pass, the stack is healthy and ready for configuration.

---

## Setting Up Filebeat on the Web Server

With Nginx writing logs on the DMZ host and ELK running on the Logging VM, the missing piece is the bridge between them — **Filebeat**. It runs on the DMZ server, watches the Nginx log files, and ships every new line directly to Elasticsearch.

---

### Installing Filebeat via Kibana

Kibana actually walks you through the Filebeat setup with a built-in tutorial. On the Logging VM, open:

```
http://localhost:5601/app/home#/tutorial/nginxLogs
```

This page gives you the exact installation commands for your OS, a pre-built Nginx module configuration, and instructions for loading the default Nginx dashboards into Kibana. Follow the steps there to install Filebeat on the **DMZ VM** — not the Logging VM.

---

### Two Important Changes to `filebeat.yml`

The tutorial will generate a `filebeat.yml` with some defaults that need adjusting before anything works correctly.

**1 — Send directly to Elasticsearch, not Logstash**

By default the tutorial points output at Logstash. We're bypassing Logstash here and shipping directly to Elasticsearch — this gives the Nginx module proper control over index mappings and field parsing, which is what makes those pre-built dashboards actually populate correctly.

In `filebeat.yml`, comment out the Logstash output and uncomment the Elasticsearch output:

```yaml
# Comment this out:
# output.logstash:
#   hosts: ["<logging-vm-ip>:5044"]

# Uncomment and configure this:
output.elasticsearch:
  hosts: ["<logging-vm-ip>:9200"]
```

> **Firewall Rule Required:** For Filebeat on the DMZ VM to reach Elasticsearch, you need a pfSense firewall rule allowing traffic from the DMZ network to port `9200` on the Logging VM. Without this, the connection will be silently dropped at the network level regardless of how `filebeat.yml` is configured.

**2 — Skip the credentials**

The tutorial will also include fields for an Elasticsearch username and password. Leave those out. When we set up the ELK stack, we explicitly disabled authentication in the Docker Compose file:

```yaml
environment:
  - xpack.security.enabled=false
```

This turns off all authentication for Elasticsearch entirely — no username, no password, no TLS. With security disabled, adding credentials to `filebeat.yml` won't just be unnecessary, it can cause connection errors.

---

### Troubleshooting: Logs Appearing in Error Log Instead of Access Log

If you open the Kibana dashboard and notice that GET requests and `200`/`304` status codes are showing up under the **Nginx error logs** panel instead of access logs, the Nginx module is misrouting them — it's picking up the wrong file for the wrong log type.

Check your Nginx module config on the DMZ VM:

```bash
sudo nano /etc/filebeat/modules.d/nginx.yml
```

Make sure the paths are explicitly set for both log types:

```yaml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
  error:
    enabled: true
    var.paths: ["/var/log/nginx/error.log*"]
```

Without explicit paths, Filebeat's auto-detection can assign the access log to the error log input. After saving the file, restart Filebeat:


```bash
sudo systemctl restart filebeat
```

---

### A Note on Security — Worth Documenting

Disabling `xpack.security` is a deliberate shortcut for this lab, and it's worth being honest about what that means:

- Anyone who can reach port `9200` on the Logging VM can freely read, write, or delete your Elasticsearch data
- There is no audit trail of who accessed what
- All data between Filebeat and Elasticsearch travels unencrypted

For a portfolio project, this is actually an opportunity rather than a weakness. Note in your README that security was intentionally disabled for lab simplicity. If you are doing this in production, research what hardening looks like — enabling `xpack.security`, setting up TLS certificates between components, and creating role-based users for Kibana and any log shippers.

---

### Loading the Pre-Built Nginx Dashboards

Once Filebeat is installed and configured, run this on the DMZ VM to import the default Kibana dashboards that ship with the Nginx module:

```bash
sudo filebeat setup --dashboards
```

This connects to Kibana and automatically pulls in a set of pre-built dashboards. Once it completes, head over to the Kibana UI on the Logging VM:
```
http://localhost:5601
````

Navigate to **Kibana → Dashboard** and search for `nginx`. You should see:

- `[Filebeat Nginx] Access and Error Logs`
- `[Filebeat Nginx] Overview`

If both dashboards appear, Filebeat has a clean connection to Kibana and the Nginx module is loaded correctly. At this point, any traffic hitting DVWA  through Nginx will start populating these dashboards automatically.

Now to get the dashboard views correctly you need to refresh the index patterns, to do that Go to: **Kibana → Management → Index Patterns → filebeat-***
Click the **refresh** button (🔄) in the top right of the index pattern page. This rescans the index and picks up any new fields that have been created since the index pattern was last updated.

Then you would have something like this:

![](assets/How%20to%20Set%20Up%20ELK%20Stack%20for%20performance%20Monitoring-20260322182608653.png)


## Conclusion

We've gone from a blank network to a fully operational log monitoring pipeline. pfSense is segmenting the traffic, DVWA is running behind an Nginx reverse proxy in the DMZ, Filebeat is shipping access logs across network zones, and Kibana is indexing and visualizing everything in real time.

More importantly, you now understand every layer of that pipeline — not just that it works, but _why_ it's built the way it is. The dedicated Logging Network, the host-level Nginx installation for cleaner log access, the direct Filebeat-to-Elasticsearch output for proper ECS field mapping — each of these was a deliberate decision, and that kind of architectural thinking is what separates someone who followed a tutorial from someone who can build and operate this infrastructure independently.

This is a strong foundation. In the next article, we'll move from monitoring to detection — writing Kibana Watcher rules that fire alerts when attack patterns hit DVWA, and routing those alerts into Slack with MITRE ATT&CK context attached.