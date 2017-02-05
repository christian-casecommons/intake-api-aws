# Intake API AWS Ansible Deployment Playbook

This playbook deploys the Casecommons Intake API application to AWS using Ansible and CloudFormation.

A CloudFormation stack is generated for each environment that includes the following AWS resources:

- `ApplicationLoadBalancer` resource - public front end ALB for the intake API application
- `ApplicationDatabase` resource - creates an RDS instance with Postgres database installed
- `ApplicationAutoscaling` resource - EC2 Autoscaling Group for ECS container instances running the intake API application
- `ApplicationCluster` resource - ECS cluster for ECS container instances running the intake API application
- `ApplicationTaskDefinition` resource - ECS task definition including nginx and intake API container definitions for running the intake accelerator application
- `AdhocTaskDefinition` resource - ECS task definition including an intake API container definition for running adhoc deployment tasks
- `IntakeApiService` resource - ECS service that schedules containers configured in the `ApplicationTaskDefinition` task definition onto the `ApplicationCluster` ECS cluster, and manages service instance registration and deployments using the `ApplicationLoadBalancer` resource
- `DbCreateTask`, `DbMigrateTask`, `SearchMigrateTask`, `SearchReindexTask` resources - custom resources that run database creation and migration, and search migration and reindex tasks required to bootstrap and update database and Elasticsearch resources for each deployment.
- `ElasticsearchLoadBalancer` resource - private front end ALB for Elasticsearch
- `ElasticsearchAutoscaling` resource - EC2 Autoscaling Group for ECS container instances running Elasticsearch
- `ElasticsearchCluster` resource - ECS cluster for ECS container instances running Elasticsearch
- `ElasticsearchTaskDefinition` resource - ECS task definition including a container definition for running Elasticsearch
- `ElasticsearchService` resource - ECS service that schedules containers configured in the `ElasticsearchTaskDefinition` task definition onto the `ElasticsearchCluster` ECS cluster, and manages service instance registration and deployments using the `ElasticsearchLoadBalancer` resource
- `KMSDecrypter` - CloudFormation custom resource Lambda function that decrypts KMS-encrypted variables.  This is used to pass the configured database password securely to the `ApplicationDatabase` resource
- `EcsTaskRunner` - CloudFormation custom resource Lambda function that executes ECS tasks defined in the `DbCreateTask`, `DbMigrateTask`, `SearchMigrateTask`, `SearchReindexTask` resources

## Quick Start

### Installing/Updating Ansible Roles

This playbook relies on several Ansible roles, which are defined in the [`roles/requirements.yml`](roles/requirements.yml) file.  

After cloning this repository, you must first install the roles by running the following command at the root of the repository:

```
$ ansible-galaxy install -r roles/requirements.yml
```

To update an existing role and force the new version of the role to overwrite the existing role, add the `--force` flag:

```
$ ansible-galaxy install -r roles/requirements.yml --force
```

### Generating a CloudFormation Stack File

To generate a CloudFormation stack file for a given environment without deploying the stack, specify the `generate` tag:

```
$ ansible-playbook site.yml =e env=dev --tags generate
```

A CloudFormation stack file will be created in a dynamically create folder `build/<timestamp>`.

> You can override the build folder by setting the variable `cf_build_folder` in your global or environment settings

### Creating/Updating a Stack

To deploy to an environment called `dev`:

```
$ ansible-playbook site.yml -e env=dev
```

Note your local environment must be setup with appropriate privileges to assume the role defined by the `sts_role_arn` value defined for the environment.

### Deleting a Stack

To delete a stack you must set the `cf_delete_stack` variable to `true`:

```
$ ansible-playbook site.yml -e env=dev -e cf_delete_stack=true
```

### Adding a new environment

To add a new environment you must first configure the environment in the local [`inventory`](inventory) file.  The following inventory file includes two environments, `dev` and `staging`.

```
[dev]
dev ansible_connection=local

[staging]
staging ansible_connection=local
```

You must also create a folder for the environment in the [`group_vars`](group_vars) folder and create a file called `group_vars/<env>/vars.yml`:

```
$ mkdir -p group_vars/staging
$ touch group_vars/staging/vars.yml
```

You can now add environment specific configuration settings in the newly created `vars.yml` file.

### Playbook Structure Reference

- [`site.yml`](site.yml) - the primary playbook to run.  Note you can create your own playbooks if required.
- [`ansible.cfg`](ansible.cfg) - sets Ansible defaults for this playbook.
- [`inventory`](inventory) - defines each environment you want to deploy to
- [`roles/requirements.yml`](roles/requirements.yml) - defines required Ansible roles for this playbook
- [`group_vars`](group_vars) folder - defines global and environment specific settings
- [`templates/site.yml.j2`](templates/site.yml.j2) - CloudFormation template in YAML/Jinja format
- `build/<timestamp>` folder - dynamically created for each playbook run.  Contains build artifacts including the generate CloudFormation stack file in YAML and compact JSON format that is uploaded to the CloudFormation service

