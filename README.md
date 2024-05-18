# prometheus-tls-encryption
## Node Exporter Configuration
#### Create a `self signed certificate` and a `Hashed Password`
```bash
openssl req   -x509   -newkey rsa:4096   -nodes   -keyout node_exporter.key   -out node_exporter.crt
htpasswd -nbB farag password
```
#### Create a configuration file for the node exporter
```bash
mkdir /etc/node_exporter
cp ~/node_exporter.key ~/node_exporter.crt /etc/node_exporter
cat << EOF > /etc/node_exporter/config.yml 
tls_server_config:
  cert_file: node_exporter.crt
  key_file: node_exporter.key
basic_auth_users:
  prometheus:$2y$05$br2UJv0wAnSmuDcPr/myBONvKOtAcKpYspKFKpz71RuS4BOVTkZ7
EOF
chown -R node_exporter:node_exporter /etc/node_exporter/
```
#### To persist changes add the changes to the service file and restart the service
```bash
cat << EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
--web.config.file="/etc/node_exporter/config.yaml"
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl restart node_exporter
```
## Prometheus Server
#### First copy the certificate from the node exporter
```bash
scp farag@192.168.76.129:/etc/node_exporter/node_exporter.crt /etc/prometheus
```
#### Add `HTTPS` and `Basic Authentication` configuration to `prometheus.yaml` file
```bash
cat << EOF > /etc/prometheus/prometheus.yml
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  external_labels:
      monitor: 'example'
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']
rule_files:
  - "/etc/prometheus/rules/*"
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    scrape_timeout: 5s
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    basic_auth:
      username: prometheus
      password: password
    scheme: 'https'
    tls_config:
      ca_file: /etc/prometheus/node_exporter.crt
      insecure_skip_verify: true
    static_configs:
      - targets: ['192.168.76.129:9100']
EOF
systemctl restart prometheus
```
## Testing
#### Target lists at the server
![image](https://github.com/abdo14m1/prometheus-tls-encryption/assets/154431880/7c9850aa-e4bd-4635-aa6f-72fc2994ce7f)
#### curl the metrics from node
![image](https://github.com/abdo14m1/prometheus-tls-encryption/assets/154431880/add99237-9b12-40da-bbdc-4ae2d490407b)


