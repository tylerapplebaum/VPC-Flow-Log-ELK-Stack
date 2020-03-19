# VPC Flow Log ELK Stack

### Background

Visualize VPC flow logs using CLoudWatch Logs, Lambda, and Elasticsearch.

This implementation uses a VPC-only Elasticsearch domain. You can simplify this by creating a publicly accessible Elasticsearch domain.

### Amazon Workspace

- Create SimpleAD directory
- Create Workspace with Amazon Linux
- Deploy the Workspace in the same VPC as the Elasticsearch domain

### Elasticsearch Service

- Create an Elasticsearch domain

Using the AWS Workspace, navigate to the Kibana endpoint. Create an Index Template so that IP addresses use the `ip` data type. This will allow you to filter using [CIDR ranges](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing).

```
PUT _template/cwl-vpc
{
    "index_patterns": ["cwl-vpcflowlogs*"],
    "settings": {
        "number_of_shards": 1
    },
    "mappings": {
        "properties" : {
            "dstaddr" : {
              "type" : "ip",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "srcaddr" : {
              "type" : "ip",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            }
        }
    }
}
```
Create visualization with this as the filter:

`NOT srcaddr:"172.16.0.0/12" AND NOT srcaddr:"10.0.0.0/8" AND NOT srcaddr:"192.168.0.0/16" AND NOT srcaddr:"100.64.0.0/10"`

### LogsToElasticsearch_vpcflowlogs

The Lambda uses the Node.js 10.x runtime. **Be sure to edit the Lambda to include your own Elasticsearch endpoint.**

The AWS default Lambda uses the CloudWatch Log Group name as the Index Type. This allows you the ability to stream multiple types of log groups to the same ES domain.

[LogsToElasticsearch_vpcflowlogs](LogsToElasticsearch_vpcflowlogs.js)

### IAM Roles

Follow [this official guide](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-cwl.html) to set up the correct IAM roles. This will need some customization to use a VPC-only Elasticsearch domain.

### Enabling VPC Flow Logs

Follow [this official guide](https://aws.amazon.com/blogs/aws/vpc-flow-logs-log-and-view-network-traffic-flows/) to enable VPC Flow Logs. It will take a few minutes to get data streamed into your new ELK stack.

### Miscellaneous

 With ElasticSearch 7.0, you need to use the default index type of `_doc`. If you do not customize the default Lambda, you will be unable to change the data type of the IP addresses.

You'll receive an error similar to below.

```
"index": {
    "_index": "cwl-vpcflowlogs-2020.03.11",
    "_type": "VPCFlowLogs",
    "_id": "35323290557054654866554530370298807650594967459719806976",
    "status": 400,
    "error": {
        "type": "illegal_argument_exception",
        "reason": "Rejecting mapping update to [cwl-vpcflowlogs-2020.03.11] as the final mapping would have more than 1 type: [_doc, VPCFlowLogs]"
    }
}
```
