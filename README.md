# Cisco Router Monitoring with SNMP Exporter, Prometheus and Grafana

A GNS3 lab that monitors interface traffic on a Cisco 7200 router.
Prometheus scrapes the router through **SNMP Exporter**, and **Grafana** visualizes the traffic in real time.

## Architecture

```
┌──────────────┐  SNMP v2c (UDP 161)  ┌────────────────────── Ubuntu / Docker ──────────────────────┐
│ Cisco 7200   │◄─────────────────────┤  snmp-exporter :9116  ◄──  Prometheus :9090  ◄──  Grafana :3000 │
│ R1           │                      │                                                              │
│ 192.168.127.133                     │                       192.168.127.129                        │
└──────────────┘                      └──────────────────────────────────────────────────────────────┘
```

| Component | Version | Port | Role |
|---|---|---|---|
| Cisco 7200 (R1) | IOS | UDP 161 | SNMP agent |
| SNMP Exporter | `prom/snmp-exporter:v0.28.0` | 9116 | Translates SNMP to Prometheus metrics |
| Prometheus | `prom/prometheus:v3.2.1` | 9090 | Scrapes and stores metrics |
| Grafana | `grafana/grafana:11.5.2` | 3000 | Dashboard |

## Lab topology (GNS3)

```
Cloud1 (eth1) ── Switch1 ── R1 f0/0 ─── R1 f1/1 ── ipterm-3
                   │          │ f1/0
                   └── R2 f0/0 ─ R2 f1/0
                                R2 f1/1 ── ipterm-2
```

| Device | Interface | IP address |
|---|---|---|
| R1 | FastEthernet0/0 | 192.168.127.133/24 |
| R1 | FastEthernet1/0 | 10.12.0.1 |
| R1 | FastEthernet1/1 | 172.16.0.1 |
| R1 | Loopback0 | 1.1.1.1 |
| Ubuntu | ens33 | 192.168.127.129/24 |

![GNS3 topology](Lab topology (GNS3).png)

Cloud1 uses **eth1 of the GNS3 VM**, which must be connected to **VMware NAT (VMnet8)**, the same network as the Ubuntu VM.

## 1. Cisco router configuration

```
enable
configure terminal
snmp-server community public RO
end
write memory
```

Verify:

```
show running-config | include snmp
ping 192.168.127.129
```

## 2. Docker stack

`docker-compose.yml`

```yaml
services:
  snmp-exporter:
    image: prom/snmp-exporter:v0.28.0
    container_name: snmp-exporter
    ports:
      - "9116:9116"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v3.2.1
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    depends_on:
      - snmp-exporter
    restart: unless-stopped

  grafana:
    image: grafana/grafana:11.5.2
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
```

`prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'snmp'
    metrics_path: /snmp
    params:
      module: [if_mib]
      auth: [public_v2]
    static_configs:
      - targets: ['192.168.127.133']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
```

Start the stack:

```bash
sudo docker compose up -d
sudo docker compose ps
```

## 3. Test SNMP Exporter with curl

```bash
curl -G "http://localhost:9116/snmp" \
  --data-urlencode "target=192.168.127.133" \
  --data-urlencode "module=if_mib" \
  --data-urlencode "auth=public_v2" \
  | grep -E '^if(In|Out)Octets'
```

Expected output (one line per interface):

```
ifInOctets{ifDescr="FastEthernet0/0",ifIndex="1",ifName="Fa0/0"} ...
ifOutOctets{ifDescr="FastEthernet0/0",ifIndex="1",ifName="Fa0/0"} ...
```

## 4. Prometheus (PromQL)

Open `http://192.168.127.129:9090` and check **Status → Targets**: the `snmp` job should be **UP**.

Inbound traffic (bits per second):

```promql
rate(ifInOctets{instance="192.168.127.133"}[1m]) * 8
```

Outbound traffic (bits per second):

```promql
rate(ifOutOctets{instance="192.168.127.133"}[1m]) * 8
```

Single interface in Mbps:

```promql
rate(ifInOctets{instance="192.168.127.133",ifIndex="3"}[1m]) * 8 / 1000000
```

> Prometheus may show *"metric might not be a counter"*. This is only a naming notice: `ifInOctets` is an SNMP counter, so `rate()` is correct.

## 5. Grafana dashboard

1. Open `http://192.168.127.129:3000` and log in.
2. **Connections → Data sources → Add data source → Prometheus**, URL `http://prometheus:9090`, then **Save & test**.
3. **Dashboards → New → Add visualization**, select Prometheus, and add:
   - Query A: `rate(ifInOctets{instance="192.168.127.133"}[1m]) * 8`, legend `In {{ifName}}`
   - Query B: `rate(ifOutOctets{instance="192.168.127.133"}[1m]) * 8`, legend `Out {{ifName}}`
4. Set **Unit → Data rate → bits/sec** and save the dashboard.

Generate traffic from R1 so the graph has data:

```
ping 192.168.127.129 repeat 100000 size 1400
```

## Screenshots

### Prometheus (PromQL)

`rate(ifInOctets{instance="192.168.127.133"}[1m]) * 8`

![Prometheus PromQL](images/prometheus.png)

### Grafana graph

![Grafana graph]Grafana.png)

## Troubleshooting

| Problem | Check |
|---|---|
| curl returns nothing | `ping 192.168.127.133` from Ubuntu, SNMP community on R1, try `auth=public` |
| Target is DOWN in Prometheus | Target IP in `prometheus.yml`, then `sudo docker restart prometheus` |
| R1 cannot ping Ubuntu (ARP `FAILED`) | GNS3 VM Network Adapter 2 must be **NAT (VMnet8)** and connected; restart GNS3 and the GNS3 VM |
| Graph is flat | Generate traffic with `ping ... repeat 100000` |
| Forgot Grafana password | `sudo docker exec -it grafana grafana cli admin reset-admin-password admin` |
