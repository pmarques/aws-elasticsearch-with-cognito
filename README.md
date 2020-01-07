# AWS Managed Elasticsearch with Cognito RBAC

This is compact solution / template to show how you can configure AWS Elasticsearch with cognito and apply some restrictions.

1. Create a CloudFormation Stack with elastisearch.yaml
2. Create users in the AWS Cognito User Pool console and assign on of the groups
3. Access the kibana, you can find the URL in the Outputs of the CloudFormation Stack

## Tests

Using Kibana Dev Tools:

### Admin
Only users in the **Admin** group can create docs:

```
PUT twitter/_doc/1
{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elasticsearch"
}
```

### Developers

Users in other groups can only read docs in their projects:

* **ProjectAGroup** can readon from **-proj-a** cluster
* **ProjectBGroup** can readon from **-proj-b** cluster

```
GET twitter/_doc/1
```

## Findings

Before discussing the issues, keep in mind ES works like this:

* ES has REST API and basic actions are CRUD (Create, Read, Update and Delete)
* ES also allows "less CRUD APIs" (at least purists would say that) and by that I mean: they allow multi search and bulk operations where the index can be specified on the HTTP request Body

At the time, 6th of January 2020:

* AWS ES as Cognito integration only allows, as far as I know, to block requests based on Path. That prevents someone to read, modify or create a document using the Document "CRUD" API.
* Requests with the index in the HTTP Body bypass Cognito IAM Policy, there is a way to disable this ni the ES, although if we do that kibana stops working because loads of things are apparently based on multi search.
* Since multi search is required for Kibana we can't disable `_msearch`
* `_bulk` ES REST API can be used as a _"hack"_ for write requests

To block it we can add a Deny statement for the `_bulk` requests.

Other APIs would need to be evaluated to check other possible issues

* I'm not entirely sure about the current implementation of beats and logstash, but they were using bulk request to index the logs and metrics which means that if we block this on ES we would probably break the log ingestion.
* We can block for instance _bulk in the Cognito IAM Policy although we need to evaluate all the APIs to make sure that other "hacks" can't be used to at leasts edit the data

Also:

* AWS ES seems to white list the ES API, so explicit Denies in the IAM Policies are required to block this _"hacks"_
* AWS ES Service doesn't support cross cluster search so we can segregate data per AWS ES Domain.

**NOTE:** The CloudFormation template has a explicit Deny to block `_bulk` requests, you can remove that and test this in the Kibana Dev Tools:

Bypass write permissions with `_bulk` (no index in the HTTP path):

```
POST _bulk
{ "index" : { "_index" : "test", "_id" : "1" } }
{ "field1" : "value1" }
```

Test block multi-search when using the index in the HTTP Path:

```
GET test/_msearch
{}
{"query" : {"match_all" : {}}, "from" : 0, "size" : 10}
```

Bypass read permissions with `_msearch` (no index in the HTTP path):

```
GET _msearch
{"index" : "test"}
{"query" : {"match_all" : {}}}
```