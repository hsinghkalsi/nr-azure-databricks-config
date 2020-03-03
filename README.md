# New Relic Infrastructure Agent with Azure Databricks
This readme provide instructions to deploy along with apache spark on host integration on Azure databricks platform.

> ***This is a beta version for use in POC environment only*** 

## Instructions

1. Create a new notebook to deploy the cluster intialization script
2. Configure **<NR_WORKING_FOLDER>** with the location to put the init script.
3. Replace **<NR_LICENSE_KEY>** with your New Relic license.
4. Run this notebook to create the script newrelic_install.sh in /tmp folder.
5. Configure a cluster with the ***newrelic_install.sh*** cluster-scoped init script using the UI, Databricks CLI, or by invoking the Clusters API.

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
