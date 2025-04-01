---
layout: post
title: "Enabling Developer Autonomy through IaC and Policy-as-Code"
date: 2025-04-01 00:00:00 +0100
tags: opa opentofu automation devops cdk
category: software
---

Are you a platform engineer feeling overwhelmed by the amount of support you need to provide DevOps teams? Or are you part of a DevOps team, frustrated at the complexities and slow iteration loops you're seeing with infrastructure-as-code and configuration compliance? By adopting a policy of making infrastructure more similar to what developers are used to instead of forcing developers to get used to infrastructure and by shifting configuration checks earlier into the feedback loop, we've been able to alleviate some of these pains.

[1. Introduction](#who-are-you-anyways)

[2. Making Infrastructure-as-Code Easier with CDKTF](#making-infrastructure-as-code-easier-with-cdktf)

[3. Making Infrastructure-as-Code Better with OpenPolicyAgent (OPA)](#making-infrastructure-as-code-better-with-openpolicyagent-opa)

[4. Conclusion](#conclusion)

[5. Resources](#resources)

In this post, I'm going to be detailing how we use [CDK for Terraform](https://github.com/hashicorp/terraform-cdk) and TypeScript to improve the developer experience of writing infrastructure-as-code, and Open Policy Agent to help developer's avoid common security and best-practice gotchas at [TV4 Play](https://www.tv4play.se/) and [MTV Katsomo](https://www.mtv.fi/). This is a text version of a talk that I'm going to present at OpenTofu day EU 2025. If you prefer to watch the talk, I'll update this post with a link once it's available.

# Who are you, anyways?

Thanks for asking! My name is David Stevens and I'm a platform engineer / cloud-guy working with improving the developer experience of infrastructure-as-code and DevOps work at TV4 in Sweden and MTV3 in Finland.

TV4 and MTV3 are sibling companies operating the largest commercial TV channels in Sweden and Finland, respectively, and have a shared infrastructure platform for their streaming services with millions of active users. I've been working at TV4 in their cloud infrastructure team for roughly three years, and one thing that has been absolutely critical in that team is putting a large amount of effort into improving the developer experience every step of the way from writing infrastructure-as-code to deploying applications to the cloud.

This is a necessity for us as we use OpenTofu (the open-source fork of Terraform) to manage more than 65 000 resources across five different providers, with the majority being in AWS. We have 18 different teams of developers across all fields, from data scientists to frontend developers to platform engineers. All of these roughly 80 developers are responsible for designing and operating their own infrastructure with support and guidance where necessary from my team. A majority of these developers are consultants, meaning that rapid onboarding is crucial. In total, we have more than 1 400 workspaces and more than 300 different projects in our infrastructure platform, with a wide range of different cloud services being used.

_All of this is supported by my team of five._

Those mathematics don't add up if an excessive amount of handholding is required from my cloud infrastructure team, so the only option is to obsess over the developer experience. This means making infrastructure as easy as possible to write and maintain, and to make it as difficult as possible to accidentally introduce production outages through bad configuration. Put effort into making things easier for the developers who are using your platform -- that will make things easier for you and save you time in the long run.

# Making Infrastructure-as-Code Easier with CDKTF

The first and perhaps most important step of being a platform engineer and enabling development teams to operate DevOps is providing them with a way to efficiently manage their infrastructure. For many, this is through infrastructure-as-code (IaC). Oftentimes, this is through Terraform or its open-source fork, OpenTofu. This is what we use at TV4 and MTV. Rather than trying to convince you to start using IaC or writing OpenTofu HCL, this post is going to assume you're already doing that and propose writing _less_ HCL. Why? To make OpenTofu easier.

## What's difficult about OpenTofu anyways?

If you're an experienced user of OpenTofu, Terraform, or other IaC tools it can sometimes be difficult to remember where the learning curve is in the language. Since our goal is that all of our developers should be capable of maintaining their infrastructure, including those whose primary experience is within fields that are far from infrastructure such as frontend web development or data science, it's important for us to continuously remind ourselves what's difficult about infrastructure-as-code. Here are a few common pain points that I've seen when onboarding developers to IaC:

### Stringly-typed resources

When providers are strongly-typed, writing IaC can be a breeze. You can easily go through your resources, getting auto-complete suggestions from your editor and friendly error messages when you accidentally type `readonlyRootFileSytstem` instead of `readonlyRootFileSystem`. However, in many providers you will eventually run in to a field which is simply typed as a `string` and the examples page in the provider registry shows the reddest of flags: `jsonencode`. This kind of behaviour is often jokingly referred to as "stringly-typed", something I will do here as well to offer a moment of brevity in what is about to become a brief rant.

<details>
    <summary> When I say "brief", it's a three-paragraphs long rant about specifically the AWS resource `aws_ecs_task_definition`, but the complaints apply to other providers as well. Expand this if you're interested. </summary>

<br/>
I'll be the first to admit that this is a bit of a personal vendetta I have, in addition to being something I've heard developers complain about. I also admit that I feel compassion for the provider developers who offload the task of keeping up with ever-changing APIs in the major cloud providers to simple JSON-blobs. However, I will never say that these fields make for a good end-user experience, especially for developers who are new to the resources and lack familiarity with the schema that they are expected to follow on their own.

<br/><br/>

Take my personal arch-nemesis from my time at TV4: <code class="language-plaintext highlighter-rouge">aws_ecs_task_definition</code>. We use AWS ECS as our primary compute platform, so this resource comes up often. ECS is a container orchestration platform that can be thought of as a managed and largely abstracted Kubernetes, trading off the flexibility and cloud-agnosticism of Kubernetes for lower operational overhead, tighter integrations with the AWS ecosystem of services and fewer knobs to tweak. Each application in ECS needs a task definition, which is similar to a pod specification in Kubernetes. Within this task definition exists the offending field: <code class="language-plaintext highlighter-rouge">container_definitions</code>, which is where you specify a wide variety of options from container images to port mappings to computational resource requests. This one field has a complicated enough schema that a TypeScript type definition for it will easily take more than a hundred lines, but in the OpenTofu provider it takes only one: <code class="language-plaintext highlighter-rouge">container_definitions: string</code>.

<br/><br/>

Worse still, the AWS OpenTofu provider doesn't perform any plan-time validation that the object that has been passed is correct.

<br/><br/>

Even worse, <b> neither does the AWS API </b> which happily accepts input with the typo <code class="language-plaintext highlighter-rouge">readonlyRootFileSytstem</code>and silently ignores anything it doesn't recognize, setting the default value <code class="language-plaintext highlighter-rouge">false</code> and leading to both OpenTofu and the developer thinking that everything went great and the apply was successful even though your actual root filesystem is fully writeable. Congratulations, your typo introduced a potential hidden attack vector!

<br/><br/>

If you can't tell, I learned this the hard way after many hours of debugging, going as far as reading the source code of the AWS provider before I eventually spotted the typo.

</details>

<br/>

Stringly-typed providers are probably never going to disappear. There has historically been improvements within this area, such as AWS' data source `aws_iam_policy_document` or Akamai's `akamai_property_rules_builder`, both of which allow you to generate the necessary JSON in a strongly-typed manner. However, these changes come slowly and it can feel difficult to work around the resources where they do not yet exist in plain OpenTofu.

### State and constants sharing

The conventional way to avoid hard-coding references to provisioned resources is using output blocks. The producer of the data creates an output block defining what to share, and the consumer of the data creates a `terraform_remote_state` data source to read the state.

In order to use `terraform_remote_state`, the developer has to configure the `backend` block for the workspace that they are reading from. This creates a hard-coded reference between the two workspaces and oftentimes leads to a lot of code duplication in the case where the same remote state storage is used.

The developer also becomes forced to think about inter-workspace dependency ordering, thinking carefully about the order in which workspaces are to be applied or destroyed. I've had multiple cases where developers have come and asked for help because they destroyed workspaces in the wrong order and the state read suddenly started failing. Oftentimes, the easiest way to fix this is to recreate the workspace that was just destroyed and then redo the operations in the correct order.

Within an organisation, it is useful to be able to share certain constants. Consider CIDR blocks of known trusted networks, organisation account identifiers, trusted DNS zones, and other such information that is generally useful to have but not tied to a specific project. In OpenTofu, there is no obvious way to share such constants across workspaces.

The best constant sharing approaches I've seen are either creating an "empty" workspace which defines all of the constants as outputs to be accessed with `terraform_remopte_state`, or to publish an "empty" OpenTofu module which defines all of the outputs. Both approaches have their benefits and drawbacks, but both bring additional overhead and feel more like workarounds than an intended use-case.

### HCL

Many things have been said about OpenTofu and Terraform's declarative language, HCL. Without going too far in to value judgements, one thing that has been recurring throughout the workplaces I've been at is that HCL can be a bit unintuitive for beginners.

The declarative part isn't bad - the syntax is easier to read and write than JSON or YAML so it already has an advantage over AWS CloudFormation, and it's pretty easy to grasp even if you haven't seen the language before. The tricky part is when you start incorporating HCL meta-arguments like `count`, `dynamic`, `for_each`, and `depends_on`. Things get even trickier when your project starts scaling into needing to manage multiple modules that are being created with a wide range of inputs and inter-module dependencies.

In particular, conditionals in HCL are janky at best.

Consider the case of writing a OpenTofu module which optionally has some resource. The conventional way to do this is to add `count = var.enable_resource ? 1 : 0` to your resources. This quickly becomes a large amount of repetitive boilerplate, and is unintuitive syntax. Personally, I find it a bit confusing to keep track of which resources are created during which conditions when I read the source code of large modules like the [AWS EKS modules](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/modules/karpenter/main.tf). You could argue this is simply a matter of poor code structure, but it remains unintuitive for beginners.

Even more unintuitive is the syntax for conditionally including a resource with a `for_each`:

```hcl
resource "example" "example" {
  for_each = {
    for k, v in var.iterable : k => v
    if var.enable_resource
  }
}
```

None of this is impossible to understand, but it is an additional hurdle to overcome and something that can put people off when they're busy just trying to move their tickets to "_Done_". These hurdles add up on top of everything else they need to learn to get started with IaC, especially when you consider that:

### Infrastructure itself is difficult

This is perhaps the most important and central thing to remember. Many of our developers' first encounter with cloud infrastructure as a whole is through figuring out their infrastructure-as-code setup to solve something that isn't behaving the way they expect. Oftentimes, this will be code that was written by someone else who may or may not still be in that team.

The more someone needs to learn, the more overwhelming it's going to be and the longer onboarding gets.

What if we can make infrastructure-as-code more similar to the developers' way of working instead of making them adapt to the infrastructure way of working?

## CDK for Terraform (and OpenTofu)

AWS saw this problem with their IaC-framework CloudFormation and launched the AWS Cloud Development Kit (CDK) back in 2018. This is essentially a wrapper allowing you to use a conventional programming language which gets compiled to CloudFormation. In 2020, HashiCorp (the makers of Terraform) launched the [CDK for Terraform (CDKTF)](https://developer.hashicorp.com/terraform/cdktf), based on the AWS CDK.

CDKTF allows you to write your Terraform in TypeScript, Python, Java, C#, or Go using auto-generated bindings toward the regular Terraform or OpenTofu providers. The code then gets "compiled" to regular HCL or Terraform JSON. Here's a simple example in TypeScript, provisioning a Docker container:

```typescript
import { Construct } from "constructs";
import { App, TerraformStack } from "cdktf";
import { DockerProvider } from "@cdktf/provider-docker/lib/provider";
import { Container } from "@cdktf/provider-docker/lib/container";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new DockerProvider(this, "docker", {});
    new Container(this, "nginx", {
      image: "nginx",
      name: "nginx",
      ports: [{ internal: 80, external: 80 }],
    });
  }
}

const app = new App();
new MyStack(app, "minimal");
app.synth();
```

Running `cdktf synth --hcl` results in the following:

```hcl
terraform {
  required_providers {
    docker = {
      version = "3.0.2"
      source  = "kreuzwerker/docker"
    }
  }
  backend "local" {
    path = "/Users/david/tv4/opentofu-day/cdktf/tofu-projects/minimal/terraform.minimal.tfstate"
  }
}

provider "docker" {}

resource "docker_container" "nginx" {
  image = "nginx"
  name  = "nginx"
  ports {
    external = 80
    internal = 80
  }
}
```

## Isn't that just OpenTofu with extra steps?

While it is true that CDKTF is just an abstraction layer on top of OpenTofu and can't actually do anything that can't be done with OpenTofu, there are a lot of benefits that come from using it.

### Syntax

Firstly, and perhaps most significantly, onboarding is easier as developers don't need to learn new syntax. This doesn't mean that they don't need to learn anything new as they still need to learn the semantics of how to work with OpenTofu, but it's one fewer thing to learn. Instead of figuring out how to use `for_each`, `dynamic`, and `count` to control their resources, they can use regular `for`-loops and `if`-statements. This comes much more naturally to someone used to working in imperative languages, and is in my personal opinion easier to follow when you have big conditional blocks than regular HCL `count = 0`-shenanigans even for experienced users.

### Strong typing

Writing your OpenTofu in TypeScript also allows you to create strongly typed wrappers on top of "stringly" typed fields. One very simple utility function that we've found tremendous benefits in is:

```typescript
/**
 * Call JSON.stringify with input data type checking.
 * @param obj The object of type `T` to stringify
 * @returns JSON-serialized string of `obj`
 */
export function stringify<T>(obj: T): string {
  return JSON.stringify(obj);
}
```

This allows you to use a strongly typed alternative to `jsonencode` with a custom type definition, which gets compiled to a string literal in HCL. We have written multiple such type definitions, covering everything from AWS IAM policy types for use in inline policies instead of creating separate `aws_iam_policy_document` data sources to the `aws_ecs_task_definition` `container_definitions` field that I ranted about earlier. This allows you to catch errors in these fields at _compile-time_, before you even run a plan, complete will helpful red squigglies and error messages in your editor. It's hard to overstate how helpful this is.

You could even take this further through subclassing the provider's resources through inheritance. As a simple example, consider the earlier case of `aws_ecs_task_definition` where we wished the `container_definitions` field was typed more strongly than `string`. Through TypeScript, we can do that!

```typescript
import {
  EcsTaskDefinition,
  EcsTaskDefinitionConfig,
} from "@cdktf/provider-aws/lib/ecs-task-definition";
import { Fn } from "cdktf";
import { Construct } from "constructs";

export type TypedEcsTaskDefinitionConfig = Omit<EcsTaskDefinitionConfig, "containerDefinitions"> & {
  containerDefinitions: EcsContainerDefinition[];
};

export class TypedEcsTaskDefinition extends EcsTaskDefinition {
  constructor(scope: Construct, id: string, config: TypedEcsTaskDefinitionConfig) {
    super(scope, id, {
      ...config,
      containerDefinitions: Fn.jsonencode(config.containerDefinitions),
    });
  }
}

export type EcsContainerDefinition = {
  name?: string;
  image?: string;
  command?: string[];
  containerPort?: number;
  cpu?: number;
  ...
}
```

Here, we define a subclass of the AWS provider's resource which has the exact same configuration options as the base resource and functions the exact same way _except_ that the `container_definitions` field is now fully type-checked at compile-time (and in your editor).

All of these type definitions and custom subresources can be exported as part of a regular TypeScript package. You can also use exports to share constants or any other helper functions that you can think of, whether they are TypeScript functions that are applied during compile-time or compositions of base OpenTofu functions that are applied during actual OpenTofu execution.

### Abstract anything

In regular OpenTofu, you can abstract away the intricacies of configuring resources into modules. However, there are certain things like the OpenTofu backend and provider configuration that cannot be abstracted away and must be done on a top-level in your code. You can write scripts to bootstrap this configuration to help developers get started, but it cannot be hidden completely from their view.

In CDKTF, you can use JavaScript packages instead of plain HCL modules and abstract anything.

```typescript
import { RemoteBackend, TerraformStack } from "cdktf";
import { basename } from "path";
import { cwd } from "process";
import { AwsProvider } from "@cdktf/provider-aws/lib/provider";
import { Construct } from "constructs";

export enum Team {
  CLOUD_INFRA = "cloud-infra",
}

export enum AwsAccount {
  Internal = "01234567890",
  Dev = "123456789012",
  Prod = "234567890123",
}

export class MyTofuStack extends TerraformStack {
  protected providers: { aws: Record<string, AwsProvider> };
  constructor(scope: Construct, id: string, options: { awsAccounts: AwsAccount[]; team: Team }) {
    super(scope, id);
    new RemoteBackend(this, {
      hostname: process.env.TF_REMOTE_BACKEND_HOSTNAME,
      organization: process.env.TF_REMOTE_BACKEND_ORGANIZATION!,
      workspaces: {
        name: `${basename(cwd())}-${id}`,
      },
    });
    const providers: Record<string, AwsProvider> = {};
    for (const account of options.awsAccounts) {
      providers[account] = new AwsProvider(this, `aws-${account}`, {
        alias: account,
        assumeRole: [{ roleArn: `arn:aws:iam::${account}:role/opentofu` }],
        defaultTags: [{ tags: { Team: options.team } }],
      });
    }
    this.providers = { aws: providers };
  }
}
```

At TV4 and MTV, we automatically configure backend workspace information based on the directory structure your code is in in our shared infrastructure monorepo. This is done by calling arbitrary JavaScript operating system functions like `basename` and `cwd`. You can also freely use environment variables, web requests, or _any other logic you please_ to template your code during the compilation-phase. This is exemplified in the code snippet above where the remote backend is configured using the directory path and environment variables, and multiple AWS providers are configured assuming various roles for a pre-defined set of AWS account IDs. This class is then used as:

```typescript
import { Construct } from "constructs";
import { App } from "cdktf";
import { AwsAccount, MyTofuStack, Team } from "cdktf-extensions";
import { Instance } from "@cdktf/provider-aws/lib/instance";

class MyStack extends MyTofuStack {
  constructor(scope: Construct, id: string) {
    super(scope, id, {
      // We want this stack in the Dev and Prod accounts.
      awsAccounts: [AwsAccount.Dev, AwsAccount.Prod],
      team: Team.CLOUD_INFRA,
    });

    // Note: JavaScript for-loop over providers! No need for for_each
    for (const [accountName, provider] of Object.entries(this.providers.aws)) {
      new Instance(this, `compute-${accountName}`, {
        ami: "ami-01456a894f71116f2",
        instanceType: "t2.micro",
        provider,
      });
    }
  }
}

const app = new App();
// You can optionally add arguments to MyStack and create multiple instances of it, each of which
// will end up in a different OpenTofu workspace.
new MyStack(app, "test");

app.synth();
```

All told, TypeScript turns into the most feature-rich OpenTofu templating engine you can imagine.

This goes even further if you start using JavaScript classes instead of regular HCL modules for your code reuse. If you do this, you can suddenly nest modules and share logic not only through composition as in regular HCL, but through object-oriented concepts like inheritance. We use inheritance extensively, with examples including a VPC-local AWS Lambda function being a subclass of a regular Lambda function or a AWS CloudFront CDN with a VPC origin being a subclass of a CloudFront with a generic custom origin. Subclassing in this way allows you to keep the code much tidier than it would be through composition or `count = var.enable_vpc ? 1 : 0` blocks throughout your module, and allows you to expose more narrow interfaces toward the end-user of the module than needing to expose all possible input variables even though some might be specific to VPC-local Lambdas.

This is a win not only for the users of the modules who get cleaner interfaces to keep track of but also for the maintainers of the modules who get more flexibility in how they structure their resources.

## Cross-workspace state sharing

CDKTF also makes state sharing much easier than explicitly working with `data_terraform_remote_state` and outputs by introducing stacks.

Consider the following trivial pub/sub use-case where a `producer` stack creates a AWS SNS topic and a `consumer` stack needs the ID of that topic to subscribe to it. In CDKTF, sharing state is as simple as passing object attributes. The producer creates a SNS topic and exposes it as an attribute, which is passed to the constructor of the consumer.

```typescript
// producer.ts
import { Construct } from "constructs";
import { SnsTopic } from "@cdktf/provider-aws/lib/sns-topic";
import { AwsAccount, MyTofuStack, Team } from "cdktf-extensions";

export class Producer extends MyTofuStack {
  public topic: SnsTopic;

  constructor(scope: Construct, id: string) {
    super(scope, id, {
      awsAccounts: [AwsAccount.Prod],
      team: Team.CLOUD_INFRA,
    });
    this.topic = new SnsTopic(this, "topic", {});
  }
}
```

```typescript
// consumer.ts
import { Construct } from "constructs";
import { SqsQueue } from "@cdktf/provider-aws/lib/sqs-queue";
import { SnsTopicSubscription } from "@cdktf/provider-aws/lib/sns-topic-subscription";
import { AwsAccount, MyTofuStack, Team } from "cdktf-extensions";

export type ConsumerOptions = {
  topicArn: string;
};

export class Consumer extends MyTofuStack {
  constructor(scope: Construct, id: string, options: ConsumerOptions) {
    super(scope, id, {
      awsAccounts: [AwsAccount.Prod],
      team: Team.CLOUD_INFRA,
    });

    const queue = new SqsQueue(this, "queue", {});
    new SnsTopicSubscription(this, "subscription", {
      topicArn: options.topicArn,
      protocol: "sqs",
      endpoint: queue.arn,
    });
  }
}
```

The two are connected with a simple `main.ts`:

```typescript
// main.ts
import { App } from "cdktf";
import { Producer } from "./producer";
import { Consumer } from "./consumer";

const app = new App();
const producer = new Producer(app, "producer");
new Consumer(app, "consumer", { topicArn: producer.topic.arn });
app.synth();
```

When you compile this with `cdktf synth --hcl`, you get a directory structure like:

```
.
├── main.ts
├── consumer.ts
├── producer.ts
└── cdktf.out
     ├── stacks
     │   ├── producer
     │   │   ├── metadata.json
     │   │   └── cdk.tf
     │   └── consumer
     │       ├── metadata.json
     │       └── cdk.tf
     └── manifest.json
```

where `producer/cdk.tf` automatically contains an output for the `.arn` attribute of the topic since that is what was used in the consumer:

```hcl
// producer/cdk.tf
terraform {
  # required_providers omitted for brevity
  backend "remote" {
    workspaces = {
      name = "multistack-producer"
    }
  }
}

provider "aws" {/* configuration omitted for brevity */}

resource "aws_sns_topic" "topic" {}

output "cross-stack-output-aws_sns_topictopicarn" {
  value     = "${aws_sns_topic.topic.arn}"
  sensitive = true
}
```

and `consumer/cdk.tf` automatically contains a `data_terraform_remote_state` pointing to the producer's workspace:

```hcl
terraform {
  # required_providers omitted for brevity
  backend "remote" {
    workspaces = {
      name = "multistack-consumer"
    }
  }
}

provider "aws" {/* configuration omitted for brevity */}

resource "aws_sqs_queue" "queue" {}

resource "aws_sns_topic_subscription" "subscription" {
  endpoint  = "${aws_sqs_queue.queue.arn}"
  protocol  = "sqs"
  topic_arn = "${data.terraform_remote_state.cross-stack-reference-input-producer.outputs.cross-stack-output-aws_sns_topictopicarn}"
}

data "terraform_remote_state" "cross-stack-reference-input-producer" {
  backend = "remote"
  config = {
    workspaces = {
      name = "multistack-producer"
    }
  }
}
```

State sharing in this manner becomes effortless. Of course, this requires the stacks to be configured within the same `main.ts`, so explicitly defining outputs can still be useful for state sharing between less closely linked stacks.

### Doing things in the right order with the CDKTF CLI

If you take things one step further and use the `cdktf` CLI instead of `tofu` to run your deployments, CDKTF will automatically identify inter-stack dependencies and build a graph over the cross-stack execution order. Running `cdktf deploy consumer producer` will automatically deploy the `producer` stack first, followed by the `consumer` stack.

Running `cdktf deploy consumer` without including the `producer` stack leads to error

> Error: Usage Error: The following dependencies are not included in the stacks to run: producer. Either add them or add the --ignore-missing-stack-dependencies flag.

The same applies to destroys, which makes unnestling cross-stack dependencies much easier. Similar functionality has been added to HashiCorp Terraform, but it does not yet exist in OpenTofu.

### CDKTF: The Good, the Bad, and the ugly-but-I-promise-it-doesn't-happen-often

In summary, we are very happy with our choice to go all-in on CDKTF. The platform team finds it easier to write and maintain our module libraries, onboarding developers is easier, and we are able to work around flaws that we find in the providers that we use. I've heard from multiple of the developers that have left TV4 that CDKTF is one of the things they miss the most about our infrastructure platform, and it has been very well received overall.

Oon a personal note, over the past couple of years of development our CDKTF library has become incredibly productive, and whenever I go back to writing plain OpenTofu HCL I find myself missing TypeScript. This is not something I had expected to feel before I started this journey.

However, CDKTF isn't perfect. Future support for the project is unclear at the moment, with the latest major update coming from HashiCorp in January 2024. While it is somewhat feature-complete from our perspective, a lack of development is concerning, especially when paired with the CLI sometimes being a bit tricky to configure for use with OpenTofu instead of Terraform. Disk usage can also spiral out-of-hand if you aren't careful, as the type definitions for large providers like the AWS provider can become very large. For the AWS provider, the bindings are currently 352 MB, which is in addition to the disk space that the actual provider takes. Across many projects, this adds up.

Furthermore, while the abstractions of TypeScript are tremendously useful, the code gets plain ugly when the abstractions break. The most common case this happens is when you need to iterate over dynamic attributes of a resource. You can no-longer use a JavaScript for-loop for this, and instead have to use a workaround called [iterators](https://developer.hashicorp.com/terraform/cdktf/concepts/iterators). Some times this is strictly necessary, such as when creating a DNS-validated TLS certificate in AWS, but each time I write such code I find myself wishing there was a better way.

### Advice

First and foremost, give CDKTF a try! I think you might find that you enjoy it. Beyond that, I have some advice that has helped us make a more productive infrastructure platform which hopefully can be useful to you too if you find yourself in a platform engineering role.

The most important thing is the same advice I'd give anyone building anything: dog-food your platform. Write your own infrastructure using the same CDKTF library that you build for the developers to write theirs. This will rapidly help you identify room for improvement, and helps you feel comfortable in providing support when it is requested.

It's also very helpful to build your libraries with multiple levels of abstractions. The AWS CDK comes with a whole library of such abstractions that can be used as inspiration, and groups abstractions into three levels:

1. Level 1 (L1) abstractions are raw resources. These are automatically exported by CDKTF.
2. Level 2 (L2) abstractions are small but common compositions of resources, like bundling a Docker image repository with some permission policies to be able to access it. These become the bulk of what you'll use to build the next level:
3. Level 3 (L3) abstractions are very high-level use-cases. Consider a class called `ContainerService`, which might set up a Docker repository, GitHub Actions deployment permissions, runtime service configurations, load balancing, a CDN, and DNS records, all bundled into a single module.

Ideally, a developer should be able to get their application up in a minimal production-ready state using only a L3 construct or an L3 construct paired with a couple of L2 constructs. This is crucial, as it allows developers to opt-in to further detail and complexity or remain on a sane default if they do not want to dive deeper.

Finally, make sure that you spend time hanging out with the developers. Not only do they (probably) not bite, small-talk is often the only time you'll hear about the minor pet peeves and gripes that might not be considered significant enough to bring up as a feature request but definitely add up to a more painful experience.

# Making Infrastructure-as-Code Better with OpenPolicyAgent (OPA)

So, you've built an effective infrastructure platform where your developers are happily hacking away creating infrastructure autonomously without you needing to intervene. Except you do need to intervene, because setting up functional infrastructure isn't the same thing as ssetting up good infrastructure.

## What's difficult about configuring infrastructure anyways?

First and foremost, need I remind you that _infrastructure itself is difficult_ and can be overwhelming for beginners? Modern public cloud providers are so flexible and often unopinionated that they present a vast number of different options for each resource and even more ways that different resource types can be chained together. When dealing with raw resources, you slowly learn which half of the available options to ignore entirely as they don't matter to you and which half to set to which specific parameters that should never be configured in any other way. This is oerwhelming, and even if you provide helpful higher-level constructs that abstract away these raw resources most of the time the developers will eventually need to do interact with raw resources and face these complexities head on.

This is especially important as the default values in the cloud providers aren't always the best practice either. Consider AWS ElastiCache clusters, their managed Redis/Valkey service. If you create a `aws_elasticache_replication_group` resource, it will default to using the last open-source version of Redis instead of using the newer Valkey version. Valkey is fully backward compatible, performs better, and is significantly cheaper on AWS. There is absolutely no reason not to run Valkey if you are using ElastiCache and don't need to be on a specific old version, but the provider still defaults you to Redis.

It's also easy to misuse cloud resources. Continuing on the ElastiCache example, it and AWS' managed SQL clusters in RDS use a clustering strategy with a primary node which can process both reads and writes and then a host of secondary nodes which can only process read operations. To scalably use a cluster, you want to maintain separate connections in your application and route all of your writes to the primary node and all of your reads to the secondary nodes. It is incredibly easy to expect AWS to handle this for you and simply use the `.endpoint` attribute of the RDS cluster and not even realise that there is a second attribute called `.reader_endpoint` which you should be using as well. The end result of this is that your application seems like it's working fine, but in reality you don't actually have a database cluster but instead a single instance which is doing all of the work while your replicas are idling.

Finally, there is of course also the policy compliance and security perspective. You may be working in a regulated industry that requires your cloud environment to fulfill certain standards, compliance which can be difficult to show. You may also just want to make sure that you're following security best practices and avoid accidentally misconfiguring something.

Handling any such configuration checks is difficult to do and difficult to scale.

### So how have you been handling it?

At TV4 and MTV3, we have historically set up a variety of reactive checks to identify misconfigurations. These are often in the form of cloud provider tooling like AWS Config which scans our environments periodically and reports any detected errors to the platform team. We then identify and speak to the team that owns the infrastructure in question. Other times, we stumble upon more nuanced misconfigurations when working with the infrastructure platform.

This type of reactive approach isn't scalable. It's also a horrible developer experience, since the feedback that their infrastructure is misconfigured can come days or weeks after they thought they were done with setting up the infrastructure.

### What could we do instead?

What if there was a tool that we could use to run proactive checks instead of reactive ones? Perhaps something that we run on every OpenTofu plan, alerting the developer before the change is even deployed? This would offload the platform team and drastrically speed up the feedback loop for developers.

## OpenPolicyAgent for Policy-as-Code

We found that tool in OpenPolicyAgent (OPA). OPA is a general purpose tool for evaluating policies against more-or-less arbitrary input data. Since OpenTofu plans can easily be exported to JSON through `tofu show -json <plan-file>`, we can natively feed that JSON plan to OPA and start running policies against it.

Consider a security example: in AWS ECS Fargate, which is their serverless container runtime, you can either set a runtime version to `LATEST` or a specific version. It's a best practice to always set this to `LATEST` so that patches are applied automatically by AWS.

```hcl
resource "aws_ecs_service" "invalid" {
  launch_type = "FARGATE"
  platform_version = "1.4.0"
  # ...
}

resource "aws_ecs_service" "valid" {
  launch_type = "FARGATE"
  platform_version = "1.4.0"
  # ...
}
```

The JSON plan that is generated by OpenTofu contains two important top-level properties: `planned_values` and `resource_changes`. `planned_values` contains the resource configuration that OpenTofu intends for the resource to end up in, and `resource_changes` contains information about the changes that are going to be applied to get there.

In order to avoid overwhelming users with warnings for infrastructure they're not touching, let's make a policy that only checks resources that are currently being modified. Policies in OPA are defined in a logic programming language called Rego. First, let's make a helper function that fetches all resources of a certain that are being created or updated:

```rego
resources(plan, type) := [
{"address": resource.address, "configuration": resource.change.after} |
	some resource in plan.resource_changes
	resource.type == type
	some action in resource.change.actions
	action in {"create", "update"}
]
```

The beauty of logic programming languages is that they are very declarative. This function creates a array comprehension of form `[{...} | ...]`, where the mapping on the left of the `|` is populated if and only if every condition on the right of the `|` is true.

In this case, we are looking for some resource in the `resource_changes` such that it has the intended type and the action we're trying to do to it is either creating it or updating it. Since we're looping in an array comprehension, these conditions get tested for each and every resource in `resource_changes` and we end up with an array of exactly what we wanted. Neat, right? Next up, we just need to make a policy:

```rego
fargate_uses_old_version(plan) := {deny |
	some service in resources(plan, "aws_ecs_service")
	service.configuration.launch_type == "FARGATE"
	service.configuration.platform_version != "LATEST"
	deny := {
		"reason": "Require LATEST Fargate version",
		"severity": "medium",
		"resource": service.address,
	}
}
```

Here, I've chosen to write the policy logic as a function again. The specifics of how this function gets invoked can differ between your OpenTofu backend, but you shouldn't need much more than a thin wrapper to use the function. Let's break it down.

A OpenTofu plan violates the rule if:

1. For some resource which is an `aws_ecs_service`
2. Which uses Fargate
3. and the platform version isn't `LATEST`

The final statement `deny := {}` simply sets an output value that is used in the outer set comprehension to report all violations and not just the first that is found. We also include some metadata about the rule that can be displayed in error messages.

That's all it takes to get started with OPA!

## Reflections on OPA

We've gradually been introducing more and more policies over the past few months and it has been a success. The developers genuinely appreciate being able to get immediate feedback on their plans and have rapidly taken to fixing the warnings even though OPA violations are currently just set to warn the user rather than block the apply. As a result of OPA, we've had developers ask us how to modify their Docker containers so that they can be run as a non-root user and other such security best practices that we really weren't expecting people to actively start caring about. This is a great win for the security posture of the company!

It has also been very helpful to use OPA as a sort of unit test preventing recurring misconfigurations. The clearest example of this is the ElastiCache and RDS read replica usage issue that I mentioned earlier: it didn't take very many lines of code to write a policy that checked that every OpenTofu plan that uses the primary endpoint attribute also uses the secondary endpoint attribute. The same way a unit test can be added to application code to prevent a bug from reappearing, we can use OPA to prevent infrastructure bugs from reappearing.

The actual policy language, Rego, has also turned out to be very pleasanat and productive to write once you get your head around it. However, some aspects of it are trickier than others.

In particular, Rego and OPA use a somewhat unconventional import and namespacing structure compared to the programming environments we might interact with on a more usual basis. Combined with the lack of a standardised package manager (although there is some progress on that front), getting started with writing reusable policy packs can be a bit tricky.

It also becomes tricky because there are fewer pre-built open-source policy packs for OPA than for other Policy-as-Code options such as Checko at the moment. We're trying to help here, by open-sourcing our own policy pack beginning with ports of AWS policies from the AWS services Control Tower and Config.

# Conclusion

At TV4 and MTV3, our next steps are going to be introducing many more policies to expand our resource coverage. We're also considering inestigating the use of OPA as a pre-commit hook to run policies against OpenTofu code before a plan has been running, tightening the feedback loop further. We're also going to investigate deeper usage of the CDKTF CLI, and the work that would be necessary to get it to natively operate with OpenTofu instead of HashiCorp Terraform.

I hope that this post has inspired you to give CDKTF and OpenTofu a try. Most importantly, I think it's important to remember that as a platform engineer, _making life easier for your platform users also makes life easier for you_. Try to constantly iterate based on their feedback, and always identify what could be done better.

# Resources

- [HashiCorp's CDKTF getting-started guide](https://developer.hashicorp.com/terraform/tutorials/cdktf/cdktf-install)
- [OpenPolicyAgent's guide to checking Terraform plans](https://www.openpolicyagent.org/docs/latest/terraform/)
- [OpenPolicyAgent's language reference](https://www.openpolicyagent.org/docs/latest/policy-language/)
- [My open-source policy pack for AWS and OpenTofu](https://github.com/stevensdavid/opentofu-opa) (which is painfully lacking a README at the moment)
