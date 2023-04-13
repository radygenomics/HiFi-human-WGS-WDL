# Configuring the Amazon Genomics CLI

The Amazon Genomics CLI (`agc`) allows users to orchestrate workflow execution using AWS Batch. See the [getting started docs](https://aws.github.io/amazon-genomics-cli/docs/) for information on installing and using the `agc` to configure and run workflows.

## Deploying a context with `agc`

Once you have installed and authenticated with the `agc`, you can deploy a context using an agc project YAML file.

An [example file](agc-project.yaml) that has the workflow, reference data source, and both on-demand and spot contexts configured using Cromwell as the engine is provided here. This will create an agc project named `agc-humanwgs`, with contexts `spotContext` and `onDemandContext`. If you only require one context (on-demand or spot), delete the portion of the agc-project.yaml file that corresponds to the context you do not wish to deploy.

Note that deploying these resources **will incur costs** even if you are not actively running workflows; ensure that [contexts that are not in use are destroyed](https://aws.github.io/amazon-genomics-cli/docs/reference/agc_context_destroy/) to avoid incurring ongoing costs.

To deploy the context, from the directory containing the `agc-project.yaml` file, run:

```bash
agc context deploy --all
```

# Checking and requesting quota in AWS

See [resources requirements](../../README.md#resource-requirements) for information on the minimum requirements for running the workflow. Typically in a new AWS environment, additional vCPU quota will be required.

## Checking current quota

1. Navigate to [the AWS console](https://console.aws.amazon.com/).
2. In the top right corner, select the region where your `agc` deployment is located.
3. Navigate to EC2.
4. In the menu on the left, select 'Limits'.
5. Filter the limits by searching for "Standard". The current limit field indicates the number of vCPUs that you currently have access to.
- Spot instance limit: `All Standard (A, C, D, H, I, M, R, T, Z) Spot Instance Requests`
- On-demand instance limit: `Running On-Demand All Standard (A, C, D, H, I, M, R, T, Z) instances`

If the number of vCPUs in the context you plan to run the workflow in is less than the limites specified in [the resources requirements](../../README.md#resource-requirements) section, you will need to request additional quota before you can run the workflow.

## Requesting additional quota

5. Continuing from the steps outlined in [checking the current quota](#checking-current-quota), select the service you want to request an increase for.
6. In the top right corner, select 'Request limit increase'.
7. Fill out the appropriate fields in the request form, ensuring that the region you select is the region where you have deployed your `agc` and where your data is located. 256 vCPUs are recommended for running trio data.

Low quota increase requests are typically fulfilled within a 1-2 hours.

# Configuring and running the workflow

## Filling out workflow inputs

Fill out any information missing in [the inputs file](inputs.aws.json). Ensure that all data files used by the workflow are at locations that have been configured in the agc-project.yaml file; see the [granting access to other data files](#granting-access-to-other-data-files) for more information.

See [the inputs section of the main README](../../README.md#workflow-inputs) for more information on the structure of the inputs.json file.

### Determining available zones

To determine available zones in AWS, look for the ZoneName attributes output by the following command:

```bash
aws ec2 describe-availability-zones --region <region>
```

For example, the zones in region us-east-2 are `"us-east-2a us-east-2b us-east-2c"`.

### Determining the AWS batch queue ARN

**Note that if you are using a `miniwdl` engine, you can skip these steps; workflows run via miniwdl will run exclusively in the job queue to which they are submitted.**

1. Visit [the AWS console](https://console.aws.amazon.com/).
2. Navigate to the Batch service.
3. In the lefthand sidebar, select "Compute environments". Note the name of the compute environment with the provisioning model SPOT (if you have deployed a context using spot instances) and the name of the compute environment with provisioning model "EC2" (if you have deployed a context that does not use spot instances).
4. In the lefthand sidebar, select "Job queues".
5. Clicking into an individual queue will show information about the compute environment ("Compute environment order"). Identify the job queue with the Compute environment name that matches the name you identified for the SPOT compute environment; copy the Amazon Resource Name (ARN) for this job queue. This is the value that should be used for the `aws_spot_queue_arn`. Repeat this process to find the ARN for the `aws_on_demand_queue_arn`.

- If `preemptible = true`, only the `aws_spot_queue_arn` is required.
- If `preemptible = false`, only the `aws_on_demand_queue_arn` is required.

## Running the workflow

From the directory where your `agc-project.yaml` is located, run:

`agc workflow run humanwgs --context <context> --inputsFile <input_file_path.json>`

The running workflow can be monitored via [`agc workflow` commands](https://aws.github.io/amazon-genomics-cli/docs/reference/agc_workflow/), or via the AWS console.

# Reference data hosted in AWS

AWS reference data is hosted in the `us-west-2` region  in the bucket `s3://dnastack-resources`.

To use AWS reference data, add the following line to the data section of your [`agc-project.yaml`](https://aws.github.io/amazon-genomics-cli/docs/concepts/projects/):

```yaml
data:
  - location: s3://dnastack-resources
    readOnly: true
```

The [AWS input file template](inputs.aws.json) has paths to the reference files in s3 prefilled. The template [agc-project.yaml](agc-project.yaml) file has this section filled out already; you will not need to edit it further if you are deploying a context using this file.

### Granting access to other data files

S3 buckets outside of the reference files can be accessed by adding additional data blocks to the agc-project.yaml file. See the [agc documentation](https://aws.github.io/amazon-genomics-cli/docs/concepts/data/) for more details on adding additional data sources. All inputs referenced in the inputs.json file will need to be at locations that have been configured in the agc-project.yaml.