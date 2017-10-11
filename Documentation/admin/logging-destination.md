# Customizing log destination

In order for Fluentd to send your logs to a different destination, you will need to use different Docker image with the correct Fluentd plugin for your destination. Once you have an image, you need to replace the contents of the `output.conf` section in your [fluentd-configmap.yaml][fluentd-config] with the appropriate [match directive][fluentd-match] for your output plugin.

## Prebuilt images

There are currently 4 prebuilt Debian based Docker images in the [quay.io/coreos/fluentd-kubernetes][quay-fluentd-Kubernetes] registry available for various logging destinations:

- [quay.io/coreos/fluentd-kubernetes:v0.12-debian-cloudwatch][quay-fluentd-kubernetes]
- [quay.io/coreos/fluentd-kubernetes:v0.12-debian-logentries][quay-fluentd-kubernetes]
- [quay.io/coreos/fluentd-kubernetes:v0.12-debian-loggly][quay-fluentd-kubernetes]
- [quay.io/coreos/fluentd-kubernetes:v0.12-debian-elasticsearch][quay-fluentd-kubernetes]

**Note**: there are Alpine based images which are automatically published along side the Debian images, but they cannot be used in conjunction with the systemd input plugin, because Alpine has no `libsystemd` package available.

To use one of these images, update the `image` field in your [fluentd-ds.yaml][fluentd-ds] manifest, and update your [fluentd-configmap.yaml][fluentd-config] `output.conf` with the correct match configuration for your configured output plugin.

If you deploy Elasticsearch into your cluster, ensure the hostname and port of the service match the value in the `output.conf` section of your [fluentd-configmap.yaml][fluentd-config].

### Using a different storage destination than Elasticsearch

To change where your logs are sent, change the image in [fluentd-ds.yaml][fluentd-ds] to an image providing the necessary output plugin. The `output.conf` stanza in [fluentd-configmap.yaml][fluentd-config] must also be updated to match the new output plugin.

#### Logentries

To change to the logentries image, replace the line containing `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-elasticsearch` with `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-logentries`.

Next, update your [fluentd-configmap.yaml][fluentd-config] `output.conf`:

```
<match **>
  # Plugin specific settings
  type logentries
  config_path /etc/logentries/logentries-tokens.conf

  # Buffer settings
  buffer_chunk_limit 2M
  buffer_queue_limit 32
  flush_interval 10s
  max_retry_wait 30
  disable_retry_limit
  num_threads 8
</match>
```

**Note**: You will need to also modify your `fluentd-ds.yaml` to add a secret/volumeMount for your `logentries-token.conf` referenced in the config above.

#### Cloudwatch

