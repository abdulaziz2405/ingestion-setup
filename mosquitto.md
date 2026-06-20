
Create data directory
```bash
mkdir -p /mnt/data/mosquitto/{config,data,log}
```


Create mosquitto service in `/etc/systemd/system/prod-ingestion-mosquitto-01.service`:
```systemd
[Unit]
Description=Docker Service for mosquitto
After=network.service docker.service
Requires=docker.service

[Service]
ExecStartPre=-/usr/bin/docker stop -t 60 mosquitto
ExecStartPre=-/usr/bin/docker rm mosquitto
ExecStart=/usr/bin/docker run \
    --name mosquitto \
    -p 1883:1883 \
    -p 9001:9001 \
    -v /mnt/data/mosquitto/config:/mosquitto/config \
    -v /mnt/data/mosquitto/data:/mosquitto/data \
    -v /mnt/data/mosquitto/log:/mosquitto/log \
    eclipse-mosquitto:1.6.11

ExecStop=-/usr/bin/docker stop -t 60 mosquitto
ExecReload=/usr/bin/docker restart mosquitto

Restart=always
RestartSec=20s

SuccessExitStatus=SIGKILL SIGTERM 143 137

[Install]
WantedBy=multi-user.target
WantedBy=docker.service
```


Start service:
```bash
systemctl daemon-reload
systemctl enable --now prod-ingestion-mosquitto-01.service
```


Check logs of mosquitto
```bash
tail -f /mnt/data/mosquitto/log/mosquitto.log
```

Mosquitto is working if you see logs like this:
```
1781877274: Opening ipv4 listen socket on port 1883.
1781877274: Opening ipv6 listen socket on port 1883.
1781877274: mosquitto version 1.6.11 running
```
