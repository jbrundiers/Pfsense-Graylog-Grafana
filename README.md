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
Modify Grylog GeoIP configuration via WebGui:

![GeoIpConf](https://github.com/jbrundiers/Pfsense-Graylog-Grafana/blob/master/GraylogGeoIPConf.JPG)



### 14. Download the following configuration files
```
sudo wget https://raw.githubusercontent.com/a3ilson/pfelk/master/conf.d/01-inputs.conf
```
```
sudo wget https://raw.githubusercontent.com/a3ilson/pfelk/master/conf.d/11-firewall.conf
```
```
sudo wget https://raw.githubusercontent.com/a3ilson/pfelk/master/conf.d/20-geoip.conf
```


### 17. Download the following configuration file
```
sudo wget https://raw.githubusercontent.com/a3ilson/pfelk/master/conf.d/patterns/pf-12.2019.grok
```


### 22. Login to pfSense and Forward syslogs
- In pfSense navigate to Status->System Logs, then click on Settings.
- At the bottom check "Enable Remote Logging"
- (Optional) Select a specific interface to use for forwarding
- Enter the ELK local IP into the field "Remote log servers" with port 5140
- Under "Remote Syslog Contents" check "Everything"
- Click Save



