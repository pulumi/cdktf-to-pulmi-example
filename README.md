# CDKTF to Pulumi migration example

An example that shows how to convert a CDK for Terraform (CDKTF) project to [Pulumi](https://pulumi.com), including importing existing cloud resources to bring them under management with Pulumi.

## Overview

This repository contains a CDKTF project written in TypeScript that contains two stacks, `dev` and `prod`, that provision a VM and associated networking — VPC, subnet, security groups, route table, and internet gateway — on Amazon EC2. The walkthrough below guides you through the process of:

1.  Initially deploying the project with CDKTF
1.  Converting the project into a new Pulumi TypeScript project
1.  Importing the deployed cloud resources to start managing them with Pulumi
1.  Destroying the Pulumi stack and all of its resources (to clean up)

## Prerequisites

To complete the walkthrough, you'll need:

-   A current version of Node.js
-   [CDKTF](https://developer.hashicorp.com/terraform/cdktf) and [Terraform](https://www.terraform.io/)
-   [Pulumi](https://www.pulumi.com/docs/install/) and either a free [Pulumi Cloud account](https://app.pulumi.com/signup) or a configured [DIY backend](https://www.pulumi.com/docs/iac/concepts/state-and-backends)
-   Your AWS credentials configured

## Step 1: Deploying the project with CDKTF

Start by deploying the project's `dev` stack to AWS with CDKTF:

```bash
cdktf deploy dev
```

Once the stack is deployed, you'll see 7 new AWS resources were created, along with 3 new Terraform outputs:

```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:
hostname = "ec2-44-242-147-13.us-west-2.compute.amazonaws.com"
ip = "44.242.147.13"
url = "http://ec2-44-242-147-13.us-west-2.compute.amazonaws.com"
```

In a moment, you'll also see that the virtual machine is up and running:

```bash
curl http://ec2-44-242-147-13.us-west-2.compute.amazonaws.com
Hello, world!
```

Make sure your Terraform state file was also created at `./terraform.dev.tfstate`. You'll use this file later to bring these new resources under management with Pulumi.

## Step 2: Converting the project to Pulumi

Next, you'll convert the CDKTF project into a new Pulumi TypeScript project. The process of conversion happens in two steps: first, exporting the project to Hashicorp Configuration Language (HCL), and second, translating the HCL into a new Pulumi program with [`pulumi convert`](https://www.pulumi.com/docs/iac/cli/commands/pulumi_convert/).

### Exporting the project to HCL

Use the CDKTF `synth` command to export the project to HCL. This produces a flat list of all providers, input variables, resources declarations, and Terraform outputs in HCL format, which Pulumi's conversion tooling natively understands:

```bash
cdktf synth --hcl
```

The generated HCL for the `dev` stack should now be at `./cdktf.out/stacks/dev/cdk.tf`.

### Translating the HCL into TypeScript

Make a folder for the new Pulumi project and copy your state and HCL files into it:

```bash
mkdir my-project && cd my-project
cp ../terraform.dev.tfstate .
cp ../cdktf.out/stacks/dev/cdk.tf .
```

In the new project folder, run `pulumi convert` to create a new TypeScript project from the generated HCL:

```bash
pulumi convert --from terraform --language typescript
```

Open `index.ts` in the new project and see that all resources have been translated into their Pulumi equivalents.

## Step 3: Importing existing cloud resources

At this point, the new Pulumi [project](https://www.pulumi.com/docs/iac/concepts/projects/) has been created, but it doesn't yet contain any [stacks](https://www.pulumi.com/docs/iac/concepts/stacks/). In this step, you'll create a new stack to bring the resources you deployed in Step 1 under management with Pulumi.

### Creating a new stack

Start by initializing a new, empty Pulumi stack:

```bash
pulumi stack init dev
```

### Importing resources

Next, use [`pulumi import`](https://www.pulumi.com/docs/iac/cli/commands/pulumi_import/) to bring the resources from your Terraform state file into the new stack:

```bash
pulumi import --from terraform --out ./imported.ts ./terraform.dev.tfstate
```

Notice Pulumi emits a preview of the resources to be imported:

```
Previewing import (dev)

     Type                              Name                   Plan
 +   pulumi:pulumi:Stack               my-project-dev         create
 =   ├─ aws:ec2:Vpc                    vpc                    import
 =   ├─ aws:ec2:Subnet                 subnet                 import
 =   ├─ aws:ec2:SecurityGroup          secGroup               import
 =   ├─ aws:ec2:RouteTableAssociation  routeTableAssociation  import
 =   ├─ aws:ec2:RouteTable             routeTable             import
 =   ├─ aws:ec2:InternetGateway        gateway                import
 =   └─ aws:ec2:Instance               server                 import

Resources:
    + 1 to create
    = 7 to import
    8 changes
```

When prompted, choose `yes` to complete the import process.

### Updating the code

At this point, all resources have been imported, but their definitions in `index.ts` need to be updated to match their current deployment state. Imported resources are also [marked protected](https://www.pulumi.com/docs/iac/concepts/resources/options/protect/) to prevent them being deleted or replaced accidentally, so the code needs to be updated to reflect this as well.

Open `imported.ts` (which was just generated by `pulumi import`) and review the generated code for each resource, taking note of any relevant differences. In this example, all that's needed is to remove any `null` property settings (as the `null` values translated from HCL aren't valid for these resources) and to add the [`protect`](https://www.pulumi.com/docs/iac/concepts/resources/options/protect/) resource option to each one.

For example, the `RouteTable` resource in `index.ts` should go from this:

```typescript
const routeTable = new aws.ec2.RouteTable("routeTable", {
    routes: [
        {
            carrierGatewayId: null,
            cidrBlock: "0.0.0.0/0",
            coreNetworkArn: null,
            destinationPrefixListId: null,
            egressOnlyGatewayId: null,
            gatewayId: gateway.id,
            ipv6CidrBlock: null,
            localGatewayId: null,
            natGatewayId: null,
            networkInterfaceId: null,
            transitGatewayId: null,
            vpcEndpointId: null,
            vpcPeeringConnectionId: null,
        },
    ],
    vpcId: vpc.id,
});
```

To this:

```typescript
const routeTable = new aws.ec2.RouteTable(
    "routeTable",
    {
        routes: [
            {
                cidrBlock: "0.0.0.0/0",
                gatewayId: gateway.id,
            },
        ],
        vpcId: vpc.id, // <- All `null` settings removed
    },
    { protect: true } // <- The `protect` resource option added
);
```

The complete list of resources that need updating is:

-   `aws.ec2.Vpc`
-   `aws.ec2.InternetGateway`
-   `aws.ec2.Subnet`
-   `aws.ec2.RouteTable`
-   `aws.ec2.RouteTableAssociation`
-   `aws.ec2.SecurityGroup`
-   `aws.ec2.Instance`

Once this is done, you can safely delete `imported.ts`:

```bash
rm imported.ts
```

### Running an initial deployment

The last step is to run an initial deployment to bring the new code and stack state into alignment with one another.

Do that by running `pulumi up`:

```bash
pulumi up
```

Again, notice Pulumi renders a preview of the changes to be made — which in this case should be minimal, since your resources have all been imported, and their settings have been updated accordingly.

You should see that all 8 resources are unchanged — the original 7, plus the stack itself — and that the stack now defines three Pulumi outputs whose values match those of the original CDKTF-deployed resources:

```
Previewing update (dev)

     Type                 Name            Plan
     pulumi:pulumi:Stack  my-project-dev

Outputs:
  + hostname: "ec2-44-242-147-13.us-west-2.compute.amazonaws.com"
  + ip      : "44.242.147.13"
  + url     : "http://ec2-44-242-147-13.us-west-2.compute.amazonaws.com"

Resources:
    8 unchanged
```

> #### Seeing replacements?
>
> If it looks like the `Instance` resource is being replaced, try comparing the `user_data` value in `cdk.tf` to `userData` in `index.ts`. These two values should be the same, but some versions of `cdktf synth` add a line break to the end of the generated HCL, which may trigger a replacement). Removing this extra character, or overwriting the `userData` property in `index.ts` with code from your original `../main.ts`, should fix this:
>
> ```typescript
> userData: [
>     "#!/bin/bash",
>     "echo 'Hello, world!' > index.html",
>     "nohup python -m SimpleHTTPServer 80 &"
> ].join("\n"),
> ```

Choose `yes` to complete the deployment:

```
Updating (dev)

     Type                 Name            Status
     pulumi:pulumi:Stack  my-project-dev

Outputs:
  + hostname: "ec2-44-242-147-13.us-west-2.compute.amazonaws.com"
  + ip      : "44.242.147.13"
  + url     : "http://ec2-44-242-147-13.us-west-2.compute.amazonaws.com"

Resources:
    8 unchanged
```

That's it! You've successfully migrated your CDKTF project to Pulumi. 🎉

## Step 4: Cleaning up

Before you can tear down all of the AWS resources you just created, you'll need to remove the deletion protection mentioned above. To do this, you can either delete all of the `protect` resource options you added in the previous step or just change their values from `true` to `false`:

```diff
- { protect: true }
+ { protect: false }
```

Then, run another update to apply the removal of that option:

```bash
pulumi up
```

In the preview, you should see that the `protect` option is changing:

```
Previewing update (dev)

     Type                              Name                   Plan     Info
     pulumi:pulumi:Stack               my-project-dev
     ├─ aws:ec2:Vpc                    vpc                             [diff: ~protect]
     ├─ aws:ec2:Subnet                 subnet                          [diff: ~protect]
     ├─ aws:ec2:InternetGateway        gateway                         [diff: ~protect]
     ├─ aws:ec2:SecurityGroup          secGroup                        [diff: ~protect]
     ├─ aws:ec2:RouteTableAssociation  routeTableAssociation           [diff: ~protect]
     ├─ aws:ec2:RouteTable             routeTable                      [diff: ~protect]
     └─ aws:ec2:Instance               server                          [diff: ~protect]
```

Choose `yes` to complete the update.

At this point, you can destroy the `dev` stack and all of its resources with `pulumi destroy`:

```bash
pulumi destroy
```

Pulumi prompts you to approve the destroy action:

```
Previewing destroy (dev)

     Type                              Name                   Plan
 -   pulumi:pulumi:Stack               my-project-dev         delete
 -   ├─ aws:ec2:Instance               server                 delete
 -   ├─ aws:ec2:Vpc                    vpc                    delete
 -   ├─ aws:ec2:RouteTableAssociation  routeTableAssociation  delete
 -   ├─ aws:ec2:RouteTable             routeTable             delete
 -   ├─ aws:ec2:InternetGateway        gateway                delete
 -   ├─ aws:ec2:Subnet                 subnet                 delete
 -   └─ aws:ec2:SecurityGroup          secGroup               delete

Outputs:
  - hostname: "ec2-44-242-147-13.us-west-2.compute.amazonaws.com"
  - ip      : "44.242.147.13"
  - url     : "http://ec2-44-242-147-13.us-west-2.compute.amazonaws.com"

Resources:
    - 8 to delete
```

Choose `yes` to complete the destroy operation:

```
Resources:
    - 8 deleted
```

To delete the Pulumi stack itself (which is now empty) along with all of its history, run:

```bash
pulumi stack rm dev
```

## Wrapping up, and next steps

We hope this walkthrough has given you an idea of what it's like to migrate a project from CDKTF to Pulumi. To learn more about how Pulumi works, how it differs from from CDKTF (and from Terraform), how to handle additional conversion scenarios, and more, we recommend reading:

-   [The Pulumi docs](https://www.pulumi.com/docs/), to get a better understanding of Pulumi IaC and the broader Pulumi platform
-   [How Pulumi Works](https://www.pulumi.com/docs/iac/concepts/) to dig into core concepts like projects, stacks, configuration, secrets, and the Pulumi resource model
-   [Migrating to Pulumi from Terraform](https://www.pulumi.com/docs/iac/guides/migration/migrating-to-pulumi/from-terraform/) for more detailed, Terraform-specific migration guidance

We'd also encourage you to join us in [the Pulumi Community Slack](https://slack.pulumi.com) to ask questions and learn from those who've successfully made the leap from CDKTF and Terraform to Pulumi.

Welcome to the Pulumi community! 💜
