# ArktIQ IT Monitoring Stack setup

The ArktIQ IT monitoring stack consists of a monitoring server and several monitored clients.

## Architectural overview

### Wireguard setup for safe shipping from remote servers across the internet
Both the monitor server and remote monitored clients run wireguard for safe shipping of metrics and logs from remote servers across the internet.

The monitor server runs on 10.100.0.1 and the monitored clients agent ships from 10.100.0.x. Wireguard isn't an absolute requirement to talk to the monitoring server, as monitored server within the internal network may be allowed to talk directly to the monitor server. Especially the monitoring agent of the monitor server itself.

### Monitor server stack
The monitoring server runs Grafana as the visual layer, Prometheus for metrics collection, Loki for log collection, Alertmanager for alerting and Blackbox for uptime monitoring. Additionally, there is a smtp-relay service for notification via e-mail. 

See docker-compose.yml and related config directories.

### Monitor agent
The monitoring clients run Grafana Alloy, which push metrics to Prometheus and logs to Loki. 

The clients run Alloy both on bare metal, to fetch host logs and metrics, and it runs as a docker container to fetch docker logs and docker metrics. So two alloy instances per remote server.


## Installation notes

### Start with the Wireguard installation

First, add wireguard on monitoring server.

Then, add wireguard on each monitored server, and add them as peers on the monitor wireguard.

Run on both Webserver and Monitoring server, install Wireguard and create keys

```
sudo apt update
sudo apt install wireguard -y
wg genkey | tee privatekey | wg pubkey > publickey
```

On the monitoring server, let's call the keys monitor-private-key, monitor-public-key
On the webserver, let's call the keys monitored-private-key, monitored-public-key

#### Configure the monitoring server

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.100.0.1/24
PrivateKey = <monitor-private-key>

[Peer]
PublicKey = <monitored-public-key>
Endpoint = <monitored-public-ip>:51821
AllowedIPs = 10.100.0.2/32
PersistentKeepalive = 25
```

#### Configure the monitored server

Create `/etc/wireguard/wg1.conf`:

```ini
[Interface]
Address = 10.100.0.2/24
ListenPort = 51821
PrivateKey = <monitored-private-key>

[Peer]
PublicKey = <monitor-public-key>
AllowedIPs = 10.100.0.1/32
```

Note, default wireguard port is 51820. I've put 51821 in this doc because I had another tunnel on the default port on the monitored server.

The tunnel will automatically spin up on server boot. If it fails, check that you allow outgoing UDP on port 51820 and/or 51821 to flow from your home lab to the VPS, and open for incoming traffic on UDP ports 51820 and/or 51821 in your VPS firewall.

Start wireguard like this:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Simply running `wg` will tell you if a handshake has been performed, which is a sign that the connection was successful, and you have a tunnel.


### Install the monitoring server

The monitoring server has been installed on a Ubuntu 24.04 Proxmox LXC, so adapt if needed. However, the setup is pretty straight forward, just install git and docker and get going.

Git:
```bash
sudo apt-get install git
```

Docker (https://docs.docker.com/engine/install/ubuntu/):
```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)

# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl status docker
```

You need to clone this repo first. If it were a private repo, log in and add your public key to github and use your private key to log in:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh -T git@github.com
```

Start the ssh agent, add your key and test the connection to github. If unsuccessful, verify that you use the correct key and that you've added the key to github. Clone the repo:

```bash
git clone git@github.com:ArktIQ-IT/monitoring-stack.git
```

Now enter the monitoring-stack directory, and create an `.env` file with the following content:

```env
DOCKER_TZ=Europe/Oslo
NOTIFICATION_EMAIL=

# Gmail email account for SMTP relay
GMAIL_USERNAME=
GMAIL_PASSWORD=

# Grafana config
GRAFANA_ADMIN_USER=
GRAFANA_ADMIN_PASSWORD=
GRAFANA_DOMAIN=
```

This setup assumes using a simple docker container relaying email through gmail, but of course, you'd set up a relay through some email service for your business.

The setup also assumes you have a proxy/ingress, for example Traefik, somewhere that will use your Grafana server as its backend. Don't expose Grafana directly to the internet like this!

