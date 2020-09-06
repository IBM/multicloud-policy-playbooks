# multicloud-policy-playbooks
Publish policy playbooks for policy operators to consume.

## Introduction
**Ansible Operator to invoke Compliance job for VMs in Ansible Tower Inventory**

Note: The SRE Search API currently does not retrieve the Ansible Tower inventory. This is in plan for 3Q 2020. Temporarily, we search the Inventory directly in Ansible Tower. The Evidence API also requires the full inventory, however it does not directly connect to Ansible Tower. The VM Operator retrieves the full inventory from Tower for VMs in groups named OS:rhel7 and pushes the inventory to evidence API at the start of every run. This step will not be required when we move over to using the SRE search API. It will instead be replaced with a request to find the applicable VMs from the SRE search.

Ansible Tower can have multiple inventories. Each inventory can have multiple Groups. Virtual Machines in Ansible Tower belong to one or more of these groups. When the Policy applies to multiple VMs using the vmSelector, the VM must have all specified tags (treated as "and" of the multiple tags) as below:
```
        vmSelector:
          tags:
            env: prod
            OS: rhel7
```
Ansible Tower does not directly support tags. Instead we use group names such as "OS:rhel7", "env:prod", "env:dev", "env:test" and search for group names that the VMs belong to. We add the extra environment variable in Settings->Jobs ```"ANSIBLE_TRANSFORM_INVALID_GROUP_CHARS": "ignore"``` because Tower does not like the colon character in the group names. This variable is to prevent the deprecation Warning message in the job logs.
```
[DEPRECATION WARNING]: The TRANSFORM_INVALID_GROUP_CHARS settings is set to allow bad characters in group names by default, this will change, but still be user configurable on
deprecation. This feature will be removed in version 2.10. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.
 [WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
```

Groups are not shared across inventories, instead groups belong to an inventory. Thus the group names will appear multiple times, once for each inventory. i.e. "OS:rhel7" group will be created in each inventory. Same for other group names.
For the above specified vmSelector, all accessible inventories that have group names "env:prod" and "OS:rhel7" (case insensitive) are searched and the VMs belonging to both those groups are selected. Thus, the multiple same named groups across inventories are treated as a single tag. Nested groups are not searched.
The evidence API uses case sensititive search, so please use the exact case sensity name.

When a job template is created, the credentials are not attached to the job template. Instead the REST API call to Tower provides the credential ids when the Job template is invoked. The inventory variables, group variables and machine variables are used for finding the credential used for each VM. The credentials are stored in encrypted form within Ansible Tower Credentials. The VMs are associated with the corresponding credentials to use for running the job. This is done with the variables os_credential and the jumphost_credential. The Ansible Operator first attempts to find these variable values from Machine variables. If any are absent, then it searches for the first Group that can provide the variable. If missing, then it searches the Inventory variables. The Ansible Operator finds the inventory id from the inventory name, the credential ids from the credential variables, the job template id from the job template name in the Ansible Tower Secret and invokes the Job. The os_credential must be provided. The jumphost_credential does not need to be set if the Machine is directly accessible from Ansible Tower. The jumphost credential is a Custom Credential that allows you to jump across multiple jumphosts to connect to the endpoint. Additional instructions for configuring Ansible Tower with Cascading Jumphosts is provided at https://developer.ibm.com/recipes/tutorials/multiple-jumphosts-in-ansible-tower-part-1/

