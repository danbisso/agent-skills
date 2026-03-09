# CDK Construct Reference (aws-cdk-lib v2)

Quick lookup for import paths, the `.grantX()` matrix, and core value types. All imports are submodules of the single `aws-cdk-lib` package; `Construct` comes from `constructs`.

## Imports

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';

import * as lambda from 'aws-cdk-lib/aws-lambda';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { SqsEventSource, DynamoEventSource } from 'aws-cdk-lib/aws-lambda-event-sources';

import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as sqs from 'aws-cdk-lib/aws-sqs';
import * as sns from 'aws-cdk-lib/aws-sns';
import * as subs from 'aws-cdk-lib/aws-sns-subscriptions';

import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as scheduler from 'aws-cdk-lib/aws-scheduler';
import * as schedulerTargets from 'aws-cdk-lib/aws-scheduler-targets';

import * as sfn from 'aws-cdk-lib/aws-stepfunctions';
import * as tasks from 'aws-cdk-lib/aws-stepfunctions-tasks';

import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import { ApplicationLoadBalancedFargateService } from 'aws-cdk-lib/aws-ecs-patterns';

import * as apigw from 'aws-cdk-lib/aws-apigateway';            // REST API
import * as apigwv2 from 'aws-cdk-lib/aws-apigatewayv2';        // HTTP/WebSocket API
import * as apigwv2int from 'aws-cdk-lib/aws-apigatewayv2-integrations';

import * as iam from 'aws-cdk-lib/aws-iam';
import * as kms from 'aws-cdk-lib/aws-kms';
import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
import * as ssm from 'aws-cdk-lib/aws-ssm';
import * as cloudwatch from 'aws-cdk-lib/aws-cloudwatch';
import * as cwActions from 'aws-cdk-lib/aws-cloudwatch-actions';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as rds from 'aws-cdk-lib/aws-rds';
```

## `.grantX()` matrix (prefer these over hand-written IAM)

| Resource | Common grants |
|---|---|
| `dynamodb.TableV2` / `Table` | `grantReadData`, `grantWriteData`, `grantReadWriteData`, `grantFullAccess`, `grantStreamRead` |
| `s3.Bucket` | `grantRead`, `grantWrite`, `grantReadWrite`, `grantPut`, `grantDelete` |
| `sqs.Queue` | `grantSendMessages`, `grantConsumeMessages`, `grantPurge` |
| `sns.Topic` | `grantPublish`, `grantSubscribe` |
| `lambda.Function` | `grantInvoke`, `grantInvokeUrl` |
| `secretsmanager.Secret` | `grantRead`, `grantWrite` |
| `ssm.StringParameter` | `grantRead`, `grantWrite` |
| `kms.Key` | `grantEncrypt`, `grantDecrypt`, `grantEncryptDecrypt` |
| `sfn.StateMachine` | `grantStartExecution`, `grantRead` |
| `events.EventBus` | `grantPutEventsTo` |

Each grant computes the minimal action set, scopes to the resource ARN, and — for encrypted resources — adds the KMS key grant automatically. The grantee is any `IGrantable` (a Function, Role, User, or anything with an exec role).

If you must write raw IAM, scope it:

```ts
fn.addToRolePolicy(new iam.PolicyStatement({
  actions: ['dynamodb:Query'],
  resources: [table.tableArn, `${table.tableArn}/index/*`],
}));
```

## Core value types

```ts
cdk.Duration.seconds(30); cdk.Duration.minutes(5); cdk.Duration.days(7);
cdk.Size.mebibytes(512); cdk.Size.gibibytes(1);

cdk.RemovalPolicy.RETAIN;   // keep on stack delete — use for data stores
cdk.RemovalPolicy.DESTROY;  // delete — dev/throwaway only
cdk.RemovalPolicy.SNAPSHOT; // RDS/DynamoDB: snapshot then delete

lambda.Runtime.NODEJS_22_X;
lambda.Architecture.ARM_64; // cheaper; set on Function for Graviton
dynamodb.AttributeType.STRING | NUMBER | BINARY;
dynamodb.Billing.onDemand(); dynamodb.Billing.provisioned({ readCapacity, writeCapacity });
ec2.InstanceClass.T4G; ec2.InstanceSize.MICRO;
```

## Importing existing resources (don't recreate)

```ts
const bucket = s3.Bucket.fromBucketName(this, 'Existing', 'my-bucket');
const secret = secretsmanager.Secret.fromSecretNameV2(this, 'Db', 'prod/db');
const vpc = ec2.Vpc.fromLookup(this, 'Vpc', { isDefault: false, tags: { env: 'prod' } });
const param = ssm.StringParameter.fromStringParameterName(this, 'Cfg', '/app/config');
```

`fromLookup` reads real account state at synth and caches into `cdk.context.json` (commit it). `from*Name`/`from*Arn` are pure references — no lookup, no validation.

## CLI cheat-sheet

```bash
cdk bootstrap aws://ACCOUNT/REGION   # once per account+region
cdk synth                            # emit CloudFormation to cdk.out/
cdk diff App-Stateless               # ALWAYS before deploy; read replacements
cdk deploy App-Stateless --require-approval broadening
cdk deploy --all --concurrency 4
cdk destroy App-Stateless
cdk ls                               # list stacks
cdk doctor                           # environment sanity
```

`--require-approval broadening` prompts only when IAM/security broadens — good CI default. Never use `--require-approval never` on prod without a reviewed diff in the pipeline.