Additionally, keep in mind the following:

 - An SMTP relay running on port 25 needs extra permissions. On proxmox add the following to the LXC config:

`vi /etc/pve/lxc/101.conf`

Add:
```yaml
unprivileged: 0
lxc.apparmor.profile: unconfined
lxc.cap.drop:
```

This requires a restart.

Finally, you should check out your alertmanager settings (configs) and prometheus settings (configs) to set email settings and websites being monitored.


Now, just `docker compose up -d` and your monitoring server spins up. Check logs for errors.


## Setting up monitored servers

We're setting up Grafana Alloy to monitor servers. We use a unified installation in Docker, with one alloy instance to monitor both docker containers and the host system.

### Unified alloy installation

Install once in docker, and scrape host and docker metrics. No need to install on bare metal.

Simply create a directory for your docker-compose file, and place the config.alloy-file in a suitable directory.

Then run `docker compose up -d`. Of course, you will need to have set up the wireguard tunnel first (see above).

With this unified setup, you can tag containers with two additional labels (service + env) to make them easier to group and find.

E.g.:

```yaml
services:
  api:
    image: ghcr.io/arktiq/api:latest
    labels:
      no.arktiq.service: api
      no.arktiq.env: prod
```

Each log line will automatically get this in Loki:

```logql
job="arktiq/docker"
instance="ubuntu-private-server"
service="api"
env="prod"
container="api-1"
```

Docker metrics are still only labeled with:
```logql
job="arktiq/docker"
instance="ubuntu-private-server"
```
But remember, that cAdvisor will still output labels for filtering such as image and name to filter on.


### Recommended Grafana Dashboards

Import these dashboards in Grafana for comprehensive monitoring:

**Host/Node Monitoring:**
- **Node Exporter Full** (ID: 1860) - Comprehensive host metrics dashboard
  - URL: https://grafana.com/grafana/dashboards/1860
  - Shows CPU, memory, disk, network, and system load
  - I've included an adapted version of this dashboard in the `grafana/provisioning/dashboards/` directory

**Docker Container Monitoring:**
- **Docker Container & Host Metrics** (ID: 179) - Docker container metrics
  - URL: https://grafana.com/grafana/dashboards/179
  - Shows container CPU, memory, network, and disk usage
  - I've included an adapted version of this dashboard in the `grafana/provisioning/dashboards/` directory

- **Docker Monitoring** (ID: 893) - Alternative Docker dashboard
  - URL: https://grafana.com/grafana/dashboards/893
  - I've included an adapted version of this dashboard in the `grafana/provisioning/dashboards/` directory


**Logs:**
- Use Grafana's built-in Explore feature with Loki datasource
- Filter logs by:
  - `{node="hostname"}` - View logs from a specific host
  - `{log_type="docker"}` - View only Docker container logs
  - `{log_type="host"}` - View only host/system logs
  - `{compose_service="service-name"}` - View logs from a specific Docker Compose service
  - `{container_name="container-name"}` - View logs from a specific container

**Example Log Queries:**
- All logs from a specific host: `{node="server1"}`
- Docker logs from a specific service: `{log_type="docker", compose_service="web"}`
- System errors: `{log_type="host", severity="err"}`
- Kernel logs: `{log_type="host", unit="kernel"}`

### Custom Dashboard Provisioning

Custom dashboards can be automatically provisioned by placing JSON exports in the `grafana/provisioning/dashboards/` directory.

The dashboards will be automatically imported and available in Grafana on startup. This ensures your custom dashboards are included when setting up a new monitoring server.

**Note:** The dashboard provisioning is configured in `grafana/provisioning/dashboards/dashboards.yml`. See `grafana/provisioning/dashboards/README.md` for more details.

### Alerting

The following alerts are configured in Prometheus:
- **InstanceDown** - Critical alert for when instances are down
- **DiskUsageHigh**: Triggers when disk usage exceeds 80% for 5 minutes
- **CPUUsageHigh**: Triggers when CPU usage exceeds 90% for 5 minutes
- **MemoryUsageHigh**: Triggers when memory usage exceeds 90% for 5 minutes
- **ContainerNetworkIOHigh**: Triggers when a container's network IO exceeds 5 Mbps for 5 minutes
- **WebsiteDown** - Critical alert for when websites are unreachable
