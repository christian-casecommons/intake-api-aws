# Intake API AWS Ansible Deployment Playbook

This playbook deploys the Casecommons Intake API application to AWS using Ansible and CloudFormation.

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

## Playbook Structure

- [`site.yml`](site.yml) - the primary playbook to run.  Note you can create your own playbooks if required.
- [`ansible.cfg](ansible.cfg)` - sets Ansible defaults for this playbook.
- [`inventory`](inventory) - defines each environment you want to deploy to
- [`roles/requirements.yml`](roles/requirements.yml) - defines requires Ansible roles for this playbook
- [`group_vars`](group_vars) folder - defines global and environment specific settings
- [`templates/site.yml.j2`](templates/site.yml.j2) - CloudFormation template in YAML/Jinja format
- `build/<timestamp>` folder - dynamically created for each playbook run.  Contains build artifacts including the generate CloudFormation stack file in YAML and compact JSON format that is uploaded to the CloudFormation service

