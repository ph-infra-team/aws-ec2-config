# test_awx_aws_connect

AWX test playbook repository for validating private EC2 management through AWS Systems Manager Session Manager.

## Purpose

This repo proves the enterprise access model:

```text
GitLab pipeline -> central AWX configure template -> AWX job -> AWS SSM connection plugin -> private EC2 instance
```

No SSH, public IP, or bastion host is required.
No long-term team AWS access keys are stored in this repo or in repo-level
GitLab variables.

## Requirements

The AWX execution environment must include:

- `amazon.aws` collection
- `community.aws` collection
- `boto3`
- `botocore`

Use the registered AWX Execution Environment:

```text
ee-aws-ssm
```

The `ansible-awx` image is only the GitLab pipeline control image. It calls AWX
APIs. The AWX job itself runs inside `ee-aws-ssm`.

## AWX Resources

AWX resources are declared in:

```text
awx/team_vars.yml
```

The shared GitLab pipeline launches the trusted central AWX job template:

```text
central-awx-configure-resources
```

That central job creates or updates this repo's AWX Project and Job Template.

Application engineers should edit `awx/team_vars.yml`, playbooks, and
playbook-owned `group_vars`. They should not edit the central AWX project or
central templates.

## AWX Inventory

Use the inventory created by Terraform pipeline sync:

```text
Organization: platform-team
Inventory: maas-ec2-dev
Group: linux
```

The host should have variables from Terraform output, including:

```yaml
ansible_host: 172.31.22.206
private_ip: 172.31.22.206
instance_id: i-xxxxxxxxxxxxxxxxx
```

`instance_id` is required for SSM. The AWX host name and `ansible_host` may be
the private IP for readability, but AWS `StartSession` must target the EC2
instance ID.

## Group Variables

The `linux` group uses SSM connection variables from:

```text
playbooks/group_vars/linux/ssm.yml
```

The file lives under `playbooks/group_vars` so AWX/Ansible loads it when running
`playbooks/ping.yml`. If it is placed only at the repository root, AWX may not
load it and Ansible falls back to SSH.

This is the chosen pattern for this repo:

```text
Option A: playbook-owned group_vars
```

That keeps the connection behavior beside the playbook while Terraform/AWX own
the actual inventory and host records. If every job in the enterprise should use
SSM for a given AWX group, move the same variables into AWX inventory group
variables through central config-as-code instead.

Required runtime variable:

```text
AWX_SSM_BUCKET_NAME
```

This bucket is used by the `amazon.aws.aws_ssm` connection plugin for module transfer.
Use a dedicated encrypted SSM transfer bucket in production. The current shared
bucket is acceptable only when approved by the platform owner.

## Test Job Template

The shared pipeline creates or updates this job template:

```text
Name: test-awx-aws-connect-ping
Inventory: maas-ec2-dev
Project: test-awx-aws-connect
Playbook: playbooks/ping.yml
Execution Environment: ee-aws-ssm
Credentials:
  <AWX_BASE_AWS_CREDENTIAL_NAME>
  test-awx-aws-connect-assume-role
```

The launch job runs against:

```text
linux
```

Expected result:

```text
raw hostname command returns the EC2 hostname
raw Python version command returns /usr/bin/python3 version
```

## Required CI/CD Variables

These should be inherited from the `infra_team/automation` GitLab group, not
set separately in every app repo:

```text
AWX_HOST
AWX_USERNAME
AWX_PASSWORD
AWX_SCM_CREDENTIAL_NAME
AWX_BASE_AWS_CREDENTIAL_NAME
AWX_SSM_BUCKET_NAME
AWS_DEFAULT_REGION
```

This repo or its owning team group must provide the team role ARN:

```text
AWS_ASSUME_ROLE_ARN
```

`AWX_BASE_AWS_CREDENTIAL_NAME` is the central AWS credential in AWX that is
allowed to call `sts:AssumeRole`. It should be the custom
`AWS Shared Credentials File` credential, not the built-in AWX `Amazon Web
Services` credential. `AWS_ASSUME_ROLE_ARN` is the team account role that grants
SSM and S3 transfer-bucket permissions.

The shared pipeline creates an AWX credential named:

```text
test-awx-aws-connect-assume-role
```

using credential type:

```text
AWS Assume Role Profile
```

The job template receives both credentials:

```text
<AWX_BASE_AWS_CREDENTIAL_NAME>
test-awx-aws-connect-assume-role
```

The custom `AWS Assume Role Profile` credential type must inject:

```yaml
env:
  AWS_CONFIG_FILE: "{{ tower.filename.aws_config }}"
  AWS_PROFILE: "{{ profile_name }}"
  AWS_SDK_LOAD_CONFIG: "1"
file:
  template.aws_config: |
    [profile {{ profile_name }}]
    role_arn = {{ role_arn }}
    source_profile = awx-base
    region = {{ region }}
```

If AWX is still using `arn:aws:iam::<account>:user/awx-automation` in S3 or SSM
errors, the job is not using the assume-role profile. Verify that the job
template is not using the built-in AWS credential as the base credential.

## AWS Permission Model

The AWX base credential belongs to the platform team and is backed by the
central AWS IAM user:

```text
awx-automation
```

That user can only call `sts:AssumeRole` into approved team roles. The actual
SSM, EC2, and S3 permissions belong on the team role:

```text
awx-ssm-automation-role
```

The role needs:

```text
ssm:StartSession
ssm:TerminateSession
ssm:DescribeSessions
ssm:GetConnectionStatus
ssm:SendCommand
ssm:GetCommandInvocation
ec2:DescribeInstances
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
```

The S3 permissions should be scoped to the SSM transfer bucket.

The current transfer bucket uses SSE-KMS. The SSM connection variables therefore
send explicit encryption headers during S3 uploads:

```yaml
ansible_aws_ssm_bucket_sse_mode: aws:kms
ansible_aws_ssm_bucket_sse_kms_key_id: "{{ awx_ssm_bucket_kms_key_id | default(lookup('env', 'AWX_SSM_BUCKET_KMS_KEY_ID') | default('arn:aws:kms:us-east-1:288821543174:key/457e2959-3208-4a57-90fc-ee1570bcdcb3', true), true) }}"
```

The assumed role must be allowed to use that KMS key for `kms:GenerateDataKey`,
`kms:Encrypt`, `kms:Decrypt`, and `kms:DescribeKey`.

## Pipeline Flow

On `main`, the shared pipeline performs:

```text
validate
  -> validate awx/team_vars.yml exists and is readable

configure
  -> launch central-awx-configure-resources
  -> create/update the AWX Project, AWS assume-role profile credential, and Job Template

sync
  -> sync the AWX Project so AWX pulls this repo's latest main branch

launch
  -> manually launch test-awx-aws-connect-ping and stream AWX output into GitLab

cleanup
  -> manually remove declared AWX resources when a change ticket is provided
```

The AWX project sync happens after the GitLab commit is already checked into
`main`, so AWX can pull the current repo content when the job template launches.

## Troubleshooting

### ImagePullBackOff

If `awx-launch-job` fails before Ansible output appears and AWX shows:

```text
Receptor detail: Error creating pod: container failed to start, ImagePullBackOff
```

the AWX Kubernetes runtime cannot pull the Execution Environment image:

```text
registry.midhtech.local:5050/infra_team/automation/ee-aws-ssm:v1.0.0
```

The required AWX-side setup is documented in the `ee-aws-ssm` README. In this
environment the important fixes were:

```text
Execution Environment: ee-aws-ssm
Registry credential: gitlab-container-registry
k3s registry mirror: http://registry.midhtech.local:5050
AWX VM hosts entry: 192.168.1.101 gitlab.midhtech.local registry.midhtech.local
```

The GitLab runner can pull the image independently of AWX. AWX jobs use k3s
containerd on the AWX VM, so AWX needs its own registry credential, registry
runtime config, and DNS resolution.

### SSH Timeout

If the job starts but fails with:

```text
Failed to connect to the host via ssh
```

Ansible did not load the SSM group variables. Confirm this file exists in the
AWX project branch:

```text
playbooks/group_vars/linux/ssm.yml
```

Then run the AWX project sync job before relaunching the AWX job template.

### SSM Target Validation Error

If the job fails with:

```text
Value at 'target' failed to satisfy constraint
```

Ansible is sending an IP address to AWS SSM instead of an EC2 instance ID. The
SSM group vars must include:

```yaml
ansible_aws_ssm_instance_id: "{{ instance_id }}"
```

The `instance_id` variable is written onto the AWX host by the Terraform AWX sync
job.

### Remote Tmp Directory Error

If SSM connects but Ansible fails with:

```text
Failed to create temporary directory
```

pin the Ansible module temp path under `/tmp`:

```yaml
ansible_remote_tmp: /tmp/.ansible/tmp
```

This avoids relying on the SSM session user's home directory expansion. The
current SSM group vars already set this value.

### Python Discovery Or Exec Timeout

If SSM connects but the job hangs or fails with:

```text
Unhandled error in Python interpreter discovery
SSM exec_command timeout
```

pin the remote shell and Python interpreter:

```yaml
ansible_shell_executable: /bin/sh
ansible_python_interpreter: /usr/bin/python3
```

This avoids Ansible's interpreter discovery over the SSM interactive shell. The
current SSM group vars already set these values.

For first-line SSM validation, prefer `raw` tasks before Python-backed modules.
`raw` validates the SSM command channel without requiring Ansible module
packaging and S3 transfer to complete first.

If `raw` works but `ansible.builtin.command`, `ansible.builtin.ping`, or other
normal modules hang, SSM is working but module transfer is not. The
`community.aws.aws_ssm` connection plugin stages module payloads through S3, so
the EC2 instance also needs private connectivity to the S3 transfer bucket.

For private subnets, use one of:

```text
S3 Gateway VPC endpoint attached to the private subnet route table
NAT path to S3
```

The S3 Gateway endpoint is the cleaner private-network option. If the transfer
bucket has a restrictive bucket policy, allow access through that endpoint and
the assumed role used by AWX.

### S3 PutObject Forbidden

If SSM connects but file transfer fails with:

```text
An error occurred (403) when calling the PutObject operation: Forbidden
```

check the transfer bucket encryption policy. Buckets that require SSE-KMS may
reject uploads unless the SSM connection plugin sends the SSE mode and KMS key
ID. The current group vars default to the dedicated `midh-dev-s3` transfer
bucket KMS key, and the value can be overridden with `AWX_SSM_BUCKET_KMS_KEY_ID`.
