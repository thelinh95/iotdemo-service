#### An Ambari Stack for HDP IoT Demo
Ambari stack for easily installing and managing HDP IoT Demo which shows real time monitoring/alerts and predictions of driving violations generated by fleet of trucks. 
Also has optional steps to setup Ranger audits to Solr and a SILK dashboard to visualize audits 

Videos on the demo itself available here:
  - [Arun's Hadoop Summit 2015 keynote](https://youtu.be/FHMMcMYhmNI?t=1h25m13s)
  - [Shaun/George's Hadoop Summit 2014 Keynote](http://library.fora.tv/program_landing_frameview?id=20333&type=clip)
  - [Nauman's Demo at Phoenix Data conference 2014](http://www.youtube.com/watch?v=ErDmSIQ4gX0)
  - [![Phoenix Data conference 2014](http://img.youtube.com/vi/ErDmSIQ4gX0/0.jpg)](http://www.youtube.com/watch?v=ErDmSIQ4gX0)

Pre-reqs: 
  - You must have access to [sedev git repo](https://github.com/hortonworks/sedev) to install this service. 
  - The service currently requires that it is installed on the Ambari server node and that Kafka and Zookeeper and also running on the same node.
  - HBase and Storm must be available on the cluster and started. 
  - Falcon must be stopped before installing this service.

Previous versions:
  - For 2.2 version of the steps see [here](https://github.com/abajwa-hw/iotdemo-service/blob/master/README-22.md)
  
##### Setup steps

- Download HDP 2.3 sandbox VM image (Sandbox_HDP_2.3_VMWare.ova) from [Hortonworks website](http://hortonworks.com/products/hortonworks-sandbox/)
- Import Sandbox_HDP_2.3_VMWare.ova into VMWare and set the VM memory size to 8GB or more
- Now start the VM
- After it boots up, find the IP address of the VM and add an entry into your machines hosts file e.g.
```
192.168.191.241 sandbox.hortonworks.com sandbox    
```
- Connect to the VM via SSH (password hadoop) and restart Ambari server
```
ssh root@sandbox.hortonworks.com
/root/start_ambari.sh
```

- **Make sure Storm, HBase, Kafka, Hive are up and Falcon is down** and all these services are out of maintenance mode. You can run below from Ambari server as a shortcut

```
#Ambari password
export PASSWORD=admin
#Ambari host
export AMBARI_HOST=localhost

#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

#make sure kafka, storm, falcon are out of maintenance mode
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Falcon from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/FALCON
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Kafka from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/KAFKA
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Storm from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/STORM
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Hbase from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HBASE
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Remove Hive from maintenance mode"}, "Body": {"ServiceInfo": {"maintenance_state": "OFF"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HIVE


#Start Kafka, Storm, HBase, Hive
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start KAFKA via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/KAFKA
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start STORM via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/STORM
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start HBASE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HBASE
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start HIVE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/HIVE


#stop Falcon
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop FALCON via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/FALCON

```

- Ensure that root and storm users can create/access HBASE tables. If Ranger is installed (e.g if you running on sandbox) you can login to Ranger UI to check this:
  - Login to Ranger ui at http://sandbox.hortonworks.com:6080 (admin/admin)
  - Open the HBase policy page (http://sandbox.hortonworks.com:6080/index.html#!/hbase/3/policy/8). and ensure that groups "root" and "hadoop" have access. If not, add them. Click "Save" to refresh the policy.
  - Ensure root has authority to create tables. You can do this by SSH as root and trying to create a test table:
```
hbase shell
create 't1', 'f1', 'f2', 'f3'
```


- (Optional): Setup Solr and Banana and 'Ranger Audits' dashboard.

  - Option 1: Install new instance of Solr under /opt/solr
```
cd
wget https://github.com/abajwa-hw/security-workshops/raw/master/scripts/setup_solr_banana.sh
chmod +x setup_solr_banana.sh

./setup_solr_banana.sh <arguments>
```
    - argument options:
      - if no arguments passed, FQDN will be used as hostname to setup dashboard/view (use this if you have created local hosts entry for host where Solr will run e.g. sandbox.hortonworks.com)
      - if "publicip" is passed, the public ip address will be used as hostname to setup dashboard/view (use this on cloud environments)
      - otherwise the passed in value will be assumed to be the hostname to setup dashboard/view

    - Solr UI should be available at http://(your hostname):6083/solr/#/ranger_audits e.g. http://sandbox.hortonworks.com:6083/solr/#/ranger_audits 
    - An Empty Banana dashboard should be available at http://(your hostname):6083/banana e.g. http://sandbox.hortonworks.com:6083/banana. 

  - Option 2: Setup using HDP Search which comes pre-installed on HDP 2.3 sandbox 
```
#setup on hdp_search

cd /usr/local/
sudo wget https://github.com/abajwa-hw/security-workshops/raw/master/scripts/ranger_solr_setup.zip
sudo unzip ranger_solr_setup.zip
sudo rm -rf __MACOSX
cd ranger_solr_setup
vi install.properties   

#change below in install.properties to use HDP search setup instead of installing Solr
SOLR_INSTALL=false
SOLR_INSTALL_FOLDER=/opt/lucidworks-hdpsearch/solr
SOLR_RANGER_HOME=/opt/lucidworks-hdpsearch/solr/ranger_audit_server
SOLR_RANGER_DATA_FOLDER=/opt/lucidworks-hdpsearch/solr/ranger_audit_server/data

./setup.sh  
mkdir /opt/banana-ranger
cd /opt/banana-ranger
sudo git clone https://github.com/LucidWorks/banana.git
sudo mv banana latest
cd latest/
sudo sed -i 's/logstash_logs/ranger_audits/g' src/config.js
sudo wget https://raw.githubusercontent.com/abajwa-hw/security-workshops/master/scripts/default.json -O src/app/dashboards/default.json
mkdir build
yum install -y ant
sudo ant
cp /opt/banana-ranger/latest/build/banana-0.war /opt/lucidworks-hdpsearch/solr/server/webapps/banana.war
cp /opt/banana-ranger/latest/jetty-contexts/banana-context.xml /opt/lucidworks-hdpsearch/solr/server/contexts/
/opt/lucidworks-hdpsearch/solr/ranger_audit_server/scripts/start_solr.sh
```
  
- (Optional): setup HBase Ranger plugin to audit to Solr

```
cd /usr/hdp/2.*/ranger-hbase-plugin/       
vi /usr/hdp/2.*/ranger-hbase-plugin/install.properties
XAAUDIT.SOLR.IS_ENABLED=true
XAAUDIT.SOLR.SOLR_URL=http://sandbox.hortonworks.com:6083/solr/ranger_audits

./enable-hbase-plugin.sh
```

- (Optional): setup Hive Ranger plugin to audit to Solr
```
cd /usr/hdp/2.*/ranger-hive-plugin/       
vi /usr/hdp/2.*/ranger-hive-plugin/install.properties
XAAUDIT.SOLR.IS_ENABLED=true
XAAUDIT.SOLR.SOLR_URL=http://sandbox.hortonworks.com:6083/solr/ranger_audits

./enable-hive-plugin.sh
```

- Now retart HBase and Hive to register the plugins.

- To deploy the IOTDEMO stack, run below
```
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'`
sudo git clone https://github.com/abajwa-hw/iotdemo-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/IOTDEMO   

#on sandbox
sudo service ambari restart

#on non sandbox
sudo service ambari-server restart
```
- Then you can click on 'Add Service' from the 'Actions' dropdown menu in the bottom left of the Ambari dashboard:

On bottom left -> Actions -> Add service -> check IOTDEMO server -> Next -> Next -> Configure service -> Next -> Deploy
![Image](../master/screenshots/select-service.png?raw=true)

Things to remember while configuring the service
  - The service currently requires that it is installed on the Ambari server node and that Kafka and Zookeeper and also running on the same node.
  - Under "Advanced demo-config", you need to enter your github credentials to allow the service to access the IoT demo artifacts
  - Under "Advanced user-config", you need to update your ambari user/password/port configuration (if not using default)
  - The IoT demo configs are available under "Advanced demo-env", but do not require updating as all required configs will be auto-populated:
    - Ambari host
    - Name node host/port
    - Nimbus host
    - Hive metastore host/port
    - Supervisor host
    - HBase master host
    - Kafka host/port (also where ActiveMQ will be installed)
  
![Image](../master/screenshots/config1.png?raw=true)

![Image](../master/screenshots/config2.png?raw=true)
      

- On successful deployment you will see the IOTDEMO service as part of Ambari stack and will be able to start/stop the service from here:
![Image](../master/screenshots/started-service.png?raw=true)


- You can see the parameters you configured under 'Configs' tab
![Image](../master/screenshots/started-config.png?raw=true)

- One benefit to wrapping the component in Ambari service is that you can now monitor/manage this service remotely via REST API
```
export SERVICE=IOTDEMO
export PASSWORD=admin
export AMBARI_HOST=localhost

#detect name of cluster
output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`

#get service status
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X GET http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#start service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Start $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "STARTED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE

#stop service
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X PUT -d '{"RequestInfo": {"context" :"Stop $SERVICE via REST"}, "Body": {"ServiceInfo": {"state": "INSTALLED"}}}' http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/$SERVICE
```


- To remove the IOTDEMO service: 
  - Stop the service via Ambari
  - Delete the service
  
    ```
export PASSWORD=admin
export AMBARI_HOST=localhost

output=`curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari'  http://$AMBARI_HOST:8080/api/v1/clusters`
CLUSTER=`echo $output | sed -n 's/.*"cluster_name" : "\([^\"]*\)".*/\1/p'`
    
curl -u admin:$PASSWORD -i -H 'X-Requested-By: ambari' -X DELETE http://$AMBARI_HOST:8080/api/v1/clusters/$CLUSTER/services/IOTDEMO
    ```
  - Remove artifacts 
  
    ```
rm -rf /var/lib/ambari-server/resources/stacks/HDP/2.2/services/iotdemo*
rm -rf /root/sedev
    ```


#### Access webapp

- Open the webapp at http://sandbox.hortonworks.com:8081/storm-demo-web-app/
  - Login
  - Generate <50 events
  - Navigate to the monitoring and prediction webapps

- Alternatively, you can launch it from Ambari via [iFrame view](https://github.com/abajwa-hw/iframe-view)
![Image](../master/screenshots/iot-predictionapp.png?raw=true)

- Check Storm view for metrics
![Image](../master/screenshots/iot-stormview.png?raw=true)

- Alternatively check Storm UI for metrics
![Image](../master/screenshots/storm.png?raw=true)


#### Access Ranger audits dashboard

- Open the Ranger Audits dashboard at http://sandbox.hortonworks.com:6083/banana

- By default you will see a visualization of HBase reads/gets:
![Image](../master/screenshots/iot-rangeraudit-hbase-get-1.png?raw=true)
![Image](../master/screenshots/iot-rangeraudit-hbase-get-2.png?raw=true)

- Change the query filter to search for writes/puts:
![Image](../master/screenshots/iot-rangeraudit-hbase-put-1.png?raw=true)

- Now open Hive view and query the truck_events_text_partition table:
![Image](../master/screenshots/iot-hive-query.png?raw=true)

- On the Ranger audits dashboard, query for Hive audits:
![Image](../master/screenshots/iot-rangeraudit-hive.png?raw=true)

- Now disable the global allow policy on Hbase and Hive and wait 30s:
![Image](../master/screenshots/iot-disable-hbasepolicy.png?raw=true)
![Image](../master/screenshots/iot-disable-hivepolicy.png?raw=true)

- Try running the same query in Hive view

- At this point, you should should see some Hbase audit records with result=0
![Image](../master/screenshots/iot-rangeraudit-hbase-rejection.png?raw=true)
![Image](../master/screenshots/iot-rangeraudit-hive-rejection.png?raw=true)

- Confirm the same by opening the Audit tab of Ranger: http://sandbox.hortonworks.com:6080

![Image](../master/screenshots/iot-ranger-hbase-rejection.png?raw=true)
![Image](../master/screenshots/iot-ranger-hive-rejection.png?raw=true)

- Re-enable the global allow policies.


