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
### 4. Install GeoIp Database
```
wget -t0 -c http://geolite.maxmind.com/download/geoip/database/GeoLite2-City.tar.gz
tar -xvf GeoLite2-City.tar.gz
cp GeoLite2-City_*/GeoLite2-City.mmdb /etc/graylog/server
```


### 5. Summary Installation

Now your server should expose you the following services externally:

Graylog	      on port 9000    http://hostname:9000

Grafana	      on port 3000    http://hostname:3000

and the following services localy:

Elasticsearch on port 9200	  http://localhost:9200

Cerebro	      on port 9001    http://localhost:9001




```

### 8. Download Maxmind Databases
```
sudo geoipupdate -d /usr/share/GeoIP/
```

### 9. Add cron (automatically updates Maxmind everyweek on Sunday at 1700hrs)
```
sudo nano /etc/cron.weekly/geoipupdate
```
- Add the following and save/exit
```
00 17 * * 0 geoipupdate -d /usr/share/GeoIP
```





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

### 18. Enter your pfSense/OPNsense IP address (01-inputs.conf)
```
Change line 9; the "if [host] =~ ..." should point to your pfSense/OPNsense IP address
Change line 12-16; (OPTIONAL) to point to your second PF IP address or ignore
```

### 19. Revise/Update w/pf IP address (01-inputs.conf)
```
For pfSense uncommit line 28 and commit out line 25
For OPNsense uncommit line 25 and commit out line 28
```

# Configure Services

### 20. Start Services on Boot as Services (you'll need to reboot or start manually to proceed)
```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl enable elasticsearch.service
sudo /bin/systemctl enable kibana.service
sudo /bin/systemctl enable logstash.service
```

### 21. Start Services Manually
```
systemctl start elasticsearch 
systemctl start kibana 
systemctl start logstash 
```

### 22. Login to pfSense and Forward syslogs
- In pfSense navigate to Status->System Logs, then click on Settings.
- At the bottom check "Enable Remote Logging"
- (Optional) Select a specific interface to use for forwarding
- Enter the ELK local IP into the field "Remote log servers" with port 5140
- Under "Remote Syslog Contents" check "Everything"
- Click Save



### 23. Set-up Kibana
- In your web browser go to the ELK local IP using port 5601 (ex: 192.168.0.1:5601)
- Click the wrench (Dev Tools) icon in the left pannel 
- Input the following and press the click to send request button (triangle)
- https://raw.githubusercontent.com/a3ilson/pfelk/master/Dashboard/GeoIP(Template)
- Click the gear icon (management) in the lower left
- Click Kibana -> Index Patters
- Click Create New Index Pattern
- Type "pf*" into the input box, then click Next Step
- In the Time Filter drop down select "@timestamp"
- Click Create then verify you have data showing up under the Discover tab

### 24. Import dashboards
 - In your web browser go to the ELK local IP using port 5601 (ex: 192.168.0.1:5601)
 - Click Management -> Saved Objects
 - You can import the dashboards found in the `Dashboard` folder via the Import buttom in the top-right corner.

### Troubleshooting
- Restart services:
```
systemctl stop elasticsearch 
systemctl stop kibana 
systemctl stop logstash 
systemctl start elasticsearch 
systemctl start kibana 
systemctl start logstash 
```

- Check logs for errors:
```
sudo vi /var/log/logstash/logstash-plain.log
sudo vi /var/log/elasticsearch/elasticsearch.log
(Press Shift + G to scroll to bottom, Escape then type ":q!" to exit)
```

- Check the Pipeline Viewer UI to visualize your pipeline

This feature is part of the (free) X-Pack addons.
You only need to enable monitoring for Logstash. 
To do this have the following lines in `/etc/logstash/logstash.yml`:
```
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.hosts: [ "http://your-elasticsearch-instance:9200" ]
```
Restart Logstash
```
systemctl restart logstash
```
Go to Stack Monitoring / Logstash / Pipelines / Choose the ID of your relevant pipelines.

If this helped, feel free to make a contribution:

