# Install GrayLog and Grafana to monitor Pfsense
Grafical monitoring the Pfsense firewall (PfsenseGG)

### Prerequisites
- Centos 7 minimal 
- pfSense v2.4.4+ 


# Preparation

### 1. Install Graylog
Install Graylog according to the manufacturers specifications as described in the following link:
http://docs.graylog.org/en/3.1/pages/installation/os/centos.html

Edit “/etc/graylog/server/server.conf”  and change the root_timzone according to your needs. In my case I had to change 
form UTC to Europe/Berlin:
```
  root_timezone = Europe/Berlin 
```

### 2. Download and install Cerebro
```
wget https://github.com/lmenezes/cerebro/releases/download/v0.8.3/cerebro-0.8.3-1.noarch.rpm
yum localinstall cerebro-0.8.3-1.noarch.rpm
```
Edit “/usr/lib/systemd/system/cerebro.service”  and change the default port 9000 of CEREBRO to port 9001, 
because GrayLog is already running on this port.
Change the line:
```
  ExecStart=/usr/share/cerebro/bin/cerebro 
```
To:
```
  ExecStart=/usr/share/cerebro/bin/cerebro -Dhttp.port=9001
```
```
systemctl enable cerebro
firewall-cmd --zone=public --add-port=9001/tcp --permanent
firewall-cmd --reload
systemctl start cerebro
```
### 3. Download and install Grafana
Create a repository file for Grafana.
```
cat <<EOF | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```
```
yum -y install grafana
systemctl enable grafana-server
firewall-cmd --add-port=3000/tcp --permanent
firewall-cmd --reload
systemctl start grafana-server
```
This will start the grafana-server process as the grafana user, which is created during package installation. 
The default HTTP port is 3000, and default user and group is admin.
For later use in the dashboard install additional plugin:
```
grafana-cli plugins install grafana-worldmap-panel
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install savantly-heatmap-panel
```
### 4. Download and Install Maxmind GeoIp Databases

```
yum install geoipupdate
geoipupdate -d /usr/share/GeoIP/
```
Add cron (automatically updates Maxmind everyweek on Sunday at 1700hrs)
```
nano /etc/cron.weekly/geoipupdate
```
Add the following and save/exit
```
00 17 * * 0 geoipupdate -d /usr/share/GeoIP
```
Modify Graylog GeoIP configuration via WebGui:

![GeoIpConf](https://github.com/jbrundiers/Pfsense-Graylog-Grafana/blob/master/Pictures/GraylogGeoIPConf.JPG)


### 5. Download the Custom Content Pack
This content pack includes Input PFsenseGG-Input with 3 extractors and their lookup tables, Grok patterns PF_*, the Pipeline PFsenseGG-Pipeline and the Stream PFsenseGG-Stream.

We can take it from the Git directory or sideload it from github to the Workstation you do the deployment on:
https://raw.githubusercontent.com/jbrundiers/Pfsense-Graylog-Grafana/master/PFsenseGG-content-pack.json

Modify Graylog Stream Index for PFsenseGG-Stream: 

![StreamConf](https://github.com/jbrundiers/Pfsense-Graylog-Grafana/blob/master/Pictures/GraylogStreamSettings.JPG)


### 6. Use CEREBRO to manipulate the Elastic-Index
By default graylog for each index that is created generates its own template and applies it every time the index rotates. If we want our own templates we must create them in the same elasticsearch. 
We have to install our own custom index tempate because we need index fields of the type geo_point. We will convert the geo type dest_ip_geolocation and src_ip_geolocation to type geo_point to be used in the World Map panels since graylog does not use this format.
To import the template open cerebro and go to more/index template and create a new template. In the name fill in "pfsensegg-custom".

Get the Index Template from the GIT repo you cloned or sideload it from:

https://raw.githubusercontent.com/jbrundiers/Pfsense-Graylog-Grafana/master/PFsenseGG-custom.elasticsearch

Open the git file that has the template and paste its content into cerebro and click "create".

![IndexConf](https://github.com/jbrundiers/Pfsense-Graylog-Grafana/blob/master/Pictures/CerebroIndexTemplate.JPG)


!!! IMPORTANT: Now we will stop the graylog service to proceed to eliminate the index through Cerebro.
```
 systemctl stop graylog-server.service
```

In Cerebro we stand on top of the index and unfold the options and select delete index.
![IndexDel](https://github.com/jbrundiers/Pfsense-Graylog-Grafana/blob/master/Pictures/CerebroIndexDelete.JPG)

We start the graylog service again and this will recreate the index with this template.

```
 systemctl start graylog-server.service
```
Once this procedure is done, we don't need Cerebro for daily work, so it could be disable in docker-compose.yml.

### 7. Login to pfSense and Forward syslogs
- In pfSense navigate to Status->System Logs, then click on Settings.
- At the bottom check "Enable Remote Logging"
- (Optional) Select a specific interface to use for forwarding
- Enter the ELK local IP into the field "Remote log servers" with port 5140
- Under "Remote Syslog Contents" check "Everything"
- Click Save



