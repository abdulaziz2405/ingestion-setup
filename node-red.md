# Prerequisites:
- You have installed docker engine installed via apt in your system.
- You have a ready to use directory for node-red data which we will consider as `<data_directory>`
# Setting up a node-red
### 1. Run the systemd service

Run the node-red via systemd placing the systemd service in `/etc/systemd/system/prod-ingestion-node-red.service` or any other service filename that is convenient for your usage.
```
[Unit]
Description=Docker Service for node-red
After=network.service docker.service
Requires=docker.service

[Service]
User=root
Group=root
Type=simple
ExecStartPre=-/usr/bin/docker stop -t 60 node-red
ExecStartPre=-/usr/bin/docker rm node-red
ExecStart=/usr/bin/docker run \
    --name node-red \
    -p 1880:1880 \
    -v <data_directory>:/data \
    nodered/node-red

ExecStop=-/usr/bin/docker stop -t 60 node-red
ExecReload=/usr/bin/docker restart 'node-red'

Restart=always
RestartSec=20s

SuccessExitStatus=SIGKILL SIGTERM 143 137

[Install]
WantedBy=multi-user.target
WantedBy=docker.service
```
Place the ready to use pipelines stored in this repository `configs/node-red.tar.gz` to `<data_directory>`:
```
tar -xzf configs/node-red.tar.gz <data_directory>
```

Reload the systemd:
```
systemctl daemon-reload
```


Start the service:
```
systemctl enable --now prod-ingestion-node-red.service
```

### 2. Create users passwords and then place them in config as shown below (create 2 users):
Execute the following command to create a hash password for admin user when the prompt for user input is out, enter the password and save the password.
```
docker exec -it node-red npx node-red admin hash-pw
```
Place the saved password to the following configuration file which should be placed at <data_directory>/settings.js:
```
module.exports = {
    adminAuth: {
        "type": "credentials",
        "users": [
            {
                "username": "admin",
                "password": "<your-password-hash>",
                "permissions": "*"
            }
        ]
    }
};
```
Restart the service:
```
systemctl restart prod-ingestion-node-red.service
```

### 3. Configuring node-red

Log in to the node-red ui in port 1880 with admin credentials that were generated in step 2.


#### Mosquitto:
Go to Prod-Influx tab and double tap on meteodb. The config window will pop up. It is required to configure mosquitto credentials for node-red to authenticate.

It is possible to set mosquitto admin password via:
```
docker exec -it <mosquitto-container-name> mosquitto_passwd -c /mosquitto/config/passwd admin
```
#### InfluxDB:

First, the token should be generated via influx cli:
```
influx auth create \
  --org amudario \
  --all-access \
  --description "node-red-token" \
  --host http://localhost:8086 \
  -t <operator-token-here>
```
Copy the `TOKEN` field and navigate to the ui. In the Prod-Influx tab find prod-influx block and double tap it. Now you are able to configure the endpoint and set the token itself.