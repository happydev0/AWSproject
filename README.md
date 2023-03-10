# AWS Lambda Node.js Example Project

**DEPRECATED**: This project is no longer maintained. If you wish to process a Kinesis stream of Snowplow events using an AWS Lambda application, we recommend using the [Snowplow JavaScript and TypeScript Analytics SDK][analytics-sdk].

[![Build Status][travis-image]][travis] [![Release][release-image]][releases] [![License][license-image]][license]

## Introduction

This is an example [AWS Lambda][aws-lambda] application for processing a [Kinesis][aws-kinesis] stream of events ([introductory blog post][blog-post]). It reads the stream of simple JSON events generated by our event generator. Our AWS Lambda function aggregates and buckets events and stores them in [DynamoDB][aws-dynamodb].

This was built by the Data Science team at [Snowplow Analytics][snowplow], who use AWS Lambda in their projects.

**Running this requires an Amazon AWS account, and will incur charges.**

_See also:_ [Spark Streaming Example Project][spark-streaming-example-project] | [Spark Example Project][spark-example-project]

## Overview

We have implemented a super-simple analytics-on-write stream processing job using AWS Lambda. Our AWS Lambda function, written in JavaScript, reads a Kinesis stream containing events in a JSON format:

```json
{
  "timestamp": "2015-06-05T12:54:43.064528",
  "type": "Green",
  "id": "4ec80fb1-0963-4e35-8f54-ce760499d974"
}
```

Our job counts the events by `type` and aggregates these counts into 1 minute buckets. The job then takes these aggregates and saves them into a table in DynamoDB:

![dynamodb-table-image][dynamodb-table-image]

## Developer Quickstart

Assuming git, [Vagrant][vagrant-install] and [VirtualBox][virtualbox-install] installed:

```bash
 host$ git clone https://github.com/snowplow/aws-lambda-nodejs-example-project.git
 host$ cd aws-lambda-example-project
 host$ vagrant up && vagrant ssh
guest$ cd /vagrant
guest# npm install grunt
guest$ npm install
guest$ grunt --help
```

## Tutorial

You can follow along in [the release blog post][blog-post] to get the project up and running yourself.

The following steps assume that you are running inside Vagrant, as per the Developer Quickstart above.

### 1. Setting up AWS

First we need to configure a default AWS profile:

```bash
$ aws configure
AWS Access Key ID [None]: ...
AWS Secret Access Key [None]: ...
Default region name [None]: us-east-1
Default output format [None]: json
```

