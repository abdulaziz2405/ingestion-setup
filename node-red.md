# Setting up a red-node
## 1. Run the systemd service
Prerequisites:
- You have installed docker engine installed via apt.
- You have a ready to use directory for node-red data which we will consider as `<data_directory>`

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
Reload the systemd:
```
systemctl daemon-reload
```

Start the service:
```
systemctl enable --now prod-ingestion-node-red.service
```

Execute the following command to create a hash password for admin user when the prompt for user input is out, enter the password and save the password.
```
docker exec -it node-red npx node-red admin hash-pw
```

## 2. Create users passwords and then place them in config as shown below (create 2 users):
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