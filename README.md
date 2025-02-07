# EC2 Instance Connect Connection Plugin for Ansible
The EC2 Instance Connect (ECI) connection plugin was created to take advantage of AWS's <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html">ECI</a> capability Rather than rely on public keys statically stored on resources, this allows us to take advantage of using AWS native roles and permissions to access and manage linux servers instead. 

This is helpful in situations where you need to use continue to use ansible over AWS native instance management solutions, but want to take advantage of AWS's native IAM model for authorization as well as to avoid sharing of long living private keys.

Check [releases](https://github.com/mpieters3/ansible-eci-connector/releases) for versions of this library for older Ansible versions

## Installation into Ansible
Drop eci.py into a connection plugin location, as outlined in https://docs.ansible.com/ansible/latest/dev_guide/developing_locally.html. Must have boto3 and ec2instanceconnectcli python libraries available

AWS Servers must be set up to support <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-set-up.html">EC2 Instance Connect</a>. 

### Parameters
For parameter details, use ansible-doc -t connection eci

In general, aligned to the same requirements as most other <a href="https://docs.ansible.com/ansible/latest/modules/ec2_module.html">aws related modules and tasks in ansible</a>. Namely, in one way or another AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY must be set, and we must also have region. Generally, this is set either at the host level or globally.

TODO: 
The connection plugin can take either instance_id or use ip address (public or private) or hostname to determine the correct connection details. 

## Local Testing
We test the plugin by doing the following:
1. Create a security group (opening port 22 from 0.0.0.0/0)
2. Creates a t2.micro aws linux ami; doesn't set any keypair, so not accessible with 'normal' ssh
3. Connects using eci with instance-id & ip address (preferred) information as root, echo basic message
4. Connects using eci with ip address host information as ec2-user, echo basic message


### Env Setup
Using WSL2 or a Linux instance, setup a new python venv
```
python3 -m venv venv/
source venv/bin/activate
pip install requirements.txt
ansible-galaxy collection install amazon.aws
```

Make sure the plugin is being pulled in correctly... from the workspace directory, run the following command to make sure you're getting the connection info:
env ANSIBLE_CONNECTION_PLUGINS=./plugins/connection ansible-doc -t connection eci

### Inventory Example
```
[Machines]
mytest1 ansible_host=<ip address> aws_region=ap-northeast-1 ansible_connection=eci

# use aws profile. Instance should be placed in same region with aws profile
mytest2 ansible_host=<ip address> profile=dev ansible_connection=eci

# assign instance id directly, will run faster
mytest1 ansible_host=<ip address> aws_region=ap-northeast-1 profile=dev ansible_connection=eci eci_instance_id=i-1234567890

# Other cloud provider
mytest2 ansible_host=<ip address> ansible_connection=eci eci_platform=alicloud eci_region=cn-hongkong
mytest2 ansible_host=<ip address> ansible_connection=eci eci_platform=alicloud eci_region=cn-hongkong eci_instance_id=i-abc1234567890
```

### Running playbook
Continuing in your venv, set the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY for the account the test should run in, then run the test
```
export AWS_ACCESS_KEY_ID='<<YOUR_ACCESS_KEY_ID>>'
export AWS_SECRET_ACCESS_KEY='<<YOUR_SECRET_ACCESS_KEY>>'
cd /workspaces/ansible-eci-connector/test
env ANSIBLE_CONNECTION_PLUGINS=../plugins/connection ansible-playbook -vv demo.yml
```

## Why not MSSH? 
While <a href="https://github.com/mingbowan/mssh/blob/master/mssh.py">mssh</a> may be an option as well, it was important to ensure better support for everything the original SSH provider has

## (Optional) Support Alicloud
> x86 machine only because Ali-instance-cli cannot run at Arm env.

To support alicloud session manager (Like AWS ECI), do the following steps:

- install Alicloud SDK
```
pip3 install --upgrade setuptools wheel twine check-wheel-contents packaging==22
pip3 install aliyun-python-sdk-core aliyun-python-sdk-ecs
```
- install Alicloud Instance Cli tool [official doc](https://www.alibabacloud.com/help/en/ecs/user-guide/register-a-public-key-and-connect-to-an-instance-with-the-key-by-using-ali-instance-cli)
```
curl -O https://aliyun-client-assist.oss-accelerate.aliyuncs.com/session-manager/linux/ali-instance-cli
chmod a+x ali-instance-cli
mv ali-instance-cli /usr/local/bin
```
- Setup Credentials
  - use environment variable
    - ALICLOUD_ACCESS_KEY_ID
    - ALICLOUD_ACCESS_KEY_ID
  - Prepared alicloud profile

## TODO
- IP Address to instance id (or vice versa?) lookup
- remove temp keys when run finishes
- Look at incorporating into or deprecating in favor of [ansible-collections/community.aws](https://github.com/ansible-collections/community.aws)
- - The S3 bucket does add additional complexity that this avoids...
