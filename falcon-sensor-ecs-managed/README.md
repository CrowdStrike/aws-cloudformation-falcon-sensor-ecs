# Falcon Sensor CloudFormation Template for ECS Managed Instances
Automatically secure all ECS Managed instances in your Amazon ECS cluster by deploying the Falcon sensor using CloudFormation templates.

Deploying the Falcon sensor as an [ECS Managed Daemon](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/managed-daemons.html)
through CloudFormation gives you:

- **Complete coverage**: Automatically protects all ECS Managed Instances in your ECS cluster
- **Zero-touch deployment**: New ECS Managed Instances receive the sensor immediately when added
- **Automatic recovery**: By default, if a sensor exits, ECS replaces the instance, and the new instance runs the sensor before other workloads
- **Infrastructure as code**: Deploy consistently across environments with version control

## Table of contents
- [Requirements](#requirements)
  - [Capacity providers](#capacity-providers)
  - [AWS CLI region](#aws-cli-region)
  - [Required IAM Permissions](#required-iam-permissions)
- [CloudFormation template support for Falcon sensor versions](#cloudformation-template-support-for-falcon-sensor-versions)
- [Deploy the Falcon sensor on an ECS cluster with ECS Managed Instances](#deploy-the-falcon-sensor-on-an-ecs-cluster-with-ecs-managed-instances)
  - [About the CloudFormation template](#about-the-cloudformation-template)
  - [Instance replacement](#instance-replacement)
  - [Step 1: Get the Falcon sensor image](#step-1-get-the-falcon-sensor-image)
  - [Step 2: Define your deployment parameters](#step-2-define-your-deployment-parameters)
  - [Step 3: Get the CloudFormation template](#step-3-get-the-cloudformation-template)
  - [Step 4: Deploy with CloudFormation](#step-4-deploy-with-cloudformation)
  - [Step 5: Verify the deployment](#step-5-verify-the-deployment)
    - [Template Configuration Parameters](#template-configuration-parameters)
- [Upgrade the Falcon sensor](#upgrade-the-falcon-sensor)
  - [Step 1: Get the new Falcon sensor image and update your ECR repository](#step-1-get-the-new-falcon-sensor-image-and-update-your-ecr-repository)
  - [Step 2: Deploy the updated stack](#step-2-deploy-the-updated-stack)
  - [Step 3: Verify that the upgrade completed successfully](#step-3-verify-that-the-upgrade-completed-successfully)
- [Uninstall the sensor](#uninstall-the-sensor)
- [Advanced Configuration](#advanced-configuration)
  - [Configure deployment behavior](#configure-deployment-behavior)
  - [Configure daemon criticality](#configure-daemon-criticality)
  - [Enable logging](#enable-logging)
  - [Enable ECS exec](#enable-ecs-exec)
  - [Enable resource limits](#enable-resource-limits)

## Requirements
Verify you meet these requirements:

- AWS CLI is configured with the appropriate permissions
- Git installed for accessing templates
- You have access to the CrowdStrike Falcon container registry
- You have an existing ECS cluster with ECS Managed Instances, or you have the ability to create one
- Your cluster has at least one ECS Managed Instances capacity provider. For info, see [Capacity providers](#capacity-providers).
- For all Falcon sensor for Linux requirements, see [Falcon Sensor for Linux System Requirements](https://falcon.crowdstrike.com/documentation/page/edd7717e/falcon-sensor-for-linux-system-requirements)

### Capacity providers
ECS Managed Daemons are scheduled only on the instances managed by the capacity providers associated with the daemon.
The template requires the `CapacityProviderArns` parameter: a comma separated list of the ECS Managed Instances capacity
provider ARNs that the Falcon sensor should run on. Instances managed by capacity providers that are not in this list are
not protected.

To protect every ECS Managed Instance in your cluster, include all the cluster's ECS Managed Instances capacity
providers. Get their ARNs as a comma separated list:
```bash
  export CAPACITY_PROVIDER_ARNS=$(aws ecs describe-capacity-providers \
    --cluster $ECS_CLUSTER_NAME \
    --query "join(',', capacityProviders[?managedInstancesProvider].capacityProviderArn)" \
    --output text)
```

### AWS CLI region
Ensure your AWS CLI is configured to use the same region as your ECS cluster. CloudFormation stacks are region-specific,
and your deployment will fail if the CLI is configured for a different region than your cluster. Verify the CLI region:

- Check your current region: `aws configure get region`
- Set your region: `aws configure set region <your-region>`
- Alternatively, specify the region in each command: `aws cloudformation deploy --region <your-region> …`

### Required IAM Permissions
An ECS execution role is required for every deployment. On ECS Managed Instances, the instance role does not include
permissions to pull images from ECR or send container logs to CloudWatch, so ECS uses the execution role for both. If you
plan to use the execute-command feature, the additional required permissions are shown in the table.

<table>
  <thead>
    <tr>
      <th>Role</th>
      <th>Required Permissions</th>
      <th>Notes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Deployment User/Role</td>
      <td>
        cloudformation:CreateStack<br>
        cloudformation:DeleteStack<br>
        cloudformation:DescribeStacks<br>
        cloudformation:UpdateStack<br>
        cloudformation:ListStackResources<br>
        cloudformation:DescribeStackResources<br>
        cloudformation:CreateChangeSet<br>
        cloudformation:ExecuteChangeSet<br>
        cloudformation:DeleteChangeSet<br>
        cloudformation:DescribeChangeSet<br>
        <br>
        ecs:DescribeClusters<br>
        ecs:RegisterDaemonTaskDefinition<br>
        ecs:DescribeDaemonTaskDefinition<br>
        ecs:DeleteDaemonTaskDefinition<br>
        ecs:CreateDaemon<br>
        ecs:UpdateDaemon<br>
        ecs:DeleteDaemon<br>
        ecs:DescribeDaemon<br>
        ecs:ListDaemonDeployments<br>
        ecs:DescribeDaemonDeployments<br>
        ecs:TagResource<br>
        ecs:UntagResource<br>
        ecs:ListTagsForResource<br>
        <br>
        iam:PassRole<br>
        <br>
        logs:CreateLogGroup<br>
        logs:DeleteLogGroup
      </td>
      <td style="text-align:left;vertical-align:top">
        Standard ECS CloudFormation deployment permissions<br>
        <br>
        Logs permissions are required when logging is enabled
      </td>
    </tr>
    <tr>
      <td>ECS Execution Role</td>
      <td>AmazonECSTaskExecutionRolePolicy</td>
      <td style="text-align:left;vertical-align:top">Always required. Used to pull the Falcon sensor image from ECR and send container logs to CloudWatch</td>
    </tr>
    <tr>
      <td>ECS Task Role</td>
      <td>
        ssmmessages:CreateControlChannel<br>
        ssmmessages:CreateDataChannel<br>
        ssmmessages:OpenControlChannel<br>
        ssmmessages:OpenDataChannel
      </td>
      <td style="text-align:left;vertical-align:top">Only required when ECS exec is enabled</td>
    </tr>
  </tbody>
</table>

## CloudFormation template support for Falcon sensor versions
<table>
  <thead>
    <tr>
      <th>CloudFormation Template Version</th>
      <th>Falcon Sensor Version</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>>= 0.1.x</td>
      <td>>= 8.12.x</td>
    </tr>
  </tbody>
</table>

## Deploy the Falcon sensor on an ECS cluster with ECS Managed Instances
To complete your Falcon sensor for Linux deployment on an ECS cluster with ECS Managed Instances with AWS CloudFormation, you should complete these steps:

- [Step 1: Get the Falcon sensor image](#step-1-get-the-falcon-sensor-image)
- [Step 2: Define your deployment parameters](#step-2-define-your-deployment-parameters)
- [Step 3: Get the CloudFormation template](#step-3-get-the-cloudformation-template)
- [Step 4: Deploy with CloudFormation](#step-4-deploy-with-cloudformation)
- [Step 5: Verify the deployment](#step-5-verify-the-deployment)

### About the CloudFormation template
The CloudFormation template deploys the Falcon sensor as an ECS daemon on your ECS cluster with
ECS Managed Instances, ensuring complete protection across all ECS Managed Instances.

<table>
  <tr>
    <th>Aspect</th>
    <th>Details</th>
  </tr>
  <tr>
    <td>Architecture</td>
    <td>
      <ul>
        <li>Two-container design: an initialization container and a main sensor container</li>
        <li>ECS daemon scheduling ensures 1 sensor per ECS Managed Instance managed by the capacity providers in CapacityProviderArns</li>
        <li>Automatically protects new instances as they join those capacity providers</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Resources</td>
    <td>
      <ul>
        <li>Memory reservation: 512MiB default (configurable)</li>
        <li>CPU reservation: 256 CPU units default (configurable)</li>
        <li>Storage: Persistent volumes at /opt/CrowdStrike/falconstore and /opt/CrowdStrike/config</li>
        <li>Platform: Requires Linux-based ECS Managed Instances</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Security</td>
    <td>
      <ul>
        <li>Runs with elevated privileges (root user, privileged mode)</li>
        <li>Uses shared IPC and PID modes for complete host visibility</li>
        <li>These permissions are required for proper security monitoring</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Configuration</td>
    <td>
      <ul>
        <li>Basic: CID, image path, cluster name, IAM roles</li>
        <li>Advanced: Proxy settings, logging levels, tagging, cloud region</li>
        <li>Logging: CloudWatch integration available but disabled by default</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Management</td>
    <td>
      <ul>
        <li>Daemon name: crowdstrike-falcon-node-daemon</li>
        <li>New instances from cluster scaling events get the sensor before other workloads</li>
        <li>Instances are replaced, not restarted in place. For info, see <a href="#instance-replacement">Instance replacement</a></li>
        <li>Configurable rolling update behavior via deployment bake time and drain percent</li>
      </ul>
    </td>
  </tr>
</table>

This template follows security best practices while ensuring the Falcon sensor has the access it needs to properly
protect your workloads.

### Instance replacement
ECS Managed Instances doesn't restart or reboot instances in place. ECS drains and replaces an instance when the sensor
daemon is placed, changed, or removed. This happens when you:

- Deploy the sensor: existing instances are replaced in batches, and the sensor starts on each new instance before your workloads
- Upgrade the sensor or redeploy with changed sensor parameters
- Delete the sensor daemon stack

When the sensor daemon is critical (`DaemonCritical=true`, the default), ECS also replaces the instance when you stop
the sensor task, or when the sensor exits.

Each replacement instance registers in the Falcon console as a new host with a new agent ID (AID). The hosts for the
replaced instances go offline. Batch size and pacing follow the `DeploymentDrainPercent` and
`DeploymentBakeTimeInMinutes` parameters. For info, see [Configure deployment behavior](#configure-deployment-behavior)
and [Configure daemon criticality](#configure-daemon-criticality).

Setting `DaemonCritical=false` doesn't prevent instance replacement when you deploy, upgrade, change, or delete the
sensor. It only stops ECS from replacing an instance when the sensor stops, exits, or becomes unhealthy. Consider it only if your
workloads can't tolerate unplanned instance drains and workload availability matters more than guaranteed coverage. The
setting applies to every instance in `CapacityProviderArns`. With `DaemonCritical=false`, instances can run workloads
without the sensor if the sensor fails to start or stops. For info, see
[Configure daemon criticality](#configure-daemon-criticality).

### Step 1: Get the Falcon sensor image
The AWS CloudFormation template requires that the Falcon sensor image is hosted in your Amazon ECR repo. Use the
CrowdStrike supported image pull script to retrieve the image from the CrowdStrike registry, and then copy the image to
your own ECR repo.

1. Retrieve the Falcon sensor image with the image pull script. For details on how to use the pull script, see [Retrieve the Falcon sensor image for your deployment](http://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment).
2. You can copy the Falcon sensor image directly to your ECR repository with the pull script. For instructions, see [Option 1: Pull an image and copy it to your private repo](https://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment#bc84c9e1).
3. The output from step 2 is two variables: `$FALCON_IMAGE_REPO` and `$FALCON_IMAGE_TAG`. These should be combined and set to `$FALCON_FULL_IMAGE_PATH`:
```bash
  export FALCON_FULL_IMAGE_PATH=$FALCON_IMAGE_REPO:$FALCON_IMAGE_TAG
```

### Step 2: Define your deployment parameters
Define your deployment parameters by setting variables that you add to your CloudFormation template. Use the table to
determine the right info.

<table>
  <thead>
    <tr>
      <th>Variable</th>
      <th>Description</th>
      <th>Example</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ECS_CLUSTER_NAME</td>
      <td>Your ECS cluster name</td>
      <td>your-ecs-cluster-name</td>
    </tr>
    <tr>
      <td>FALCON_CID</td>
      <td>Your CrowdStrike Customer ID</td>
      <td>XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX</td>
    </tr>
    <tr>
      <td>FALCON_FULL_IMAGE_PATH</td>
      <td>Full path to Falcon sensor image in ECR</td>
      <td>XXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:8.12.0-19609-1</td>
    </tr>
    <tr>
      <td>CAPACITY_PROVIDER_ARNS</td>
      <td>Comma separated list of ECS Managed Instances capacity provider ARNs. For info, see <a href="#capacity-providers">Capacity providers</a>.</td>
      <td>arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-1,arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-2</td>
    </tr>
    <tr>
      <td>ECS_EXECUTION_ROLE_ARN</td>
      <td>Your ECS task execution role with AmazonECSTaskExecutionRolePolicy attached</td>
      <td>arn:aws:iam::XXXXXXXXXXXX:role/ecsTaskExecutionRole</td>
    </tr>
  </tbody>
</table>

### Step 3: Get the CloudFormation template
Clone this CrowdStrike repo with the following command:
```bash
  git clone https://github.com/CrowdStrike/aws-cloudformation-falcon-sensor-ecs.git
```

### Step 4: Deploy with CloudFormation
First navigate to the cloned repository directory and locate the template and parameter file:
```bash
  cd aws-cloudformation-falcon-sensor-ecs/falcon-sensor-ecs-managed
```

**Note:** Deploying the sensor replaces the existing instances in your capacity providers in batches. For info, see
[Instance replacement](#instance-replacement).

Then choose one of these deployment methods:
#### Option A: Deploy using parameter file
**Important:** Ensure your AWS CLI is set to the correct region. For info, see [AWS CLI region](#aws-cli-region).

Edit the falcon-ecs-managed-daemon-parameters.json file to replace the placeholder values with your actual configuration:
```json
[
  "ECSClusterName=your-ecs-cluster-name",
  "FalconCID=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX",
  "FalconImagePath=XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:8.12.0-19609-1",
  "CapacityProviderArns=arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-1,arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-2",
  "ECSExecutionRoleArn=arn:aws:iam::XXXXXXXXXXXX:role/ecsTaskExecutionRole"
]
```

Deploy the AWS CloudFormation stack using the AWS CLI:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --template-file falcon-ecs-managed-daemon.yaml \
    --parameter-overrides file://falcon-ecs-managed-daemon-parameters.json
```

#### Option B: Deploy using command line parameters
**Important:** Ensure your AWS CLI is set to the correct region. For info, see [AWS CLI region](#aws-cli-region).

Deploy the AWS CloudFormation stack using the AWS CLI:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --template-file falcon-ecs-managed-daemon.yaml \
    --parameter-overrides \
      "ECSClusterName=$ECS_CLUSTER_NAME" \
      "FalconCID=$FALCON_CID" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH" \
      "CapacityProviderArns=$CAPACITY_PROVIDER_ARNS" \
      "ECSExecutionRoleArn=$ECS_EXECUTION_ROLE_ARN"
```

### Step 5: Verify the deployment
When the deployment is complete, verify the Falcon sensor is running on all ECS Managed Instances in your cluster.

1. Check the CloudFormation deployment status
```bash
  aws cloudformation describe-stacks \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --query "Stacks[0].StackStatus"
```
The CloudFormation stack will show `CREATE_COMPLETE` if deployed successfully.

2. Verify the ECS daemon is running and check the sensor count:
```bash
  DAEMON_ARN=$(aws cloudformation describe-stack-resources \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --logical-resource-id FalconECSDaemon \
    --query "StackResources[0].PhysicalResourceId" \
    --output text)

  aws ecs describe-daemon \
    --daemon-arn $DAEMON_ARN \
    --query "daemon.currentRevisions[0].totalRunningCount"
```
The output should match the number of ECS Managed Instances in your cluster.

**Important:** This check is required. A deployment can report `SUCCESSFUL` in the next step even when an instance has
no sensor. This can happen on first install if a replacement instance launches before the daemon exists and is filled
by other workloads. If the count is lower than the number of instances, drain the instances that have no sensor so ECS
replaces them with instances that run the sensor:
```bash
  aws ecs update-container-instances-state \
    --cluster $ECS_CLUSTER_NAME \
    --container-instances <container-instance-arn> \
    --status DRAINING
```

3. Confirm the deployment completed successfully:
```bash
  DEPLOYMENT_ARN=$(aws ecs describe-daemon \
    --daemon-arn $DAEMON_ARN \
    --query "daemon.deploymentArn" \
    --output text)

  aws ecs describe-daemon-deployments \
    --daemon-deployment-arns $DEPLOYMENT_ARN \
    --query "daemonDeployments[0].status"
```
The output should show `SUCCESSFUL`.

4. Confirm the sensors are registered in the Falcon console.
  - Go to Host Setup and management > Host management > Hosts.
  - Filter the host table to show your ECS cluster.

#### Template Configuration Parameters
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Required</th>
      <th>Description</th>
      <th>Example/Options</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ECSClusterName</td>
      <td>Yes</td>
      <td>Your ECS cluster name</td>
      <td>Example: your-ecs-cluster-name</td>
    </tr>
    <tr>
      <td>FalconCID</td>
      <td>Yes</td>
      <td>Your CrowdStrike Customer ID</td>
      <td>Example: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX</td>
    </tr>
    <tr>
      <td>FalconImagePath</td>
      <td>Yes</td>
      <td>Full path to Falcon sensor image in ECR</td>
      <td>Example: XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:8.12.0-19609-1</td>
    </tr>
    <tr>
      <td>CapacityProviderArns</td>
      <td>Yes</td>
      <td>Comma separated list of ECS Managed Instances capacity provider ARNs. The Falcon sensor runs only on instances managed by these capacity providers.</td>
      <td>Example: arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-1,arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-2</td>
    </tr>
    <tr>
      <td>ECSExecutionRoleArn</td>
      <td>Yes</td>
      <td>Your ECS task execution role already defined in IAM. Used to pull the Falcon sensor image from ECR and send container logs to CloudWatch.</td>
      <td>Example: arn:aws:iam::XXXXXXXXXXXX:role/ecsTaskExecutionRole</td>
    </tr>
    <tr>
      <td>APD</td>
      <td>No</td>
      <td>App Proxy Disable (APD)</td>
      <td>Options: true, false</td>
    </tr>
    <tr>
      <td>APH</td>
      <td>No</td>
      <td>App Proxy Host (APH)</td>
      <td>Example: proxy.example.com</td>
    </tr>
    <tr>
      <td>APP</td>
      <td>No</td>
      <td>App Proxy Port (APP)</td>
      <td>Example: 3128</td>
    </tr>
    <tr>
      <td>Trace</td>
      <td>No</td>
      <td>Set Trace Level</td>
      <td>
        Default: none<br>
        Options: err, warn, info, debug
      </td>
    </tr>
    <tr>
      <td>Feature</td>
      <td>No</td>
      <td>Falcon sensor feature Options</td>
      <td>Example: feature1,feature2</td>
    </tr>
    <tr>
      <td>Tags</td>
      <td>No</td>
      <td>Comma separated list of tags for sensor grouping</td>
      <td>Example: tag1,tag2,tag3</td>
    </tr>
    <tr>
      <td>ProvisioningToken</td>
      <td>No</td>
      <td>Falcon provisioning token value</td>
      <td>Example: abc123def456</td>
    </tr>
    <tr>
      <td>Billing</td>
      <td>No</td>
      <td>Falcon billing value</td>
      <td>Options: default, metered</td>
    </tr>
    <tr>
      <td>Cloud</td>
      <td>No</td>
      <td>CrowdStrike cloud region. The sensor can auto-derive the region; only set this if auto-detection fails in your environment.</td>
      <td>
        Default: (auto-detect)<br>
        Options: us-1, us-2, us-3, eu-1, us-gov-1, us-gov-2
      </td>
    </tr>
    <tr>
      <td>EnableLogging</td>
      <td>No</td>
      <td>Enable logging for the Falcon ECS service</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>EnableExecuteCommand</td>
      <td>No</td>
      <td>Enable ECS exec on the Falcon sensor container. This should be enabled only when required.</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>ECSTaskRoleArn</td>
      <td>No</td>
      <td>Your ECS task role already defined in IAM, with the SSM permissions for ECS exec. Required if execute-command is enabled.</td>
      <td>Example: arn:aws:iam::XXXXXXXXXXXX:role/yourEcsTaskRole</td>
    </tr>
    <tr>
      <td>DeploymentBakeTimeInMinutes</td>
      <td>No</td>
      <td>Minutes to wait after a successful daemon deployment before considering it stable.</td>
      <td>Default: 5</td>
    </tr>
    <tr>
      <td>DeploymentDrainPercent</td>
      <td>No</td>
      <td>Percentage of instances that can be updated simultaneously during a daemon deployment. Lower values reduce blast radius during updates.</td>
      <td>Default: 10</td>
    </tr>
    <tr>
      <td>TaskDefinitionFamily</td>
      <td>No</td>
      <td>Task definition family name. Leave empty to use the stack name. Set this to preserve revision history when migrating from a previous deployment.</td>
      <td>Default: (stack name)</td>
    </tr>
    <tr>
      <td>DaemonCritical</td>
      <td>No</td>
      <td>Mark the Falcon sensor daemon as critical. For info, see <a href="#configure-daemon-criticality">Configure daemon criticality</a>.</td>
      <td>
        Default: true<br>
        Option: false
      </td>
    </tr>
    <tr>
      <td>EnableResourceLimits</td>
      <td>No</td>
      <td>Enable CPU and memory hard limits for the sensor</td>
      <td>
        Default: false<br>
        Option: true
      </td>
    </tr>
    <tr>
      <td>SensorCpuReservation</td>
      <td>No</td>
      <td>Cpu Reservation for Falcon sensor container. The default is only a placeholder. CPU usage should be measured and reservation adjusted accordingly.</td>
      <td>Default: 256 (CPU units)</td>
    </tr>
    <tr>
      <td>SensorCpuLimit</td>
      <td>No</td>
      <td>CPU hard limit for Falcon sensor task. CPU limit must be non-zero, and greater than or equal to CPU reservation.</td>
      <td>
        Default: 0<br>
        Example: 512 (CPU units)
      </td>
    </tr>
    <tr>
      <td>SensorMemoryReservation</td>
      <td>No</td>
      <td>Memory Reservation for Falcon sensor container. The default is only a placeholder. Memory usage should be measured and reservation adjusted accordingly.</td>
      <td>Default: 512 (MiB)</td>
    </tr>
    <tr>
      <td>SensorMemoryLimit</td>
      <td>No</td>
      <td>Memory hard limit for Falcon sensor task. Memory limit must be non-zero, and greater than memory reservation.</td>
      <td>
        Default: 0<br>
        Example: 1024 (MiB)
      </td>
    </tr>
  </tbody>
</table>

## Upgrade the Falcon sensor
To upgrade the Falcon sensor, follow the same procedure as the initial deployment. The only difference is that you'll
need to get the new sensor version and update your deployment with the new image path.

**Note:** An upgrade replaces every instance the sensor runs on. The upgraded instances register as new hosts with new
AIDs, and the hosts for the old instances go offline. For info, see [Instance replacement](#instance-replacement).

To upgrade the Falcon sensor, you must complete these steps:

- [Step 1: Get the new Falcon sensor image and update your ECR repository](#step-1-get-the-new-falcon-sensor-image-and-update-your-ecr-repository)
- [Step 2: Deploy the updated stack](#step-2-deploy-the-updated-stack)
- [Step 3: Verify that the upgrade completed successfully](#step-3-verify-that-the-upgrade-completed-successfully)

### Step 1: Get the new Falcon sensor image and update your ECR repository
The AWS CloudFormation template requires that the Falcon sensor image is hosted in your Amazon ECR repo. Use the
CrowdStrike supported image pull script to retrieve the image from the CrowdStrike registry, and then copy the image to
your own ECR repo.

1. Retrieve the Falcon sensor image with the image pull script. For details on how to use the pull script, see [Retrieve the Falcon sensor image for your deployment](http://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment).
2. You can copy the Falcon sensor image directly to your ECR repository with the pull script. For instructions, see [Option 1: Pull an image and copy it to your private repo](https://falcon.crowdstrike.com/documentation/page/eb6c645d/retrieve-the-falcon-sensor-image-for-your-deployment#bc84c9e1).
3. The output from step 2 is two variables: `$FALCON_IMAGE_REPO` and `$FALCON_IMAGE_TAG`. These should be combined and set to `$FALCON_FULL_IMAGE_PATH`:
```bash
  export FALCON_FULL_IMAGE_PATH=$FALCON_IMAGE_REPO:$FALCON_IMAGE_TAG
```

### Step 2: Deploy the updated stack
After you get the new Falcon sensor version, redeploy the stack using one of the 2 options:

First navigate to the cloned repository directory and locate the template and parameter file:
```bash
  cd aws-cloudformation-falcon-sensor-ecs/falcon-sensor-ecs-managed
```

Then choose one of these deployment methods:
#### Option A: Deploy using parameter file
**Important:** Ensure your AWS CLI is set to the correct region. For info, see [AWS CLI region](#aws-cli-region).

Edit the falcon-ecs-managed-daemon-parameters.json file to replace the placeholder values with your actual configuration:
```json
[
  "ECSClusterName=your-ecs-cluster-name",
  "FalconCID=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-XX",
  "FalconImagePath=XXXXXXXXXXXX.XXX.ecr.region.amazonaws.com/falcon-sensor:8.12.0-19609-1",
  "CapacityProviderArns=arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-1,arn:aws:ecs:region:XXXXXXXXXXXX:capacity-provider/cp-2",
  "ECSExecutionRoleArn=arn:aws:iam::XXXXXXXXXXXX:role/ecsTaskExecutionRole"
]
```

Deploy the AWS CloudFormation stack using the AWS CLI:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --template-file falcon-ecs-managed-daemon.yaml \
    --parameter-overrides file://falcon-ecs-managed-daemon-parameters.json
```

#### Option B: Deploy using command line parameters
**Important:** Ensure your AWS CLI is set to the correct region. For info, see [AWS CLI region](#aws-cli-region).

Deploy the AWS CloudFormation stack using the AWS CLI:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --template-file falcon-ecs-managed-daemon.yaml \
    --parameter-overrides \
      "ECSClusterName=$ECS_CLUSTER_NAME" \
      "FalconCID=$FALCON_CID" \
      "FalconImagePath=$FALCON_FULL_IMAGE_PATH" \
      "CapacityProviderArns=$CAPACITY_PROVIDER_ARNS" \
      "ECSExecutionRoleArn=$ECS_EXECUTION_ROLE_ARN"
```

### Step 3: Verify that the upgrade completed successfully
After you redeploy the stack with the new Falcon sensor, you can verify that the upgrade was successful and that the
upgraded Falcon sensor is running on all ECS Managed Instances in your cluster. You verify an upgrade in the same way you verify
the initial deployment. For details, see [Step 5: Verify the deployment](#step-5-verify-the-deployment).

## Uninstall the sensor
Delete the Falcon sensor daemon stack:

```bash
  aws cloudformation delete-stack \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME
```

ECS Managed Instances drains and replaces every instance the sensor daemon ran on, regardless of the `DaemonCritical`
value, so no Falcon sensor files or processes remain. No separate cleanup step is needed, including before a
re-installation. For info, see [Instance replacement](#instance-replacement).

## Advanced Configuration
Advanced configuration includes additional configuration options outside of the basic deployment setup. To enable any
of the advanced configuration options, add the parameters to the parameters file and then redeploy the stack.

1. Add the required parameters to the falcon-ecs-managed-daemon-parameters.json file.
2. Redeploy with CloudFormation stack with the updated parameter file:
```bash
  aws cloudformation deploy \
    --stack-name falcon-ecs-managed-daemon-$ECS_CLUSTER_NAME \
    --template-file falcon-ecs-managed-daemon.yaml \
    --parameter-overrides file://falcon-ecs-managed-daemon-parameters.json
```
3. Alternative: Use `--parameter-overrides` to pass the parameters directly to the template.

**Note:** Redeploying with changed sensor parameters replaces the instances the sensor runs on. For info, see
[Instance replacement](#instance-replacement).

**Tip:** To see all the deployment steps, see [Deploy on an ECS cluster with ECS Managed Instances](#deploy-the-falcon-sensor-on-an-ecs-cluster-with-ecs-managed-instances).

### Configure deployment behavior
Control how sensor updates roll out across your cluster during stack updates. By default, ECS updates 10% of instances
at a time and waits 5 minutes after each batch before proceeding.

<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>DeploymentBakeTimeInMinutes</td>
      <td>Minutes to wait after a successful deployment before marking it stable and proceeding to the next batch</td>
      <td>Default: 5</td>
    </tr>
    <tr>
      <td>DeploymentDrainPercent</td>
      <td>Maximum percentage of instances updated simultaneously. On a 10-instance cluster, 10% means 1 instance at a time.</td>
      <td>Default: 10</td>
    </tr>
  </tbody>
</table>

**Note:** Setting `DeploymentDrainPercent` too high reduces protection coverage during updates. For security sensors,
conservative values are recommended so the fleet is never broadly unprotected at the same time.

### Configure daemon criticality
The `DaemonCritical` parameter controls what ECS does when the Falcon sensor fails to start or stops. By default, the
sensor daemon is critical.

<table>
  <thead>
    <tr>
      <th>Event</th>
      <th>DaemonCritical=true (default)</th>
      <th>DaemonCritical=false</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Sensor fails to start on a new instance</td>
      <td>The instance doesn't become active, so no workloads run without the sensor</td>
      <td>The instance becomes active and runs workloads without the sensor</td>
    </tr>
    <tr>
      <td>Sensor stops or becomes unhealthy</td>
      <td>ECS drains and replaces the instance</td>
      <td>The instance and its workloads keep running without the sensor. If the sensor task is stopped, ECS doesn't start a new one on that instance</td>
    </tr>
  </tbody>
</table>

Keep the default for most deployments. A critical sensor guarantees that no workload runs on an unprotected instance,
and an instance whose sensor is stopped is replaced.

Consider `DaemonCritical=false` only if workload availability matters more than guaranteed coverage. For example, if
your workloads can't tolerate instance drains, or if you don't want a sensor or image pull failure to block cluster
scaling. If you set `DaemonCritical=false`:

- Monitor for daemon task failures. ECS emits an Amazon EventBridge event when a daemon task fails to start.
- Regularly compare the sensor count with the instance count. For info, see [Step 5: Verify the deployment](#step-5-verify-the-deployment).
- To restore the sensor on an instance whose sensor task stopped, drain the instance so ECS replaces it with a new
  instance that runs the sensor. For the command, see [Step 5: Verify the deployment](#step-5-verify-the-deployment).

### Enable logging
Enable logging to get visibility into the Falcon sensor for Linux operations. By default, logging is disabled.

Enable logging with these parameters:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableLogging</td>
      <td>Enable logging for the sensor container</td>
      <td>true</td>
    </tr>
  </tbody>
</table>

**Note:** Container logs are sent to CloudWatch using the ECS execution role passed in `ECSExecutionRoleArn`. For info,
see [Required IAM Permissions](#required-iam-permissions).

#### Access the Logs
After enabling logging, you can access the Falcon sensor logs through CloudWatch. The logs are stored in a log group
created specifically for the sensor.

To view the logs:

1. Open the AWS CloudWatch console.
2. Navigate to **Logs > Log groups**.
3. Find the log group named `/aws/ecs/{your-cluster-name}/crowdstrike/falcon-daemon-service`.
4. The logs are separated into streams:
   1. Streams with prefix `falcon-node-init` contain logs from the initialization container
   2. Streams with prefix `falcon-node-sensor` contain logs from the main sensor container

Each ECS Managed Instance in your cluster will have its own log streams, allowing you to monitor sensor activity across your entire environment.

### Enable ECS exec
Execute commands in the Falcon sensor for Linux for troubleshooting. ECS exec should only be enabled if and when required.

Enable ECS exec with these parameters:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableExecuteCommand</td>
      <td>Enable ECS exec</td>
      <td>true</td>
    </tr>
    <tr>
      <td>ECSTaskRoleArn</td>
      <td>ECS task role with required SSM permissions</td>
      <td>
        IAM ARN for a role with these SSM permissions:<br>
          - ssmmessages:CreateControlChannel<br>
          - ssmmessages:CreateDataChannel<br>
          - ssmmessages:OpenControlChannel<br>
          - ssmmessages:OpenDataChannel
      </td>
    </tr>
  </tbody>
</table>

**Note:** ECS exec applies only to sensor tasks started after you enable it. Like any parameter change, redeploying
with `EnableExecuteCommand=true` redeploys the sensor daemon. For info, see [Instance replacement](#instance-replacement).
The instances also need outbound HTTPS access to the Systems Manager `ssmmessages` endpoint, through a NAT gateway, an
internet gateway with a public IP address, or a VPC endpoint.

> [!IMPORTANT]
> Do not run another task with ECS exec enabled and `pidMode: host` on an instance where you want to
> exec into the sensor. The sensor shares the host PID namespace, and when a second exec-enabled task joins that
> namespace, exec into the sensor container fails with `TargetNotConnectedException`. ECS still reports the sensor's
> `ExecuteCommandAgent` as `RUNNING`.

Exec into the sensor doesn't recover after the other task stops. To restore it, drain
the instance so ECS replaces it with a new instance and sensor task:
```bash
  aws ecs update-container-instances-state \
    --cluster $CLUSTER_NAME \
    --container-instances <container-instance-arn> \
    --status DRAINING
```
Tasks that use `pidMode: host` without ECS exec enabled do not affect exec into the sensor.

#### Use ECS Exec for troubleshooting
After enabling ECS exec, you can execute commands inside the Falcon sensor container using the AWS CLI:

1. Find the running task ID for the Falcon sensor. Each instance runs one sensor task. To exec into the sensor on a
   specific instance, add `--container-instance <container-instance-arn>`:
```bash
  export CLUSTER_NAME=<your-cluster-name>
  export SENSOR_TASK_ID=$(aws ecs list-tasks \
    --cluster $CLUSTER_NAME \
    --daemon-name crowdstrike-falcon-node-daemon \
    --desired-status RUNNING \
    --query 'taskArns[0]' \
    --output text)
```

2. Execute commands in the sensor container:
```bash
  aws ecs execute-command \
    --cluster $CLUSTER_NAME \
    --task $SENSOR_TASK_ID \
    --container crowdstrike-falcon-node-sensor \
    --command "/bin/bash" \
    --interactive
```
This will open an interactive shell inside the Falcon sensor container, allowing you to run troubleshooting commands.

**Warning:** Only enable ECS exec when needed for troubleshooting, and disable it when finished by setting
`EnableExecuteCommand=false` and redeploying the stack. ECS exec gives direct access to the container environment.
Restrict access to users who have appropriate permissions.

### Enable resource limits
Resource limits are constraints that can be set on the amount of CPU and memory resources allocated to the sensor. They can help
prevent the sensor from overutilizing resources so that it runs within a predictable and controlled environment to ensure that it
does not consume excessive resources, which can impact other containers in the same environment.

Setting resource limits without proper understanding of the sensor's resource requirements can lead to unintended consequences.
If the limits are set too low, the sensor may not have enough resources to function properly. Set resource limits with caution,
and be aware of these potential issues if resources are too low:

- Sensor throttling: CPU limits may be reached, impacting sensor performance.
- Unexpected termination: Memory limits may cause the sensor to terminate unexpectedly.

Enable resource limits with these parameters. When `EnableResourceLimits` is `true`, both `SensorCpuLimit` and
`SensorMemoryLimit` must be non-zero, or the deployment fails parameter validation:
<table>
  <thead>
    <tr>
      <th>Parameter</th>
      <th>Description</th>
      <th>Value</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>EnableResourceLimits</td>
      <td>Enables resource limits for the sensor</td>
      <td>true</td>
    </tr>
    <tr>
      <td>SensorCpuLimit</td>
      <td>Maximum CPU resources (CPU units) for the sensor</td>
      <td>≥ SensorCpuReservation</td>
    </tr>
    <tr>
      <td>SensorMemoryLimit</td>
      <td>Maximum memory resources (MiB) for the sensor</td>
      <td>> SensorMemoryReservation</td>
    </tr>
  </tbody>
</table>