## Configuring an environment

It is important to understand that this playbook essentially is a template generator that generates a CloudFormation stack template for a given environment from the following inputs:

- CloudFormation template - generic definition of the target environment with CloudFormation input parameters and/or Jinja markup to inject environment specific settings.  By convention this is located in `./templates/stack.yml.j2` but can be located in a custom path by configuring the `cf_template_path` variable using an absolute path - e.g. `cf_template_path: ./templates/custom.yml.j2`.  Note that the `aws-cloudformation` role includes a number of embedded templates that can be referenced using a relative path - e.g. `cf_template_path: templates/network.yml.j2` will load the `aws-cloudformation` role network template.
- Global settings - global settings defined in the `group_vars/all` folder.  By default all global settings are stored in `group_vars/all/vars.yml`, however Ansible will read variables from any file in `group_vars/all` so you can structure your variables into separate files accordingly.
- Environment specific settings - environment specific settings defined in the `group_var/<env>/folder`.  The `env` variable must reference the correct environment that you are setting.  These settings following the same conventions as global settings.

As a best practice, you should adopt the following guidelines:

- Where possible, try to define your environment specific settings as CloudFormation input parameters.  This approach may introduce an extra layer of variable indirection, but ensures your generated templates clearly communicate inputs to the stack, and allows the generated stack template to be overriden for more advanced scenarios.
- Define stack inputs globally (i.e. in `group_vars/all/vars.yml`) in the `cf_stack_inputs` dictionary.  Each input parameter should then reference an environment specific configuration variable, allowing the environment specific settings to be injected into the global `cf_stack_inputs` directory.
- Define stack name (`cf_stack_name`) and other `aws-cloudformation` inputs (i.e. anything prefixed with `cf_xxxx_xxxx`) globally.
- Define all configuration settings that are not related to Ansible roles with a prefix of `config_xxxx` - e.g. `config_db_username`.
- Define the required `sts_role_arn` input for the `aws-sts` role in each environment.

> Note that CloudFormation cannot deal with more advanced conditional scenarios, so don't be afraid to use more advanced Jinja template generation techniques if native CloudFormation techniques won't work.

### Example

Assume the following stack definition defined in `templates/stack.yml.j2`:

```
AWSTemplateFormatVersion: "2010-09-09"

Description: Intake Accelerator - {{ env }}

Parameters:
  ApplicationAMI:
    Type: String
    Description: Application Amazon Machine Image ID
  ApplicationInstanceType:
    Type: String
    Description: Application EC2 Instance Type
  Environment:
    Type: String
    Description: The target environment

Resources:
  ApplicationInstance:
    Type: "AWS::EC2::Instance"
    Properties:
      ImageId: { "Ref": "ApplicationAMI" }
      InstanceType: { "Ref": "ApplicationInstanceType" }
      Tags:
        - Key: Environment
          Value: { "Ref": "Environment" }
...
...

```

Notice that this template uses Jinja markup to define the stack description, referencing the environment (as specified by the `env` variable), as the stack description cannot reference the `Environment` input parameter.

The template includes three input parameters that are defined in the stack:

- `ApplicationAMI` 
- `ApplicationInstanceType`
- `Environment`

Each of these input parameters need to be defined in the global `cf_stack_inputs` dictionary in `groups_vars/all/vars.yml`

```
cf_stack_inputs:
  ApplicationAMI: "{{ config_application_ami }}"
  ApplicationInstanceType: "{{ config_application_instance_type | default('t2.micro') }}"
  Environment: "{{ env }}"
```

Notice that each input parameter references an environment specific setting:

- `ApplicationAMI` - references `config_application_ami`, which MUST be defined
- `ApplicationInstanceType` - references `config_application_instance_type`, which defaults to `t2.micro` if not configured
- `Environment` - references the `env` setting, which is passed in at execution time.  

Each of the environment settings can then be defined in `group_vars/<env>/vars.yml`.

For example for the `dev` environment (`group_vars/dev/vars.yml`):

```
# STS role settings
# This role must have privileges to perform all CloudFormation provisioning tasks for the 'dev' environment
sts_role_arn: "arn:aws:iam::222222222222:role/cfnBuilder"

# Environment settings
config_application_ami: ami-2222222

```

In the `prod` environment (`group_vars/prod/vars.yml`), notice the application instance type is modified from the default global setting:

```
# STS role settings
# This role must have privileges to perform all CloudFormation provisioning tasks for the 'prod' environment
sts_role_arn: "arn:aws:iam::111111111111:role/cfnBuilder"

# Environment settings
config_application_ami: ami-1111111
config_application_instance_type: t2.large

```

