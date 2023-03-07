---
layout: default
title: Rolling Upgrade - Lab
parent: Upgrades Appendix
grand_parent: Upgrading OpenSearch
nav_order: 50
redirect_from:
  - /upgrade-opensearch/appendix/rolling-upgrade-lab/
---

<!--
Testing out tabs for code blocks to identify example outputs and file names.
To use, invoke class="codeblock-label"
-->

<style>
.codeblock-label {
    display: inline-block;
    border-top-left-radius: 0.5rem;
    border-top-right-radius: 0.5rem;
    font-family: Menlo,Monaco,Consolas,Liberation Mono,Courier New,monospace;
    font-size: .75rem;
    --bg-opacity: 1;
    background-color: #e1e7ef;
    background-color: rgba(224.70600000000002,231.07080000000002,239.394,var(--bg-opacity));
    padding: 0.25rem 0.75rem;
    border-top-width: 1px;
    border-left-width: 1px;
    border-right-width: 1px;
    --border-opacity: 1;
    border-color: #ccd6e0;
    border-color: rgba(204,213.85999999999999,224.39999999999998,var(--border-opacity));
    margin-bottom: 0;
}
</style>

# Rolling Upgrade - Lab

This lab will walk You can follow these steps on your own compatible host to recreate the same cluster state the OpenSearch Project used for testing [rolling upgrades]({{site.url}}{{site.baseurl}}/install-and-configure/upgrade-opensearch/rolling-upgrade/).

