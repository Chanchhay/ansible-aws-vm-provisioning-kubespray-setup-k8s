# AWS EC2 + Ansible + Kubespray lab

Provision up to three Ubuntu 24.04 control-plane instances and two workers in the AWS Sydney region using your account's default VPC and default subnets. This is a hands-on lab, not production-ready networking.

## Before you start

- Log in with `aws login --profile devops-lab` and verify with `aws sts get-caller-identity --profile devops-lab`. The role/project must permit EC2, VPC discovery, SSM GetParameter and key-pair operations. Do not paste credentials into YAML.
- `python3 -m pip install boto3 botocore` (in the Python environment used by Ansible)
- `ansible-galaxy collection install amazon.aws`
- Ensure you have `~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub`, or generate them with `ssh-keygen -t ed25519`.
- Confirm with `aws sts get-caller-identity --profile devops-lab`
- All 5 instances cost money or use AWS credit. Public IPv4 addresses and gp3 storage can cost extra.
- You need an intact default VPC and default subnets in ap-southeast-2.

## Provision

From the project directory:

```bash
MY_CIDR="$(curl -4 -fsS https://checkip.amazonaws.com | tr -d '\n')/32"

ansible-playbook -i localhost, -c local playbooks/provision.yaml \
  -e "ssh_allowed_cidr=$MY_CIDR" \
  -e confirm_provision=true
```

For a cheaper 1 master / 2 worker lab, add this argument:

```bash
-e '{"target_node_names":["master01","worker01","worker02"]}'
```

Note: running the smaller topology later will regenerate inventory.ini with only the nodes selected. It will not delete other already-running EC2 instances.

## Test

```bash
ansible -i inventory.ini all -m ping
```

Ubuntu's default cloud user is `ubuntu`, not your GCP `chanchhay` user. The playbook imports your existing public SSH key using an EC2 Key Pair.

## Clean up

```bash
ansible-playbook -i localhost, -c local playbooks/cleanup.yaml -e confirm_destroy=true
```

To delete only one selected worker, add `-e '{"target_node_names":["worker02"]}'`.

If all nodes are removed, add `-e remove_security_group=true -e remove_key_pair=true` to delete shared lab resources.

Deleting instances can remove their root EBS volumes. Make backups if necessary. The generated inventory.ini must be refreshed before reuse. No Elastic IPs are allocated by these playbooks. Stopped EC2 instances may retain EBS costs, and public IPv4 may be metered while attached.
