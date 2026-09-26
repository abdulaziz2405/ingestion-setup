
# Prerequisites
- We recommend Ubuntu 24.04 OS
- Extra disk with minimum of 5GB size and mounted at `/mnt/data`



# Setup
## Add repo and install InfluxDB
```bash
# Add key
curl --silent --location -O https://repos.influxdata.com/influxdata-archive.key
gpg --show-keys --with-fingerprint --with-colons ./influxdata-archive.key 2>&1 | grep -q '^fpr:\+24C975CBA61A024EE1B631787C3D57159FC2F927:$'

# Add repo
cat influxdata-archive.key | gpg --dearmor | sudo tee /etc/apt/keyrings/influxdata-archive.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/keyrings/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main'
sudo tee /etc/apt/sources.list.d/influxdata.list

# Install influxdb
sudo apt-get update
sudo apt-get install influxdb2=2.7.11-1
```

## Setup environment
Copy original data folder
```bash
cp -rp /var/lib/influxdb /mnt/data/
```

Check if permissions and owners are correct:
```
root@<SERVER>:~# ls -laXh /mnt/data/influxdb/
total 16K
drwxr-xr-x 4 influxdb influxdb 4.0K <DATE> <TIME> .
drwxr-xr-x 6 root     root     4.0K <DATE> <TIME> ..
```

Replace data directory from `/var/lib/influxdb` to `/mnt/data/influxdb` in config file (`/etc/influxdb/config.toml`):
```toml
# bolt-path = "/var/lib/influxdb/influxd.bolt"
# engine-path = "/var/lib/influxdb/engine"

bolt-path = "/mnt/data/influxdb/bolt/influxd.bolt"
engine-path = "/mnt/data/influxdb/engine"
```


## Start service
Start service
```bash
systemctl daemon-reload
systemctl enable --now influxdb.service
```

Now InfluxDB should be available in your server at port 8086 (http://localhost:8086 or http://<SERVER_IP>:8086). You can continue setup in that page
