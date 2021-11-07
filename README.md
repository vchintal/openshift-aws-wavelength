# OpenShift on AWS Wavelength Automation

This repository allows you to get started installing OpenShift on AWS and on Wavelength subnets.

The automation will do the following:
1. Setup local folders and downloads the `most recent` binaries for OpenShift installation
2. Sets up AWS by creating the following:
   * VPC and associated resources _(using CloudFormation templates)_
   * Carrier Gateway and associated resources _(using CloudFormation templates)_
   * Public Hosted Zone
3. Prepares all the materials for OpenShift installation, this includes
   * Using the `install-config.yaml` to generate the install manifests
   * Create exactly two `MachineSets` of node/role type of `wavelength` set to run in Wavelength zones/subnets

This repo was possible due to the brilliant work of Ashish Aggarwal of Red Hat
* [Running an OpenShift Worker Node on AWS Wavelength for Edge Applications](https://cloud.redhat.com/blog/running-an-openshift-worker-node-on-aws-wavelength-for-edge-applications)
* [Building your first Red Hat OpenShift cluster on Verizon 5G Edge](https://verizon5gedgeblog.medium.com/building-your-first-red-hat-openshift-cluster-on-verizon-5g-edge-f9c2f07e8f20)
  
## Prerequisites

* Linux host  (Fedora 33+/RHEL 8) with the following installed 
  * Latest Ansible (2.9.x)
  * Git
* Access to the AWS environment
  * `aws configure` cli command has been run on the Linux host
  * The AWS credentials are stored under `~/.aws/credentials` file
  * At least two `Carrier IP` Elastic IPs allocated, one for every Wavelength zone, planned for installation 

## Install Preparation

### Git clone the repo 

```sh 
git clone https://github.com/vchintal/openshift-aws-wavelength
cd openshift-aws-wavelength
```
### Customization

1. Customize the `host_vars/localhost.yml` while following the embedded comments carefully
   * Ensure that the `pull_secret` that is updated is **NOT** surrounded in quotes
   * Ensure that each Wavelength zone you list has at least one `Carrier IP` Elastic IP allocated
2. Customize the `roles/aws/templates/install-config.yaml.j2` file as required 

### Run the setup playbook
```sh 
ansible-playbook ocp-aws-wavelength-setup.yml
```

Once the playbook has finished running, it has prepared all the materials in `install-dir` for OpenShift install

## Install OpenShift

### Run the OpenShift installation
```sh 
bin/openshift-install create cluster --dir=install-dir
```
> **Important**: When the installation is proceeding, you will notice three `worker` nodes join the cluster but thee two `wavelength` nodes still wouldn't show up with `oc` client despite the VMs created in EC2. At that point, you need to `associate` the `Carrier IP` Elastic IP to `Wavelength` EC2 instances for then to have access to the internet and eventually join the cluster.

## Post-Install 

For every `Deployment`/`DeploymentConfig` that need to run the workloads on the `wavelength` nodes/machines, update the configuration to add the following:

```sh
# Namespace Name: demo
# DeploymentConfig Name: mariadb
bin/oc patch dc -n demo mariadb -p '{"spec": {"template":{"spec": {"nodeSelector": {"node-role.kubernetes.io/wavelength": ""}}}}}'
```

Alternatively, you can create a project that will deploy all the workloads created in it to only the `wavelength` nodes with the following command:

```sh 
# Project Name: demo
bin/oc adm new-project --node-selector=node-role.kubernetes.io/wavelength='' demo
```