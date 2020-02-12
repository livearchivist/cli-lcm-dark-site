# CLI LCM Dark Site Mode

CLI only way to enable Dark Site mode for LCM and update MSP and Objects Controller.

Let me know if you have any questions!

## Enable Dark Site Mode and Update MSP and Objects

1. Download the LCM Dark Site Bundle, `nutanix_compatibility.tgz`, `nutanix_compatibility.tgz.sign`, and `objects-1.1.tar.gz` from the Portal. Today, these are downloadable without auth:

```
cd /tmp
wget http://download.nutanix.com/lcm/2.3.0.2/lcm_dark_site_bundle_2.3.0.2.13853.tar.gz
wget http://download.nutanix.com/NutanixObjects/1.1/objects-1.1.tar.gz
wget http://download.nutanix.com/lcm/2.0/nutanix_compatibility.tgz
wget http://download.nutanix.com/lcm/2.0/nutanix_compatibility.tgz.sign
```

2. Set up a web server, in this case I'll use Nginx.

```
yum install nginx -y
cd /usr/share/nginx/html
rm -rf ./*
```

_Note:_ You may have to make some configuration changes to the Web Server's firewall

3. Copy and unpack all the bits in the right places:

```
mkdir release
cd release
cp /tmp/lcm_dark_site_bundle_2.3.0.2.13853.tar.gz .
tar -xvf lcm_dark_site_bundle_2.3.0.2.13853.tar.gz
yes | cp -r /tmp/nutanix_compatibility* . # This will overwrite the nutanix_compatibility files from the lcm_dark_site_bundle
cp /tmp/objects-1.1.tar.gz .
tar -xvf objects-1.1.tar.gz
```

4. Validate web directory structure looks like this:

`tree -L 2 /usr/share/nginx/html`

```
/usr/share/nginx/html
└── release
    ├── builds
    ├── lcm_dark_site_bundle_2.3.0.2.13853.tar.gz
    ├── master_manifest.tgz
    ├── master_manifest.tgz.sign
    ├── modules
    ├── nutanix_compatibility.tgz
    ├── nutanix_compatibility.tgz.sign
    ├── objects-1.1.tar.gz
    ├── support.csv
    └── support.md
```

5. Enable Objects

```
# Check if enabled
curl -k https://$PCIP:9440/api/nutanix/v3/services/oss/status -u admin:'$PCPASS'

# If enabled, output will look like this:
{"service_enablement_status": "ENABLED"}

# If disabled, output will look like this:
{"service_enablement_status": "DISABLED"}

# Enable Objects
curl -k -X POST -d '{"state": "ENABLE"}'  -H 'Content-Type: application/json' https://$PCIP:9440/api/nutanix/v3/services/oss  -u admin:'$PCPASS'

# It will respond with a task UUID:
{"task_uuid": "52778567-9d99-4b4d-a141-83403de5129e"}

# While the service is being enabled, if you run the "Check if enabled" query again, it'll respond with:
{"service_enablement_status": "ENABLING"}
```

6. Validate MSP Version

Ensure MSP is on version 1.0.3 or higher

```
admin@pcvm$ mspctl controller_version
```

In my case, I was on version 1.0.2

```
MSP controller version    : 1.0.2
MSP controller build      : 79b5866
MSP controller build date : 2019-10-19
```

7. Enable LCM Dark Site (LCM 2.1.x)

_NOTE: `$NGINXIP` is the IP address of the Nginx web server set up in step #2._

```
curl -k -X POST -d '{"value":"{\".oid\":\"LifeCycleManager\",\".method\":\"lcm_framework_rpc\",\".kwargs\":{\"method_class\":\"LcmFramework\",\"method\":\"configure\",\"args\":[\"http://$NGINXIP/release\",false]}}"}' -H 'Content-Type: application/json' https://$PCIP:9440/PrismGateway/services/rest/v1/genesis -u admin:'$PCPASS'
```

Perform LCM inventory against dark site

```
curl -k -X POST -d '{"value":"{\".oid\":\"LifeCycleManager\",\".method\":\"lcm_framework_rpc\",\".kwargs\":{\"method_class\":\"LcmFramework\",\"method\":\"perform_inventory\",\"args\":[\"http://$NGINXIP/release\"]}}"}' -H 'Content-Type: application/json' https://$PCIP:9440/PrismGateway/services/rest/v1/genesis -u admin:'$PCPASS'
```

8. Update MSP and Objects Controller

We need to give LCM the plan to update MSP to 1.0.4 and Objects Controller to 1.1

**Get UUIDs for Objects Controller and MSP**

