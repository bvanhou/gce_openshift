# GCE OpenShift Deployment Automation
This repository contains playbooks used to automatically deploy and configure OpenShift clusters into AWS

###### !IMPORTANT: Make sure all variables are set and configured as needed before playbooks are run.

# Architecture
TODO: Architecture Diagram

Prerequisites
General
Workstation with the following installed
ansible
python-boto
python-boto3
python-botocore
AWS CLI tool
Satellite/Yum repository with appropriate RPMS
AWS
Appropriate AWS resource limits set
IAM user with appropriate permissions
Private VPC with connectivity to on-premise environment
Private Subnet
Access to AWS API Servers with proxy if needed
Configure environment
# Install ansible
`[user@workstation aws_devops]$ sudo yum -y install ansible python-boto python-boto3 python-botocore`

# Install aws command and API support
`[user@workstation aws_devops]$ ./scripts/aws-setup.sh`

# Reload Bash profile
`[user@workstation aws_devops]$ . $HOME/.bash_profile`

# Run AWS configuration command
```
[user@workstation aws_devops]$ aws configure
AWS Access Key ID [****************ABCD]:
AWS Secret Access Key [****************1234]:
Default region name [us-east-1]:
Default output format [None]:
```

# And test AWS configuration - Expect lots of output
`[user@workstation aws_devops]$ aws ec2 describe-instances --output text`
Create Internal AWS Network Load Balancers
If needed, network load balancers and their associated DNS entries should be created prior to the install

The nlb-create.sh script can be used to create pairs of NLB's to be used by OpenShift. nlb-create.sh takes the following parameters

name - short text string to identify NLB
subnet-id - Subnet to contain the NLB
sequence start
number of NLB pairs to create
# Create two pairs of NLB's in subnet-99999999
`[user@workstation aws_devops]$ ./scripts/nlb-create.sh example subnet-999999 1 2`

Creating 1 pair(s) of NLB's in subnet 10.1.1.18.0/24
Do you want to continue (Ctrl-C to abort)?
Creating example-nlb-9-be93-api
Created example-nlb-9-be93-api => example-nlb-9-be93-api-c744cf87a6b40d0b.elb.us-east-1.amazonaws.com
Creating example-nlb-9-be93-app
Created example-nlb-9-be93-app => example-nlb-9-be93-app-5c488c5cd32128ea.elb.us-east-1.amazonaws.com

# Existing NLB's can be listed using the nlb-list.sh command
`[ecsaws@sd-9277-023a aws_devops]$ ./scripts/nlb-list.sh`

us-east-1a example-nlb-9-be93-api 10.1.1.219 example-nlb-9-be93-api-c744cf87a6b40d0b.elb.us-east-1.amazonaws.com
us-east-1a example-nlb-9-be93-app 10.1.1.194 example-nlb-9-be93-app-5c488c5cd32128ea.elb.us-east-1.amazonaws.com
us-east-1a sandbox-nlb-1-b6cf-api 10.1.1.134 sandbox-nlb-1-b6cf-api-d877e2b487245964.elb.us-east-1.amazonaws.com
us-east-1a sandbox-nlb-1-b6cf-app 10.1.1.158 sandbox-nlb-1-b6cf-app-f2b9675aac6f172f.elb.us-east-1.amazonaws.com

Columns are:
Availability zone
NLB Name - Names ending in api are for masters, Names ending in app are for applications
IP address
AWS FQDN
Create DNS Entries
Each OpenShift cluster requires three DNS entries. These are

Public console/API endpoint - Uses api IP address - example: cluster01-webapi.aws.example.com
Private console/API endpoint - Uses api IP address - example: cluster01-internal.aws.example.com
Wild card entry for applications - Use app IP address - example: cluster01.aws.example.com
Note the installer creates a temporary DNS server on the Bastion host to allow an installation before the DNS entries are created

Configure deployment specific parameters
# Run the post-clone script
`[user@workstation aws_devops]$ ./scripts/post-clone.sh`

# Create custom config file and enter parameters or clone an existing config from source control
# All playbooks will accept this file as a parameter via --extra-vars
`[user@workstation aws_devops]$ cp ansible/example/config.yml /path/to/config.yml`
Edit ansible/variables/config.yml.
Note: The KubernetesClusters value defines a cluster and must be unique.

Run Playbooks
# Change to ansible directory
`[user@workstation aws_devops]$ cd ansible`

# Create and configure local and AWS environment
`[user@workstation ansible]$ ansible-playbook -v playbooks/aws_install.yml --extra-vars "@/path/to/config.yml" | tee /tmp/aws-install.log`

# Configure created AWS instances
`[user@workstation ansible]$ ansible-playbook -v playbooks/pre_install.yml --extra-vars "@/path/to/config.yml" | tee /tmp/pre-install.log`

# Install OpenShift - Installation output can be viewed in /tmp/ose-install.log
`[user@workstation ansible]$ ansible-playbook -v playbooks/ose_install.yml --extra-vars "@/path/to/config.yml" | tee /tmp/ose-install.log`
Post installation validation
The following playbook can be used to verify if an installation has completed successfully. If no errors are raised, the installation has passed the verification

`[user@workstation ansible]$ ansible-playbook playbooks/diagnostics.yml --extra-vars "@/path/to/config.yml" | tee /tmp/diagnostics.log`
Teardown
An environment can be torn down by running ansible/playbooks/maintenance/teardown.yml

###### !IMPORTANT: Be careful to ensure you are configured for the appropriate cluster!

`[user@workstation ansible]$ ansible-playbook playbooks/maintenance/teardown.yml --extra-vars "@/path/to/config.yml" | tee /tmp/teardown.log`