Now we can create our DynamoDB table, Kinesis stream, and IAM role. We will be using [CloudFormation](http://aws.amazon.com/cloudformation) to make our new role. Using Grunt, we can create all like so:

```bash
$ grunt init
Running "dynamo:default" (dynamo) task
{ TableDescription:
   { AttributeDefinitions: [ [Object], [Object], [Object] ],
     CreationDateTime: Sun Jun 28 2015 13:04:02 GMT-0700 (PDT),
     ItemCount: 0,
     KeySchema: [ [Object], [Object] ],
     LocalSecondaryIndexes: [ [Object] ],
     ProvisionedThroughput:
      { NumberOfDecreasesToday: 0,
        ReadCapacityUnits: 20,
        WriteCapacityUnits: 20 },
     TableName: 'my-table',
     TableSizeBytes: 0,
     TableStatus: 'CREATING' } }

Running "createRole:default" (createRole) task
{ ResponseMetadata: { RequestId: 'd29asdff0-1dd0-11e5-984e-35a24700edda' },
  StackId: 'arn:aws:cloudformation:us-east-1:84asdf429716:stack/kinesisDynamo/d2af8730-1dd0-11e5-854a-50d5017c76e0' }

Running "kinesis:default" (kinesis) task
{}

Done, without errors.
```

### 2. Connect AWS Lambda service with the new role and building the project

Wait a minute to ensure our IAM service role gets created. Now we connect the new service role to access Kinesis, CloudWatch, Lambda, and DynamoDB. We will attach an admin policy to the lambda exec role to easily access the services. Using Grunt, our AWS Lambda function gets assembled into a zip file for upload to the AWS Lambda service. Once it's zipped, we attach a service role to it:

```bash
$ grunt role
Running "attachRole:default" (attachRole) task
{ ResponseMetadata: { RequestId: '36ac7877-1dca-11e5-b439-d1da60d122be' } }

Running "packaging:default" (packaging) task
aws-lambda-example-project@0.1.0 ../../../../var/folders/3t/7nlz8rzs2mq5fg_sf3x4j7_m0000gn/T/1435519004662.0046/node_modules/aws-lambda-example-project
????????? rimraf@2.2.8
????????? async@0.9.2
????????? temporary@0.0.8 (package@1.0.1)
????????? mkdirp@0.5.1 (minimist@0.0.8)
????????? glob@4.3.5 (inherits@2.0.1, once@1.3.2, inflight@1.0.4, minimatch@2.0.8)
????????? lodash@3.9.3
????????? archiver@0.14.4 (buffer-crc32@0.2.5, lazystream@0.1.0, readable-stream@1.0.33, tar-stream@1.1.5, zip-stream@0.5.2, lodash@3.2.0)
????????? aws-sdk@2.1.23 (xmlbuilder@0.4.2, xml2js@0.2.8, sax@0.5.3)
Created package at dist/aws-lambda-example-project_0-1-0_latest.zip
...
```

### 3. Deploy zip file to AWS Lambda service and connect Kinesis to Lambda

In deploy this project to Lambda with the `grunt deploy` command:

```bash
$ grunt deploy
Running "deployLambda:default" (deployLambda) task
Trying to create AWS Lambda Function...
Created AWS Lambda Function...
```

### 4. Connect Kinesis to Lambda

The final step to getting this projected ready to start processing events is to associate our Kinesis stream to the Lambda function with this command:

```bash
$ grunt connect
Running "associateStream:default" (associateStream) task
arn:aws:kinesis:us-east-1:844709429716:stream/my-stream
{ BatchSize: 100,
  EventSourceArn: 'arn:aws:kinesis:us-east-1:2349429716:stream/my-stream',
  FunctionArn: 'arn:aws:lambda:us-east-1:2349429716:function:ProcessKinesisRecordsDynamo',
  LastModified: Sun Jun 28 2015 12:38:37 GMT-0700 (PDT),
  LastProcessingResult: 'No records processed',
  State: 'Creating',
  StateTransitionReason: 'User action',
  UUID: 'f4efc-fe72-4337-9907-89d4e64c' }

Done, without errors.
```

### 5. Sending events to Kinesis

We need to start sending events to our new Kinesis stream. We have created a helper method to do this - run the below and leave it running in a tab:

```bash
$ grunt events
Writing Kineis Event: {"timestamp":"2015-06-29T20:12:21.625Z","type":"Red"}
{ SequenceNumber: '49552099319153062484931809176874704852938278389141209090',
  ShardId: 'shardId-000000000000' }
Writing Kineis Event: {"timestamp":"2015-06-29T20:12:22.200Z","type":"Red"}
{ SequenceNumber: '49552099319153062484931809176875913778757893018315915266',
  ShardId: 'shardId-000000000000' }
Writing Kineis Event: {"timestamp":"2015-06-29T20:12:22.708Z","type":"Green"}
{ SequenceNumber: '49552099319153062484931809176877122704577507716210098178',
  ShardId: 'shardId-000000000000' }
...
```

### 6. Monitoring your job

First head over to the AWS Lambda service console, then review the logs in CloudWatch.

Finally, let's check the data in our DynamoDB table. Make sure you are in the correct AWS region, then click on `my-table` and hit the `Explore Table` button:

![dynamodb-table-image][dynamodb-table-image]

For each **BucketStart** and **EventType** pair, we see a **Count**, plus some **CreatedAt** and **UpdatedAt** metadata for debugging purposes. Our bucket size is 1 minute, and we have 5 discrete event types, hence the matrix of rows that we see.

## Roadmap

* Various improvements for the [0.2.0 release][020-milestone]
* Expanding our analytics-on-write thinking into our new [Icebucket][icebucket] project

## Credits

* [Tim Bell] [tim-b] for his blog post [Writing Functions for AWS Lambda Using NPM and Grunt][tim-b-post]
* Ian Meyers and his [Amazon-Kinesis-Aggregators-Project][amazon-kinesis-aggregators], a true inspiration for streaming analytics-on-write

## Copyright and license

AWS Lambda Example Project is copyright 2015 Snowplow Analytics Ltd.

Licensed under the **[Apache License, Version 2.0][license]** (the "License");
you may not use this software except in compliance with the License.

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

[analytics-sdk]: https://github.com/snowplow-incubator/snowplow-js-analytics-sdk
[travis]: https://travis-ci.org/snowplow/aws-lambda-nodejs-example-project
[travis-image]: https://travis-ci.org/snowplow/aws-lambda-nodejs-example-project.png?branch=master
[license-image]: https://img.shields.io/badge/license-Apache--2-blue.svg?style=flat
[license]: https://www.apache.org/licenses/LICENSE-2.0
[release-image]: https://img.shields.io/badge/release-0.1.0-blue.svg?style=flat
[releases]: https://github.com/snowplow/aws-lambda-nodejs-example-project/releases
[grunt-image]: https://cdn.gruntjs.com/builtwith.png

[spark-example-project]: https://github.com/snowplow/spark-example-project
[spark-streaming-example-project]: https://github.com/snowplow/spark-streaming-example-project

[vagrant-install]: http://docs.vagrantup.com/v2/installation/index.html
[virtualbox-install]: https://www.virtualbox.org/wiki/Downloads

[blog-post]: http://snowplowanalytics.com/blog/2015/07/11/aws-lambda-nodejs-example-project-0.1.0-released/
[020-milestone]: https://github.com/snowplow/aws-lambda-nodejs-example-project/milestones/Version%200.2.0
[dynamodb-table-image]: /docs/dynamodb-table-image.png?raw=true

[aws-lambda]: http://aws.amazon.com/lambda/
[aws-kinesis]: http://aws.amazon.com/kinesis/
[aws-dynamodb]: http://aws.amazon.com/dynamodb
[vagrant-install]: http://docs.vagrantup.com/v2/installation/index.html
[virtualbox-install]: https://www.virtualbox.org/wiki/Downloads
[tim-b]: https://github.com/Tim-B
[tim-b-post]: http://hipsterdevblog.com/blog/2014/12/07/writing-functions-for-aws-lambda-using-npm-and-grunt/
[amazon-kinesis-aggregators]: https://github.com/awslabs/amazon-kinesis-aggregators

[snowplow]: http://snowplowanalytics.com
[icebucket]: https://github.com/snowplow/icebucket