To change to the cloudwatch image, replace the line containing `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-elasticsearch` with `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-cloudwatch`. You will also need to create an IAM user and set the `AWS_ACCESS_KEY`, `AWS_SECRET_KEY` and `AWS_REGION` environment variables in your manifest [as documented in the plugin's README](https://github.com/ryotarai/fluent-plugin-cloudwatch-logs#preparation). We recommend you use secrets [as environment variables](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables) to accomplish setting the environment variables securely.

Next, update your [fluentd-configmap.yaml][fluentd-config] `output.conf`:

```
<match **>
  # Plugin specific settings
  type cloudwatch_logs
  log_group_name your-log-group
  log_stream_name your-log-stream
  auto_create_stream true

  # Buffer settings
  buffer_chunk_limit 2M
  buffer_queue_limit 32
  flush_interval 10s
  max_retry_wait 30
  disable_retry_limit
  num_threads 8
</match>
```

#### Loggly

To change to the cloudwatch image, replace the line containing `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-elasticsearch` with `image: quay.io/coreos/fluentd-kubernetes:v0.12-debian-loggly`.

Next, update your [fluentd-configmap.yaml][fluentd-config] `output.conf` (replace `xxx-xxxx-xxxx-xxxxx-xxxxxxxxxx` with your loggly customer token):

```
<match **>
  # Plugin specific settings
  type loggly_buffered
  loggly_url https://logs-01.loggly.com/bulk/xxx-xxxx-xxxx-xxxxx-xxxxxxxxxx

  # Buffer settings
  buffer_chunk_limit 2M
  buffer_queue_limit 32
  flush_interval 10s
  max_retry_wait 30
  disable_retry_limit
  num_threads 8
</match>
```

### Elasticsearch Authentication

If your Elasticsearch cluster is configured with x-pack or authentication methods, you will need to modify the `output.conf` section of your [fluentd-configmap.yaml][fluentd-config] to set credentials.

Installing the x-pack plugin on your Elasticsearch nodes enables authentication to Elasticsearch by default. The default user is `elastic` and the default password is `changeme`. Modify the configuration to include the `user` and `password` fields like so:

```
<match **>
  type elasticsearch
  log_level info
  include_tag_key true

  # Connection settings
  host elasticsearch.default.svc.cluster.local
  port 9200
  scheme https
  ssl_verify true
  user elastic
  password changeme

  logstash_format true
  template_file /fluentd/etc/elasticsearch-template-es5x.json
  template_name elasticsearch-template-es5x.json

  # Buffer settings
  buffer_chunk_limit 2M
  buffer_queue_limit 32
  flush_interval 5s
  max_retry_wait 30
  disable_retry_limit
  num_threads 8
</match>
```

### AWS Elasticsearch

Use AWS Elasticsearch Service with minimal overhead. Let's walk through the steps we're going to take to do this is to set up an Elasticsearch Service on AWS.
  		  
 - Log in to the AWS console and click Elasticsearch service.
 - Click "Create a new domain".
 - Configure the instance type, disk, and permissions. Use IAM based credentials for security. If you're not familiar with IAM users, you can read more about [IAM credentials][iam-credentials]. Here's an example config that I made specifically for this purpose. 

If you do not wish to use credentials in your configuration via the `access_key_id` and `secret_access_key` options you should use IAM policies.

First, assign an IAM instance role `ROLE` to your EC2 instances, and name it appropriately. Do not include a policy in the role. The possession of the role will be used as the authenticating factor, and to place the policy against the ES cluster.

Then, configure a policy for the ES cluster. Replace capitalized terms in the following example with the appropriate values for the cluster.
 
```
{
  "Version": "2017-08-14",
   "Statement": [
     {
       "Effect": "Allow",
       "Principal": {
         "AWS": "arn:aws:iam::ACCOUNT:role/ROLE"
       },
       "Action": "es:*",
       "Resource": "arn:aws:es:us-east-1:ACCOUNT:domain/ES_DOMAIN/*"
     },
     {
       "Effect": "Allow",
       "Principal": {
         "AWS": "*"
       },
       "Action": "es:*",
       "Resource": "arn:aws:es:us-east-1:ACCOUNT:domain/ES_DOMAIN/*",
       "Condition": {
         "IpAddress": {
           "aws:SourceIp": "XXX.XX.XXX.XX/32"
         }
       }
     }
   ]
}
```
This will allow fluentd hosts (by virtue of the possession of the role) and any traffic coming from the specified IP addresses (queries to Kibana) to access the listed endpoints. For greatest security, both the fluentd and Kibana boxes should be restricted to the verbs they require. This less secure example allows the cluster to begin ingesting logs before the policy is fully secured.

Additionally, you may also use an STS assumed role as the authenticating factor and instruct the plugin to assume this role. This is useful for cross-account access and when assigning a standard role is not possible. The endpoint configuration looks like:

 ```
  <endpoint>
     url https://CLUSTER_ENDPOINT_URL
     region eu-east-1
     assume_role_arn arn:aws:sts::ACCOUNT:assumed-role/ROLE
     assume_role_session_name SESSION_ID # Defaults to fluentd if omitted
  </endpoint>  
 ```
 
Define the policy attached to the AWS Elasticsearch cluster with the following format:
 
 ```
 {
   "Version": "2017-08-15",
   "Statement": [
     {
       "Effect": "Allow",
       "Principal": {
         "AWS": "arn:aws:sts::ACCOUNT:assumed-role/ROLE/SESSION_ID"
       },
       "Action": "es:*",
       "Resource": "arn:aws:es:eu-east-1:ACCOUNT:domain/ES_DOMAIN/*"
     }
   ]
 }
 ```
 
Attach a policy to the instance profile to ensure that the environment in which the fluentd plugin runs has the ability to assume the STS role. For example:
 
 ```
 {
     "Version": "2017-08-15",
     "Statement": {
         "Effect": "Allow",
         "Action": "sts:AssumeRole",
         "Resource": "arn:aws:iam::ACCOUNT:role/ROLE"
     }
 }
 ```
 
Next, configure fluentd to gather the selected logs and deliver them to the Elasticsearch machine. Use the [td-agent configuration file][td-config]. We're going to add the below configuration in the Kubernetes repository for the [fluentd plugin][fluentd-plugin].
 
 ```
 <match **>
   type aws-elasticsearch-service
   type_name "access_log"
   log_level info
   logstash_format true
   include_tag_key true
   flush_interval 60s
 
   buffer_type memory
   buffer_chunk_limit 256m
   buffer_queue_limit 128
 
   <endpoint>
     url https://${AWS_ES_ENDPOINT}
     region ${AWS_ES_REGION}
     access_key_id ${AWS_ACCESS_KEY_ID}
     secret_access_key ${AWS_SECRET_ACCESS_KEY}
   </endpoint>
 </match>
 ```

[fluentd-ds]: ../files/logging/fluentd-ds.yaml
[fluentd-config]: ../files/logging/fluentd-configmap.yaml
[fluentd-docs-output]: http://docs.fluentd.org/v0.12/articles/output-plugin-overview
[fluentd-match]: http://docs.fluentd.org/v0.12/articles/config-file#2-ldquomatchrdquo-tell-fluentd-what-to-do
[fluentd-plugin]:
 https://github.com/kubernetes/kubernetes/blob/7e1b9dfd0fc75311ff6339f19b514e8caaebeafd/cluster/addons/fluentd-elasticsearch/fluentd-es-image/td-agent.conf
[iam-credentials]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html
[quay-fluentd-kubernetes]: https://quay.io/repository/coreos/fluentd-kubernetes?tab=tags
[td-config]: https://docs.treasuredata.com/articles/td-agent
