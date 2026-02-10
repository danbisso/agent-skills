---
name: aws-cdk
description: Author AWS infrastructure-as-code in AWS CDK (TypeScript) like an expert — pick the right construct level (L1/L2/L3), structure App→Stack→Construct trees and environments, write L2 idioms (Lambda, DynamoDB, S3, SQS/SNS, EventBridge, Step Functions, Fargate, VPC, IAM grants, Secrets/SSM), build reusable L3 constructs, reach for the right community construct (cdk-nag, Solutions Constructs, projen, cdk-monitoring-constructs), and test with aws-cdk-lib/assertions. Use when writing or reviewing a CDK stack/construct, choosing constructs, using escape hatches, wiring IAM least-privilege, splitting stateful/stateless stacks, or adding cdk-nag/assertion tests.
---

# AWS CDK (TypeScript)

You write the infrastructure code. `aws-architect` decides WHAT to build (which services, topology, trade-offs); you turn that decision into a correct, least-privilege, testable CDK app. Lambda handler code follows `clean-code`; tests follow `tdd`. If the task is "should we use SQS or Kinesis", that is `aws-architect`, not you.

Default to `aws-cdk-lib` v2 (single package, `import { aws_* } from 'aws-cdk-lib'` or `import * as lambda from 'aws-cdk-lib/aws-lambda'`) and the v2 `Construct` from the `constructs` package. Never mix v1 `@aws-cdk/*` packages. See `construct-reference.md` for the per-service import + grant cheat-sheet.

## Construct levels — know which one you're holding

- **L1 (`Cfn*`)** — 1:1 generated CloudFormation. Every property is raw, named exactly as in the CFN spec, mostly optional even when required at deploy time. Drop to L1 only when: an L2 doesn't exist yet, a brand-new CFN property hasn't reached the L2, or you need an escape hatch. You give up validation and sensible defaults.
- **L2 (curated, e.g. `Function`, `Bucket`, `Table`)** — the default. Sensible defaults, `.grantX()` methods, helper methods, type-safe enums, automatic IAM + dependency wiring. Use these for ~everything.
- **L3 (patterns, e.g. `ApplicationLoadBalancedFargateService`)** — opinionated multi-resource compositions. Great accelerators; accept their opinions or build your own L3.

### Escape hatches — when the L2 won't let you set something

Reach through the L2 to the underlying L1 instead of dropping the whole resource to L1.

```ts
const fn = new lambda.Function(this, 'Handler', { /* ... */ });

// Access the L1 child:
const cfnFn = fn.node.defaultChild as lambda.CfnFunction;
cfnFn.addPropertyOverride('ReservedConcurrentExecutions', 5);
cfnFn.addPropertyOverride('SnapStart.ApplyOn', 'PublishedVersions');

// Override a resource-level attribute (not a property):
cfnFn.addOverride('Metadata.cdk_nag.rules_to_suppress', [/* ... */]);
cfnFn.addDeletionOverride('Properties.TracingConfig'); // remove a synthesized value

// Raw escape for a construct with no L1 child handle:
const cfnBucket = bucket.node.findChild('Resource') as s3.CfnBucket;
```

`addPropertyOverride` patches synthesized template JSON by path — use it sparingly, document why, and prefer a real L2 prop if one exists.

## App structure

`App` → one or more `Stack`s → `Construct` tree. A Stack is the unit of deployment (one CloudFormation stack). Group by deploy cadence and blast radius, NOT by AWS service.

```ts
const app = new cdk.App();
const env = { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION };

new StatefulStack(app, 'App-Stateful', { env });   // DBs, buckets, queues — long-lived
new StatelessStack(app, 'App-Stateless', { env }); // Lambdas, APIs — redeploy freely
```

- **Always set `env` explicitly** for anything non-trivial. An env-agnostic stack (`env` omitted) can't use AZ lookups, `fromLookup`, or account-specific logic. Pass `account`/`region` from context or config per environment, never hard-code prod values inline.
- **Stages** (`cdk.Stage`) bundle a set of stacks as one deployable unit — use them for dev/staging/prod promotion and CDK Pipelines. Same code, different `env` + config object.
- **Stateful vs stateless split** (memory: keep these separate): a bad stateless deploy must never threaten the database. Put DynamoDB/RDS/S3/SQS in one stack, compute/APIs in another.
- **Feature flags** live in `cdk.json` under `context` (e.g. `@aws-cdk/core:newStyleStackSynthesis`). Don't delete flags the CLI added; they pin synthesis behavior. `cdk.context.json` caches environment lookups (AZs, AMIs, VPCs) — commit it so synth is deterministic across machines.

### Cross-stack references — and when to avoid them

Passing a construct from stack A into stack B's props auto-creates a CloudFormation `Export`/`Fn::ImportValue`. This is fine within one app/account/region, but **exports become deploy-order locks**: you can't change or delete an exported value while another stack imports it. Anti-pattern: `Fn.importValue` sprawl across many stacks.