The steps used in this lab were validated on an [Amazon Elastic Compute Cloud (Amazon EC2)](https://aws.amazon.com/ec2/) `t2.large` instance using [Amazon Linux 2](https://aws.amazon.com/amazon-linux-2/) kernel version `Linux 5.10.162-141.675.amzn2.x86_64` and [Docker](https://www.docker.com/) version `20.10.17, build 100c701`. The instance was provisioned with an attached 20 GiB gp2 [Amazon Elastic Block Store (EBS)](https://aws.amazon.com/ebs/) root volume.

References to the `$HOME` path on the host machine in this procedure are represented by the tilde character ("~") to make the instructions more portable. If you would prefer to specify an absolute path, modify the volume paths define in `upgrade-demo-cluster.sh` to reflect your environment.
{: .note}

The host type used for testing was chosen arbitrarily. Specifications are included for informational purposes only and do not represent a recommendation of system hardware requirements.
{: .note}

## Set up the environment

As you follow along with this document you will define several Docker resources including containers, volumes, and a dedicated Docker network using a script we provide. You can clean up your environment with the following command if you want to start the process over.

The following command removes container names matching the regular expression `os-*`, data volumes matching `data-0*` and `repo-0*`, and the Docker network named `opensearch-dev-net`. If you have other Docker resources running on your host, then you should take care to review and modify the command to avoid removing other resources unintentionally. This command does not revert changes to host memory swapping or the value of `vm.max_map_count`.

```bash
docker container stop $(docker container ls -aqf name=os-); \
	docker container rm $(docker container ls -aqf name=os-); \
	docker volume rm -f $(docker volume ls -q | egrep 'data-0|repo-0'); \
	docker network rm opensearch-dev-net
```
{% include copy.html %}

1. Install the appropriate version of [Docker Engine](https://docs.docker.com/engine/install/) for your Linux distribution and architecture. 
1. Configure [important system settings]({{site.url}}{{site.baseurl}}/install-and-configure/install-opensearch/index/#important-settings) on your host.
    1. Disable memory paging and swapping on the host to improve performance:
	   ```bash
	   sudo swapoff -a
	   ```
	   {% include copy.html %}
	1. Increase the number of memory maps available to OpenSearch. Open the `sysctl` configuration file for editing. This example command uses the [vim](https://www.vim.org/) text editor, but you can use any available text editor you prefer:
	   ```bash
	   sudo vim /etc/sysctl.conf
	   ```
	   {% include copy.html %}
	1. Add the following line to the file:
	   ```bash
	   vm.max_map_count=262144
	   ```
	   {% include copy.html %}
	1. Save and quit.
	1. Apply the configuration change:
	   ```bash
	   sudo sysctl -p
	   ```
	   {% include copy.html %}
1. Navigate to your home directory and create a directory named `deploy`. You will use the path `~/deploy` for the deployment script, configuration files, and TLS certificates.
   ```bash
   mkdir ~/deploy && cd ~/deploy
   ```
   {% include copy.html %}
1. Download `upgrade-demo-cluster.sh` from the OpenSearch [documentation-website](https://github.com/opensearch-project/documentation-website) repository:
   ```bash
   wget https://raw.githubusercontent.com/opensearch-project/documentation-website/main/assets/examples/upgrade-demo-cluster.sh
   ```
   {% include copy.html %}
1. Run the script without any changes to deploy four containers running OpenSearch and one container running OpenSearch Dashboards, with custom self-signed TLS certificates and a pre-defined set of internal users:
   ```bash
   sh upgrade-demo-cluster.sh
   ```
   {% include copy.html %}
1. Confirm that the containers were launched successfully with the following command:
   ```bash
   docker container ls
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```bash
   CONTAINER ID   IMAGE                                           COMMAND                  CREATED          STATUS          PORTS                                                                                                      NAMES
   6e5218c8397d   opensearchproject/opensearch-dashboards:1.3.7   "./opensearch-dashbo…"   24 seconds ago   Up 22 seconds   0.0.0.0:5601->5601/tcp, :::5601->5601/tcp                                                                  os-dashboards-01
   cb5188308b21   opensearchproject/opensearch:1.3.7              "./opensearch-docker…"   25 seconds ago   Up 24 seconds   9300/tcp, 9650/tcp, 0.0.0.0:9204->9200/tcp, :::9204->9200/tcp, 0.0.0.0:9604->9600/tcp, :::9604->9600/tcp   os-node-04
   71b682aa6671   opensearchproject/opensearch:1.3.7              "./opensearch-docker…"   26 seconds ago   Up 25 seconds   9300/tcp, 9650/tcp, 0.0.0.0:9203->9200/tcp, :::9203->9200/tcp, 0.0.0.0:9603->9600/tcp, :::9603->9600/tcp   os-node-03
   f894054a9378   opensearchproject/opensearch:1.3.7              "./opensearch-docker…"   27 seconds ago   Up 26 seconds   9300/tcp, 9650/tcp, 0.0.0.0:9202->9200/tcp, :::9202->9200/tcp, 0.0.0.0:9602->9600/tcp, :::9602->9600/tcp   os-node-02
   2e9c91c959cd   opensearchproject/opensearch:1.3.7              "./opensearch-docker…"   28 seconds ago   Up 27 seconds   9300/tcp, 9650/tcp, 0.0.0.0:9201->9200/tcp, :::9201->9200/tcp, 0.0.0.0:9601->9600/tcp, :::9601->9600/tcp   os-node-01
   ```
1. The amount of time it takes to initialize and bootstrap the cluster will vary depending on the performance capabilities of the underlying host. You can watch the container logs to see what OpenSearch is doing during cluster formation.
   1. Enter the following command to display logs for container `os-node-01` in the terminal window:
      ```bash
      docker logs -f os-node-01
      ```
      {% include copy.html %}
   1. You will see a log entry like the following example when the node is ready:
      ```
      [INFO ][o.o.s.c.ConfigurationRepository] [os-node-01] Node 'os-node-01' initialized
      ```
   1. Press `Ctrl+C` to stop following container logs and return to the command prompt.
1. Use cURL to query the API. In the following command, `os-node-01` is queried by sending the request to host port `9201`, which is mapped to port `9200` on the container:
   ```bash
   curl -s "https://localhost:9201" -ku admin:admin
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
       "name" : "os-node-01",
       "cluster_name" : "opensearch-dev-cluster",
       "cluster_uuid" : "g1MMknuDRuuD9IaaNt56KA",
       "version" : {
           "distribution" : "opensearch",
           "number" : "1.3.7",
           "build_type" : "tar",
           "build_hash" : "db18a0d5a08b669fb900c00d81462e221f4438ee",
           "build_date" : "2022-12-07T22:59:20.186520Z",
           "build_snapshot" : false,
           "lucene_version" : "8.10.1",
           "minimum_wire_compatibility_version" : "6.8.0",
           "minimum_index_compatibility_version" : "6.0.0-beta1"
       },
       "tagline" : "The OpenSearch Project: https://opensearch.org/"
   }
   ```

## Add data and configure OpenSearch Security

Now that the OpenSearch cluster is running, it's time to add data and configure some OpenSearch Security settings. The data you add and settings you configure will be used to validate that these artifacts are preserved through a version upgrade.

You will perform the following steps by:
- [Configuring host and cluster settings](#configuring-host-and-cluster-settings)
- [Adding data using OpenSearch Dashboards](#adding-data-using-opensearch-dashboards)

### Configuring host and cluster settings

These steps walk you through downloading and indexing sample data, and then querying the data to establish a baseline that you can use to validate your cluster's state after the upgrade process is finished.

1. Download the sample field mappings file first:
   ```bash
   wget https://raw.githubusercontent.com/opensearch-project/documentation-website/main/assets/examples/ecommerce-field_mappings.json
   ```
   {% include copy.html %}
1. Next, download the bulk data that you will ingest to this index:
   ```bash
   wget https://raw.githubusercontent.com/opensearch-project/documentation-website/main/assets/examples/ecommerce.json
   ```
   {% include copy.html %}
1. Use the [Create index]({{site.url}}{{site.baseurl}}/api-reference/index-apis/create-index/) API to create an index using the mappings defined in `ecommerce-field_mappings.json`.
   ```bash
   curl -H "Content-Type: application/x-ndjson" -X PUT "https://localhost:9201/ecommerce?pretty" -ku admin:admin --data-binary "@ecommerce-field_mappings.json"
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
      "acknowledged" : true,
      "shards_acknowledged" : true,
      "index" : "ecommerce"
   }
   ```
1. Use the [Bulk]({{site.url}}{{site.baseurl}}/api-reference/document-apis/bulk/) API to add data to the new `ecommerce` index from `ecommerce.json`:
   ```bash
   curl -H "Content-Type: application/x-ndjson" -X PUT "https://localhost:9201/ecommerce/_bulk?pretty" -ku admin:admin --data-binary "@ecommerce.json"
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response (truncated)</p>
   ```json
   {
      "took" : 3323,
      "errors" : false,
      "items" : [
   ...
         "index" : {
            "_index" : "ecommerce",
            "_type" : "_doc",
            "_id" : "4674",
            "_version" : 1,
            "result" : "created",
            "_shards" : {
               "total" : 2,
               "successful" : 2,
               "failed" : 0
            },
            "_seq_no" : 4674,
            "_primary_term" : 1,
            "status" : 201
         }
      ]
   }
   ```
1. Perform an additional check to confirm that the data was written to the `ecommerce` index. The following command queries for documents where `customer_first_name` is `Sonya`. You can compare the response to this command against the response to the same command after upgrading OpenSearch to confirm that the data is intact:
   ```bash
   curl -H 'Content-Type: application/json' -X GET "https://localhost:9201/ecommerce/_search?pretty=true&filter_path=hits.total" -d'{"query":{"match":{"customer_first_name":"Sonya"}}}' -ku admin:admin
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
   "hits" : {
      "total" : {
         "value" : 106,
         "relation" : "eq"
      }
   }
   }
   ```

### Adding data using OpenSearch Dashboards

1. Open a web browser and navigate to port `5601` on your Docker host (for example, <code>https://<var>HOST_ADDRESS</var>:5601</code>). If OpenSearch Dashboards is running, and you have network access to the host from your browser client, then you will be presented with a login page.
    1. If the web browser throws an error because the certificates used by the test cluster are self-signed, you can work around this by bypassing the certificate check in your browser. Remember that the common name (CN) for each certficate is generated with respect to the container and node name for intra-cluster communication, so connecting to the host from a browser will still result in an "invalid CN" error.
1. Enter the default username (`admin`) and password (`admin`).
1. On the OpenSearch Dashboards **Home** page, select **Add sample data**.
1. Under **Sample web logs**, select **Add data**.
   1. Optional: Select **View data** to review the **[Logs] Web Traffic** dashboard.
1. Select the **Menu button** to open the **Navigation pane**, then go to **Security > Internal users**.
1. Select **Create internal user**.
1. Provide a **Username** and **Password**.
1. In the **Backend role** field, enter `admin`.
1. Click **Create**.

## Back up important files

Always create backups before making changes to your cluster, especially if the cluster is running in a production environment.

In this section you will:
- [Register a snapshot repository](#register-a-snapshot-repository)
- [Create a snapshot](#create-a-snapshot)
- [Back up security settings](#back-up-security-settings)
- [Copy backups to an external volume](#copy-backups-to-an-external-volume)

### Register a snapshot repository

1. Register a repository using the volume that was mapped by `upgrade-demo-cluster.sh`:
   ```bash
   curl -H 'Content-Type: application/json' -X PUT "https://localhost:9201/_snapshot/snapshot-repo?pretty" -d '{"type":"fs","settings":{"location":"/usr/share/opensearch/snapshots"}}' -ku admin:admin
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
      "acknowledged" : true
   }
   ```
1. Optional: Perform an additional check to verify that the repository was created successfully:
   ```bash
   curl -H 'Content-Type: application/json' -X POST "https://localhost:9201/_snapshot/snapshot-repo/_verify?timeout=0s&master_timeout=50s&pretty" -ku admin:admin
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
      "nodes" : {
         "UODBXfAlRnueJ67grDxqgw" : {
            "name" : "os-node-03"
         },
         "14I_OyBQQXio8nmk0xsVcQ" : {
            "name" : "os-node-04"
         },
         "tQp3knPRRUqHvFNKpuD2vQ" : {
            "name" : "os-node-02"
         },
         "rPe8D6ssRgO5twIP00wbCQ" : {
            "name" : "os-node-01"
         }
      }
   }
   ```

### Create a snapshot

Snapshots are backups of a cluster’s indexes and state. See [Snapshots]({{site.url}}{{site.baseurl}}/tuning-your-cluster/availability-and-recovery/snapshots/index/) to learn more.

1. Create a snapshot that includes all indexes and the cluster state:
   ```bash
   curl -H 'Content-Type: application/json' -X PUT "https://localhost:9201/_snapshot/snapshot-repo/cluster-snapshot-v137?wait_for_completion=true&pretty" -ku admin:admin
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```json
   {
      "snapshot" : {
         "snapshot" : "cluster-snapshot-v137",
         "uuid" : "-IYB8QNPShGOTnTtMjBjNg",
         "version_id" : 135248527,
         "version" : "1.3.7",
         "indices" : [
            "opensearch_dashboards_sample_data_logs",
            ".opendistro_security",
            "security-auditlog-2023.02.27",
            ".kibana_1",
            ".kibana_92668751_admin_1",
            "ecommerce",
            "security-auditlog-2023.03.06",
            "security-auditlog-2023.02.28",
            "security-auditlog-2023.03.07"
         ],
         "data_streams" : [ ],
         "include_global_state" : true,
         "state" : "SUCCESS",
         "start_time" : "2023-03-07T18:33:00.656Z",
         "start_time_in_millis" : 1678213980656,
         "end_time" : "2023-03-07T18:33:01.471Z",
         "end_time_in_millis" : 1678213981471,
         "duration_in_millis" : 815,
         "failures" : [ ],
         "shards" : {
            "total" : 9,
            "failed" : 0,
            "successful" : 9
         }
      }
   }
   ```

### Back up security settings

Cluster administrators can modify OpenSearch Security settings using any of the following methods:

- Modifying YAML files and running `securityadmin.sh`
- Making REST API requests to  using the admin certificate
- Making changes with OpenSearch Dashboards

Regardless of the method you choose, OpenSearch Security writes your configuration to a special system index called `.opendistro_security`. This system index is preserved through the upgrade process, and it is saved in the snapshot you created in the previous section for protection. However, restoring system indexes requires elevated access granted by the `admin` certificate. To learn more, see [System indexes]({{site.url}}{{site.baseurl}}/security/configuration/system-indices/) and [Configuring TLS certificates]({{site.url}}{{site.baseurl}}/security/configuration/tls/).

You can also export your OpenSearch Security settings by running `securityadmin.sh` with the `-backup` option on any of your OpenSearch nodes. This generates YAML files in a specified directory that you can use to re-initialize the `.opendistro_security` index for failure recovery, as an example. The following steps will guide you through generating these backup files and copying them to your host for storage.

1. Open an interactive pseudo-TTY session with one of the OpenSearch nodes by running the following command on your host:
   ```bash
   docker exec -it os-node-01 bash
   ```
   {% include copy.html %}
1. Create a directory called `backups` and navigate to it:
   ```bash
   mkdir /usr/share/opensearch/backups && cd /usr/share/opensearch/backups
   ```
   {% include copy.html %}
1. Invoke `securityadmin.sh` and create backups of your OpenSearch Security settings in `/usr/share/opensearch/backups/`:
   ```bash
   /usr/share/opensearch/plugins/opensearch-security/tools/securityadmin.sh -backup /usr/share/opensearch/backups \
      -icl \
      -nhnv \
      -cacert /usr/share/opensearch/config/root-ca.pem \
      -cert /usr/share/opensearch/config/admin.pem \
      -key /usr/share/opensearch/config/admin-key.pem
   ```
   {% include copy.html %}
   <p class="codeblock-label">Example response</p>
   ```bash
   Security Admin v7
   Will connect to localhost:9300 ... done
   Connected as CN=A,OU=DOCS,O=OPENSEARCH,L=PORTLAND,ST=OREGON,C=US
   OpenSearch Version: 1.3.7
   OpenSearch Security Version: 1.3.7.0
   Contacting opensearch cluster 'opensearch' and wait for YELLOW clusterstate ...
   Clustername: opensearch-dev-cluster
   Clusterstate: GREEN
   Number of nodes: 4
   Number of data nodes: 4
   .opendistro_security index already exists, so we do not need to create one.
   Will retrieve '/config' into /usr/share/opensearch/backups/config.yml 
      SUCC: Configuration for 'config' stored in /usr/share/opensearch/backups/config.yml
   Will retrieve '/roles' into /usr/share/opensearch/backups/roles.yml 
      SUCC: Configuration for 'roles' stored in /usr/share/opensearch/backups/roles.yml
   Will retrieve '/rolesmapping' into /usr/share/opensearch/backups/roles_mapping.yml 
      SUCC: Configuration for 'rolesmapping' stored in /usr/share/opensearch/backups/roles_mapping.yml
   Will retrieve '/internalusers' into /usr/share/opensearch/backups/internal_users.yml 
      SUCC: Configuration for 'internalusers' stored in /usr/share/opensearch/backups/internal_users.yml
   Will retrieve '/actiongroups' into /usr/share/opensearch/backups/action_groups.yml 
      SUCC: Configuration for 'actiongroups' stored in /usr/share/opensearch/backups/action_groups.yml
   Will retrieve '/tenants' into /usr/share/opensearch/backups/tenants.yml 
      SUCC: Configuration for 'tenants' stored in /usr/share/opensearch/backups/tenants.yml
   Will retrieve '/nodesdn' into /usr/share/opensearch/backups/nodes_dn.yml 
      SUCC: Configuration for 'nodesdn' stored in /usr/share/opensearch/backups/nodes_dn.yml
   Will retrieve '/whitelist' into /usr/share/opensearch/backups/whitelist.yml 
      SUCC: Configuration for 'whitelist' stored in /usr/share/opensearch/backups/whitelist.yml
   Will retrieve '/audit' into /usr/share/opensearch/backups/audit.yml 
      SUCC: Configuration for 'audit' stored in /usr/share/opensearch/backups/audit.yml
   ```
1. Optional: Create a backup directory for TLS certificates and copy the certificates. Repeat this for each node they are configured with unique TLS certificates:
   ```bash
   mkdir /usr/share/opensearch/backups/certs && cp /usr/share/opensearch/config/*pem /usr/share/opensearch/backups/certs/
   ```
   {% include copy.html %}
1. Terminate the pseudo-TTY session:
   ```bash
   exit
   ```
   {% include copy.html %}
1. Copy the backups you generated to your host:
   ```bash
   docker cp os-node-01:/usr/share/opensearch/backups ~/deploy/
   ```
   {% include copy.html %}

### Copy backups to an external volume


## Next steps:
- [Quickstart guide for OpenSearch Dashboards]({{site.url}}{{site.baseurl}}/dashboards/quickstart-dashboards/)
