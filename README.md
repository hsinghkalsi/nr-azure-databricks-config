# New Relic Infrastructure Agent with Azure Databricks
This readme provide instructions to deploy along with apache spark on host integration on Azure databricks platform.

> ***This is a beta version for use in POC environment only*** 

## Instructions

1. Create a new notebook to deploy the cluster intialization script. 
2. Configure **<NR_WORKING_FOLDER>** with the location to put the init script.
3. Replace **<NR_LICENSE_KEY>** with your New Relic license.
4. Run this notebook to create to deploy the new_relic_install.sh script in dbfs in configured folder.
5. Configure target cluster with the ***newrelic_install.sh*** cluster-scoped init script using the UI, Databricks CLI, or by invoking the Clusters API.

``` shell
dbutils.fs.put("dbfs:/<NR_WORKING_FOLDER>/newrelic_install.sh",""" 
#!/bin/sh

echo "Check if this is driver? $DB_IS_DRIVER"
echo "Spark Driver ip: $DB_DRIVER_IP"

# Create Cluster init script
cat <<EOF >> /tmp/start_nr.sh

#!/bin/sh

if [ \$DB_IS_DRIVER ]; then
  # Root user detection
  if [ \$(echo "$UID") = "0" ];                                      
  then                                                                     
    sudo=''                                                                
  else
    sudo='sudo'                                                        
  fi
  echo "license_key: <NR_LICENSE_KEY>" | \$sudo tee -a /etc/newrelic-infra.yml
  
  #Download upstart based installer to /tmp
  \$sudo wget https://download.newrelic.com/infrastructure_agent/linux/apt/pool/main/n/newrelic-infra/newrelic-infra_upstart_1.3.27_upstart_amd64.deb -P /tmp

  # Install NR Agent
  \$sudo dpkg -i /tmp/newrelic-infra_upstart_1.3.27_upstart_amd64.deb

  #Download nr-nri-spark integration
  \$sudo wget https://github.com/hsinghkalsi/nr-azure-databricks-config/releases/download/1.0.1/nr-nri-spark.tar.gz  -P /tmp

  #extract the contents to right place
  \$sudo tar -xvzf /tmp/nr-nri-spark.tar.gz -C /

  # GRAB PORT for masterUI
   while [ -z \$isavailable ]; do
    if [ -e "/tmp/master-params" ]; then
      DB_DRIVER_PORT=\$(cat /tmp/master-params | cut -d' ' -f2)
      isavailable=TRUE
    fi
    sleep 2
  done
  
   # configure spark instances
  echo "sparkmasterurl: http://\$DB_DRIVER_IP:\$DB_DRIVER_PORT
clustername: \$DB_CLUSTER_ID" > /var/db/newrelic-infra/custom-integrations/nr-nri-spark-settings.yml

# stop and restart the agent 
restart newrelic-infra

fi
EOF

# Start 
if [ \$DB_IS_DRIVER ]; then
  chmod a+x /tmp/start_nr.sh
  /tmp/start_nr.sh >> /tmp/start_nr.log 2>&1 & disown
fi

""",True)

```


# New Relic Infrastructure Integration for Apache Spark (METRICS INTEGRATION)

This New Relic  standalone integration polls the Apache Spark [REST API](https://spark.apache.org/docs/latest/monitoring.html#rest-api) for metrics and pushes them into New Relic  using Metrics API

It uses the New Relic [Telemetry sdk for go](https://github.com/newrelic/newrelic-telemetry-sdk-go)

## Requirements

* Apache Spark runnning in standalone mode (YARN and mesos not yet supported)

## Installation & Configuration

1. Download the latest package from Release.
2. Install the NR Spark Metric plugin plugin using the following command 
    > sudo tar -xvzf /tmp/nri-spark-metric.tar.gz -C /

    The following files will be installed 
    ```
    /etc/nri-spark-metric/
    /etc/nri-spark-metric/nr-spark-metric
    /etc/systemd/system/
    /etc/systemd/system/nr-spark-metric.service
    /etc/init/nr-spark-metric.conf
    ```
3. Create "*nr-spark-metric-settings.yml*" file in the the folder "*/etc/nri-spark-metric/*"  using the following format
    ```
    sparkmasterurl: "http://localhost:8080"  <== FQDN ofspark master URL
    clustername:  mylocalcluster             <== Name of the cluster
    insightsapikey: xxxx                     <== Insights api key
    pollinterval: 5                          <== Polling interval
    ```
4. Run the following command.
    > service nr-spark-metric start

5. Check for metrics in "Metric" event type in Insights

## Databricks Init script creator notebook for metrics endpoint

```
dbutils.fs.put("dbfs:/nr/nri-spark-metric.sh",""" 
#!/bin/sh
echo "Check if this is driver? $DB_IS_DRIVER"
echo "Spark Driver ip: $DB_DRIVER_IP"

# Create Cluster init script
cat <<EOF >> /tmp/start_spark-metric.sh

#!/bin/sh

if [ \$DB_IS_DRIVER ]; then
  # Root user detection
  if [ \$(echo "$UID") = "0" ];                                      
  then                                                                     
    sudo=''                                                                
  else
    sudo='sudo'                                                        
  fi

  #Download nr-spark-metric integration
  \$sudo wget https://github.com/hsinghkalsi/nr-azure-databricks-config/releases/download/1.0.3/nri-spark-metric.tar.gz  -P /tmp

  #extract the contents to right place
  \$sudo tar -xvzf /tmp/nri-spark-metric.tar.gz -C /

  # GRAB PORT for masterUI
   while [ -z \$isavailable ]; do
    if [ -e "/tmp/master-params" ]; then
      DB_DRIVER_PORT=\$(cat /tmp/master-params | cut -d' ' -f2)
      isavailable=TRUE
    fi
    sleep 2
  done
  
   # configure spark instances
  echo "sparkmasterurl: http://\$DB_DRIVER_IP:\$DB_DRIVER_PORT
clustername: \$DB_CLUSTER_ID
insightsapikey: <<Add your insights key>>
pollinterval: 5 " > /etc/nri-spark-metric/nr-spark-metric-settings.yml

   #enable 
 \$sudo systemctl enable nr-spark-metric.service
 
 #start the service 
 \$sudo systemctl start nr-spark-metric.service
 \$sudo start nr-spark-metric

fi
EOF

# Start 
if [ \$DB_IS_DRIVER ]; then
  chmod a+x /tmp/start_spark-metric.sh
  /tmp/start_spark-metric.sh >> /tmp/start_spark-metric.log 2>&1 & disown
fi

""",True)
```