```
ossUUID=`curl --max-time 25 --silent --header Content-Type:application/json --header Accept:application/json --insecure --user admin:'$PCPASS' -X POST -d '{"entity_type": "lcm_entity_v2","group_member_count": 500,"group_member_attributes": [{"attribute": "id"}, {"attribute": "uuid"}, {"attribute": "entity_model"}, {"attribute": "version"}, {"attribute": "location_id"}, {"attribute": "entity_class"}, {"attribute": "description"}, {"attribute": "last_updated_time_usecs"}, {"attribute": "request_version"}, {"attribute": "_master_cluster_uuid_"}, {"attribute": "entity_type"}, {"attribute": "single_group_uuid"}],"query_name": "lcm:EntityGroupModel","grouping_attribute": "location_id","filter_criteria": "entity_model!=AOS;entity_model!=NCC;entity_model!=PC;_master_cluster_uuid_==[no_val]"}' https://$PCIP:9440/api/nutanix/v3/groups | jq '.group_results[0].entity_results[0].entity_id'`

mspUUID=`curl --max-time 25 --silent --header Content-Type:application/json --header Accept:application/json --insecure --user admin:'$PCPASS' -X POST -d '{"entity_type": "lcm_entity_v2","group_member_count": 500,"group_member_attributes": [{"attribute": "id"}, {"attribute": "uuid"}, {"attribute": "entity_model"}, {"attribute": "version"}, {"attribute": "location_id"}, {"attribute": "entity_class"}, {"attribute": "description"}, {"attribute": "last_updated_time_usecs"}, {"attribute": "request_version"}, {"attribute": "_master_cluster_uuid_"}, {"attribute": "entity_type"}, {"attribute": "single_group_uuid"}],"query_name": "lcm:EntityGroupModel","grouping_attribute": "location_id","filter_criteria": "entity_model!=AOS;entity_model!=NCC;entity_model!=PC;_master_cluster_uuid_==[no_val]"}' https://$PCIP:9440/api/nutanix/v3/groups | jq '.group_results[0].entity_results[1].entity_id'`
```

_NOTE: You can check to ensure you're getting the right UUID by parsing with the following JQ filters:_

```
.group_results[0].entity_results[0].data[2].values[0].values[0]
prints: Objects Manager

.group_results[0].entity_results[1].data[2].values[0].values[0]
prints: MSP
```

**Generate Plan for MSP and Objects Controller**

_NOTE: Make sure $mspUUID and $ossUUID are passed into (or replaced in) the command below!_

```
curl -k -X POST -d '{"value":"{\".oid\":\"LifeCycleManager\",\".method\":\"lcm_framework_rpc\",\".kwargs\":{\"method_class\":\"LcmFramework\",\"method\":\"generate_plan\",\"args\":[\"http://$NGINXIP/release\",[[\"$mspUUID\",\"1.0.4\"],[\"$ossUUID\",\"1.1\"]]]}}"}' -H 'Content-Type: application/json' https://$PCIP:9440/PrismGateway/services/rest/v1/genesis -u admin:'$PCPASS'
```

The cluster will output something to the effect of:

```
{"value":"{\".return\": {\"node:d834cd05-f2b8-4018-80a6-81ba32166c84\": [\"Genesis service will be restarted on the cluster. User workloads will not be disrupted. Refer to KB 6945 for more details.\", \"\"]}}"}
```

**Apply Plan for MSP and Objects Controller**

_NOTE: Make sure $mspUUID and $ossUUID are passed into (or replaced in) the command below!_

```
curl -k -X POST -d '{"value":"{\".oid\":\"LifeCycleManager\",\".method\":\"lcm_framework_rpc\",\".kwargs\":{\"method_class\":\"LcmFramework\",\"method\":\"perform_update\",\"args\":[\"http://$NGINXIP/release\",[[\"$mspUUID\",\"1.0.4\"],[\"$ossUUID\",\"1.1\"]]]}}"}' -H 'Content-Type: application/json' https://$PCIP:9440/PrismGateway/services/rest/v1/genesis -u admin:'$PCPASS'
```

9. Enable MSP Airgap

```
admin@pcvm$ mspctl airgap --enable --lcm-server=http://$NGINXIP/release
```

Check MSP Airgap Status

```
admin@pcvm$ mspctl airgap -s

{"enable":true,"lcm_server":"http://$NGINXIP/release"}
```

10. Follow directions at below repo to install Objects

[objects_deploy.py](https://github.com/lauramariel/scripts/blob/master/ONCE/objects_deploy.py)
