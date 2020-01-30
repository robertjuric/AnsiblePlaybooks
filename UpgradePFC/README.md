# UpgradePFC Playbook

A playbook for running various checks before attempting to transfer an image for upgrade. I wanted to take a complete inventory and generate a new inventory of devices which are ready for an image transfer. The exact tasks and checks are described below. 

## Workflow
Running UpgradePFC.yml  
Will gatherfacts and compare running version to desired version, if they do not match, needs_upgrade=true  
Will check filesystem, if desired version image does not exist, needs_image=true  
If needs_upgrade=true and needs_image=true, it will check for freespace.  
Will generate a Needs-Freespace.txt remediation list for the inventory that requires freespace to be allocated.  
Manually remediate list and rerun the playbook  
If needs_upgrade=true and needs_image=true, and freespace exists then upgrade_ready=true  
Will generate an UpgradeInventory.yml  
Need to add a step to generate an InstallInventory.yml to pass to UpgradeCopy.yml  
  
Can then skip over to EnableSCP.yml to push SCP configuration to UpgradeInventory  
If you want to assume devices need image, Can then skip over to UpgradeCopy.yml to transfer images  
Then execute InstallOS.yml to set boot variables  
Execute MakeItSo to reboot device and complete IOS install  


## Getting Started

This playbook has a few extra requirements you may need to add to your Ansible environment.

### Prerequisites

This playbook utilizes the `ios_facts` module native to Ansible along with `ntc_show_command` from the [ntc-ansible](https://github.com/networktocode/ntc-ansible) modules.

### Initial Setup

This playbook was originally written for a subset of Cisco IOS devices and the Jinga template and host groups reflect that. If you want to adapt this playbook for your use, you must first create groups for each unique image file and ensure the groups have the following variables:

```
desired_version: 15.0(2)SE10a
desired_image: c2960-lanbasek9-mz.150-2.SE10a.bin
desired_image_size: 11831953
```
I created my host file in YAML format and assign `group_vars` by directory. I've included an empty `hosts` file and `group_vars` directory for reference.

You would also need to update the `UpgradeInventory.j2` template file to create groups in the new inventory matching your groups.

## Running the playbook

Execute the playbook with Ansible:

`ansible-playbook UpgradePFC.yml`

The playbook is configured to prompt for username and password instead of stored credentials.

The playbook will output a `needs_freespace.txt` file if needed and will also generate a `UpgradeInventory.yml` inventory file.

**You must remove empty host groups from the inventory file manually!**

## Playbook Task Details

### Defining Nodes for Upgrade

The first task is to use `ios_facts` to gather facts from the inventory. We then compare the `ansible_net_version` to the host_var `desired_version`. 

If they do not match then we set a fact `needs_upgrade: true`.

### Checking the File System

The next step is to get the file system details. This is only run on the devices which actually need an ugprade to help speed things up. I used `ntc_show_command` module because it includes a TextFSM template for the `dir` output. The output of this module is registered to the `dir_output` variable.

The first thing I wanted to check for was whether or not the `desired_image` was already on the device. It could be possible that it had already been staged and I wanted to catch that if it was already there. If `desired_image` was not in the `dir_output` then I set a fact `needs_image: true`.

The next thing I wanted to check for was whether or not there was enough free space on the filesystem to accept the image. If the freespace from `dir_output` is less than the `desired_image_size` then we dump those devices to a `needs_freespace.txt` file for manual remediation. I plan on creating a separate playbook to clean up a file system at a later time.

### Ready for Upgrade

If `needs_upgrade: true` and `needs_image: true` and freespace is larger than `desired_image_size` then I deem a device ready to receive the upgrade image. I set a fact `upgrade_ready: true`.

### Generate New Inventory

I wanted to create a separate inventory file to run for the next playbook which should transfer the image to the device. I used the `template` module and a Jinga2 template to create a YAML inventory file. The `UpgradeInventory.j2` template loops through all devices in each group and adds them to the inventory if `upgrade_ready: true`. 

The way the template works, if it loops through the devices for a group and doesn't find any ready you will have a host group without any hosts. That inventory is invalid, so **all empty groups must be manually removed before using the generated inventory file**.

## Authors

* **Robert Juric** - *Initial work* 

