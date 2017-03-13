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

## Caveats

The Ruby on Rails Elasticsearch client included in the Intake API application does not honour the `no_proxy` environment variable setting, meaning you cannot bypass the Elasticsearch URL when an HTTP proxy is configured.

As the Intake API application is deployed in a private subnet with no direct Internet access, an HTTP proxy must be used for any communications that the Intake API container requires to the Internet.

At present the Intake API container includes an entrypoint script that communicates with the AWS KMS service (which has a public API endpoint) to [inject decrypted secrets into the environment](#consuming-secrets-in-containers), hence an HTTP proxy is required for the container.  However this breaks Intake API communication with Elasticsearch, as all Elasticsearch communications are attempted via the HTTP proxy, which is whitelisted to only permit communications to AWS service endpoints.

To overcome these issues, the entrypoint script checks for the existence of an environment variable called `CLEAR_PROXY`, which unsets all proxy related environment variables:

```
...
...
if [[ -n ${CLEAR_PROXY} ]]
then
  unset http_proxy https_proxy no_proxy HTTP_PROXY HTTPS_PROXY NO_PROXY
fi
...
...
```

This is performed AFTER the script communicates with the KMS service, ensuring the KMS secret injection feature works, but then allowing the application to communicate directly with Elasticsearch.

> IMPORTANT: This approach assumes the Intake API application only communicates with a private Elasticsearch instance and does not require Internet access.  This approach will not work if the application requires access to the Internet.

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

## Generating Secrets

This playbook includes secrets that are encrypted using the AWS Key Management Service (KMS).  Secrets management is based upon the following assumptions:

- An existing KMS key already exists.  
- You have permissions to encrypt using the KMS key to create secrets
- Any resources that need to decrypt the secret have permissions to decrypt using the KMS key

### Encrypting a secret

The following example demonstrates encrypting a secret using the AWS CLI:

```
$ aws kms encryption --key-id 3ea941bf-ee54-4941-8f77-f1dd417667cd --plaintext 'my-super-secret-password'
{
    "KeyId": "arn:aws:kms:us-west-2:429614120872:key/3ea941bf-ee54-4941-8f77-f1dd417667cd",
    "CiphertextBlob": "AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITog"
}
```

You should then use the `CiphertextBlob` property from the AWS CLI output as the value for your secret configuration variable.

e.g. in `group_vars/<env>/vars.yml`:

```
# Application settings
config_db_username: myapp
config_db_password: AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITog
...
...
```

### Consuming secrets

Most AWS services do not include support for processing KMS encrypted variables, therefore you need a helper CloudFormation custom resource that can decrypt a given secret and return the plaintext value to other AWS resources.

This playbook already includes a [`KMSDecrypter` Lambda function](https://github.com/Casecommons/lambda-cfn-kms) that provides this capability.

With the supporting `KMSDecrypter` resource in place, you must then create a CloudFormation custom resource for each secret, and then reference the secret custom resource for any AWS resource variable that requires the plaintext value of your secret.

For example in the following configuration snippet, notice the following:

- The `DbPassword` stack input references the ciphertext defined in `config_db_password`
- The `DbPasswordDecrypt` resource `ServiceToken` property references the `KMSDecrypter` resource ARN
- The `DbPasswordDecrypt` resource passes the `CipherText` property to `KMSDecrypter`, which is the ciphertext generated previously and passed to the `DbPassword` input parameter.
- The `ApplicationDatabase` resource then references the `DbPasswordDecrypt` resource to obtain the plain text value of the secret.

```
# From group_vars/<env>/vars.yml
...
...
config_db_password: AQECAHgohc0dbuzR1L3lEdEkDC96PMYUEV9nITog
...
...

# From group_vars/all/vars.yml
...
...
cf_stack_inputs:
  DbPassword: "{{ config_db_password }}"
...
...

# From templates/stack.yml.j2
...
...
Resources:
...
...
  DbPasswordDecrypt:
    Type: "Custom::KMSDecrypt"
    Properties:
      ServiceToken: 
        Fn::Sub: ${KMSDecrypter.Arn}
      Ciphertext: { "Ref": "DbPassword" }
  ApplicationDatabase:
    Type: "AWS::RDS::DBInstance"
    Properties:
      MasterUserPassword:
        Fn::Sub: ${DbPasswordDecrypt.Plaintext}
...
...
```

### Consuming secrets in Containers

The base image for the Intake API Docker image used by the `ApplicationTaskDefinition` includes an entrypoint script that will automatically attempt to decrypt environment variable values prefixed using `KMS_xxxx_xxxx` and inject a new environment variable with the `KMS_` prefix removed and the decrypted plaintext set to the new environment variable value.

For example, the `ApplicationTaskDefinition` resource includes the following environment variable configuration:

```
# From templates/stack.yml.j2
...
...
Resources:
...
...
  ApplicationTaskDefinition:
    Type: "AWS::ECS::TaskDefinition"
    Properties:
      ...
      ...
      ContainerDefinitions:
      - Name: intake-api
        ...
        ...
        Environment:
          - Name: KMS_PG_PASSWORD
            Value: { "Ref": "DbPassword" }
...
...
```

At container startup, the entrypoint script with decrypt the value of `KMS_PG_PASSWORD` and export a new environment variable `PG_PASSWORD` with the plaintext value of the `DbPassword` input parameter ciphertext.

## License

Copyright (C) 2017.  Case Commons, Inc.

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU Affero General Public License as published by the Free
Software Foundation, either version 3 of the License, or (at your option) any
later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

See www.gnu.org/licenses/agpl.html
