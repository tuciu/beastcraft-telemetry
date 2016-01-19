# beastcraft-telemetry
`DS18B20` one-wire temperature monitoring with Grafana and Python on a Raspberry Pi 2 inside a motorhome.

### Installation
I've built all the pre-requisites (except [Go](http://www.aymerick.com/2013/09/24/go_language_on_raspberrypi.html)) thanks to [InfluxDB, Telegraf and Grafana on a Raspberry Pi 2](http://www.aymerick.com/2015/10/07/influxdb-telegraf-grafana-raspberry-pi.html) guide. Still took a good part of two days.

    # install InfluxDB
    sudo dpkg -i ./influxdb-armhf/influxdb_0.9.6_armhf.deb
    sudo service influxdb start
    sudo update-rc.d influxdb defaults 95 10

Install pre-built Node.js for Raspberry Pi using the handy (Adafruit)[https://learn.adafruit.com/node-embedded-development/installing-node-dot-js] guide.

    # install Grafana
    sudo dpkg -i ./grafana-armhf/grafana_3.0.0-pre1_armhf.deb
    sudo service grafana-server start
    sudo update-rc.d grafana-server defaults 95 10
    
### Configuration
This repository contains configuration specific to my environment, with five `DS18B20` sensors in total. To personalise it:

1. run `pip install -r requirement.txt`
2. update `DS18B201` sensor list and database name in `w1_thermy.py`
3. [create database](https://influxdb.com/docs/v0.9/query_language/database_management.html#create-a-database-with-create-database) (e.g. grafana) for your metrics by running `influx -execute 'CREATE DATABASE grafana;`
4. [create retention policy](https://influxdb.com/docs/v0.9/query_language/database_management.html#retention-policy-management) by running `CREATE RETENTION POLICY 'grafana_retention_policy' ON 'grafana' DURATION '4w' REPLICATION 3 DEFAULT`

#### Temperature Dashboard

1. schedule `w1_therm.py` to run every x minutes using cron (e.g. `*/5 * * * * /opt/beastcraft-telemetry/w1_therm.py`)
2. go to [http://localhost:3000](http://localhost:3000), add a new datasource and configure other options
3. import `temp.json` dashboard and modify it to suit your needs or build your own from scratch

#### UPS/Inverter Dashboard

1. ensure [Network UPS Tools](http://www.networkupstools.org/) is correctly configured to work with the UPS (use `blazer_ser` driver)
2. edit `ups.py` and adjust the default `upsname` (mine is `upsoem`) or pass from the command line using `--ups` parameter
3. schedule `usp.py` to run every x minutes using cron (e.g. `*/5 * * * * /opt/beastcraft-telemetry/ups.py`)
4. import `ups.json` dashboard and modify it to suit your needs or build your own from scratch


#### Mobile Broadband Dashboard

1. update `host` variable in `mobilebroadband.py` to match your ZTE MF823 modem IP address
2. update `mobilebroadband.json` dashboard to match router and modem web front-end IPs

```
            {
              "type": "absolute",
              "url": "http://<your_ZTE-MF823_modem_IP>/",
              "targetBlank": true,
              "title": "Modem"
            },
            {
              "type": "absolute",
              "title": "Router",
              "url": "http|https://<your_A5-V11_router_IP>/",
              "targetBlank": true
            } 
```

3. schedule `mobilebroadband.py` to run every x minutes using cron (e.g. `*/1 * * * * /opt/beastcraft-telemetry/mobilebroadband.py`)
4. install `Nginx` and configure it as a front-end for both Grafana and ZTE web GUIs

````
pi@localhost ~ $ cat /etc/nginx/sites-enabled/grafana
server {

	listen 80;
	server_name <your_host_name>;

	location / {
		proxy_pass http://localhost:3000/;
	}

	location /public/ {
		alias /usr/share/grafana/public/;
	}

	location /goform/ {
		proxy_set_header Referer http://<your_ZTE-MF823_modem_IP>/;
		proxy_pass http://<your_ZTE-MF823_modem_IP>:80/goform/;
	}
}
````

The ZTE MF823 has a REST API apart from the web GUI, which we are using to communicate with the modem from within the Grafana dashboard. The full list of command sisn't published, but looking at the modem's web interface with Chrome Developer Tools, the following commands are evident

```
# connect mobile network (HTTP GET)
http://<modem_IP>/goform//goform_set_cmd_process?isTest=false&notCallback=true&goformId=CONNECT_NETWORK

# disconnect mobile network (HTTP GET)
http://<modem_IP>/goform//goform_set_cmd_process?isTest=false&notCallback=true&goformId=DISCONNECT_NETWORK

# modem state (HTTP GET)
http://<modem_IP>/goform/goform_get_cmd_process?multi_data=1&isTest=false&sms_received_flag_flag=0&sts_received_flag_flag=0&cmd=modem_main_state%2Cpin_status%2Cloginfo%2Cnew_version_state%2Ccurrent_upgrade_state%2Cis_mandatory%2Csms_received_flag%2Csts_received_flag%2Csignalbar%2Cnetwork_type%2Cnetwork_provider%2Cppp_status%2CEX_SSID1%2Cex_wifi_status%2CEX_wifi_profile%2Cm_ssid_enable%2Csms_unread_num%2CRadioOff%2Csimcard_roam%2Clan_ipaddr%2Cstation_mac%2Cbattery_charging%2Cbattery_vol_percent%2Cbattery_pers%2Cspn_display_flag%2Cplmn_display_flag%2Cspn_name_data%2Cspn_b1_flag%2Cspn_b2_flag%2Crealtime_tx_bytes%2Crealtime_rx_bytes%2Crealtime_time%2Crealtime_tx_thrpt%2Crealtime_rx_thrpt%2Cmonthly_rx_bytes%2Cmonthly_tx_bytes%2Cmonthly_time%2Cdate_month%2Cdata_volume_limit_switch%2Cdata_volume_limit_size%2Cdata_volume_alert_percent%2Cdata_volume_limit_unit%2Croam_setting_option%2Cupg_roam_switch%2Chplmn

# enable roaming
http://<modem_IP>/goform/goform_set_cmd_process?isTest=false&notCallback=true&goformId=SET_CONNECTION_MODE&roam_setting_option=on

# disable roaming
http://<modem_IP>/goform/goform_set_cmd_process?isTest=false&notCallback=true&goformId=SET_CONNECTION_MODE&roam_setting_option=off

# enable auto-dial
http://<modem_IP>/goform/goform/goform_set_cmd_process?isTest=false&notCallback=true&goformId=SET_CONNECTION_MODE&ConnectionMode=auto_dial

# disable auto-dial
http://<modem_IP>/goform/goform/goform_set_cmd_process?isTest=false&notCallback=true&goformId=SET_CONNECTION_MODE&ConnectionMode=manual_dial
```

All the API requests require the `Referer: http://<your_ZTE-MF823_modem_IP>/` request header present.

-- ab1