- Within one app: pass the construct object directly; let CDK manage the export.
- Avoid deep cross-stack chains. If two pieces of infra are that coupled, they may belong in one stack.
- For genuinely decoupled stacks/accounts, publish a stable contract (SSM Parameter / Secrets Manager) and read it by name, rather than CFN exports.
- **Nested stacks** (`NestedStack`) help only with the 500-resource hard limit; they share a lifecycle with the parent. Don't use them for organization — use multiple top-level stacks.

## Core L2 idioms

Prefer `.grantX()` to hand-written IAM. Grants compute the minimal action set AND wire the resource policy/KMS key for you.

### Lambda + DynamoDB + grant (the canonical stack)

```ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Runtime } from 'aws-cdk-lib/aws-lambda';

export class OrdersStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const table = new dynamodb.TableV2(this, 'Orders', {
      partitionKey: { name: 'pk', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
      billing: dynamodb.Billing.onDemand(),
      removalPolicy: cdk.RemovalPolicy.RETAIN, // never auto-delete a data store
      pointInTimeRecoverySpecification: { pointInTimeRecoveryEnabled: true },
    });

    const handler = new NodejsFunction(this, 'CreateOrder', {
      entry: 'src/handlers/create-order.ts', // esbuild bundles TS -> JS automatically
      runtime: Runtime.NODEJS_22_X,
      handler: 'handler',
      memorySize: 256,
      timeout: cdk.Duration.seconds(10),
      environment: { TABLE_NAME: table.tableName },
      bundling: { minify: true, sourceMap: true, target: 'node22' },
    });

    table.grantReadWriteData(handler); // emits exactly the needed dynamodb:* actions
  }
}
```

`NodejsFunction` (from `aws-lambda-nodejs`) bundles with esbuild — no manual zip, no `lambda.Code.fromAsset` on raw JS. Reference the TS entry file; CDK transpiles and tree-shakes at synth.

### API exposure

- **Lambda Function URL** — simplest HTTPS endpoint, no API Gateway cost: `handler.addFunctionUrl({ authType: lambda.FunctionUrlAuthType.AWS_IAM })`. Use for internal/simple cases.
- **HTTP API** (`aws-apigatewayv2` + `aws-apigatewayv2-integrations`) — cheaper/faster than REST API; default to it for new HTTP services.
- **REST API** (`apigateway.RestApi` / `LambdaRestApi`) — when you need request validation, usage plans, API keys, WAF tie-in, or fine-grained method config.

### Messaging, events, scheduling

```ts
const queue = new sqs.Queue(this, 'Jobs', {
  visibilityTimeout: cdk.Duration.seconds(60), // >= 6x lambda timeout for SQS-triggered fns
  deadLetterQueue: { queue: dlq, maxReceiveCount: 3 }, // ALWAYS set a DLQ
});
queue.grantConsumeMessages(worker);
worker.addEventSource(new SqsEventSource(queue, { batchSize: 10 }));

topic.addSubscription(new subs.SqsSubscription(queue)); // SNS fan-out -> SQS

// EventBridge rule (event-pattern driven):
new events.Rule(this, 'OnOrder', {
  eventPattern: { source: ['orders'], detailType: ['OrderCreated'] },
  targets: [new targets.LambdaFunction(handler)],
});

// EventBridge Scheduler (modern cron/rate; prefer over Rule schedules for new work):
new scheduler.Schedule(this, 'Nightly', {
  schedule: scheduler.ScheduleExpression.cron({ minute: '0', hour: '3' }),
  target: new schedulerTargets.LambdaInvoke(handler, {}),
});
```

### Step Functions

Use `aws-stepfunctions` + `aws-stepfunctions-tasks`. Compose with `.next()` / `Choice` / `Parallel`; tasks carry their own IAM.

```ts
const submit = new tasks.LambdaInvoke(this, 'Submit', { lambdaFunction: submitFn });
const wait = new sfn.Wait(this, 'Wait', { time: sfn.WaitTime.duration(cdk.Duration.minutes(5)) });
const definition = submit.next(wait).next(new tasks.LambdaInvoke(this, 'Check', { lambdaFunction: checkFn }));
new sfn.StateMachine(this, 'Pipeline', {
  definitionBody: sfn.DefinitionBody.fromChainable(definition),
  stateMachineType: sfn.StateMachineType.STANDARD,
});
```

### Fargate behind an ALB (L3 pattern)

```ts
import { ApplicationLoadBalancedFargateService } from 'aws-cdk-lib/aws-ecs-patterns';

const vpc = new ec2.Vpc(this, 'Vpc', { maxAzs: 2, natGateways: 1 });
const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

const svc = new ApplicationLoadBalancedFargateService(this, 'Web', {
  cluster,
  cpu: 512,
  memoryLimitMiB: 1024,
  desiredCount: 2,
  taskImageOptions: {
    image: ecs.ContainerImage.fromAsset('./service'), // builds + pushes to ECR
    containerPort: 8080,
    environment: { NODE_ENV: 'production' },
  },
  publicLoadBalancer: true,
});
svc.targetGroup.configureHealthCheck({ path: '/health' });
svc.service.autoScaleTaskCount({ maxCapacity: 10 })
  .scaleOnCpuUtilization('Cpu', { targetUtilizationPercent: 60 });
```

