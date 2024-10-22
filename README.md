# Ansible Automation Controller (or AWX), powered by Ansible
This role provides a means to configure AAP Controller/AWX and make it idempotent.

# Requirements
This role requires the following in order to work

  * 'awx.awx' 23.2.0 or higher
  * A Red Hat subscription manifest with AAP licenses
  * Administrative access to Automation Platform Controller

Tested with Ansible 2.16 and higher

This role supports Red Hat Automation Controller 4.5.2 and up. Has not been verified against AWX.

# Scheduling jobs
Apart from configuring most AAP aspects rather straigthforward, this role supports scheduling
jobs and workflows with simple schedules (such as every X minutes/hours/etc.). Please see
defaults/main.yml for more information.

# Workarounds and/or issues
For some quirks in AAP you'll need a workaround, this sections describes the ones I needed:

## Constructed inventories do not import group_vars for created groups
If your Constructed Inventory creates new groups, for example 'database_servers', the group_vars file
is not imported from the Project directory when the inventory updates.

You can import the variables with a 'normal' inventory which has a source from the Project like this:

```
# This group is defined to import it's values, hosts are added in the Constructed Inventory
[database_servers]
```

## Constructed inventories updates do not include dependant input inventories
When updating a Constructed Inventory, it's Input Inventories are not updated with it.

In order to update them, I have added the following workflow to my configuration and scheduled
it to run every 5 minutes. This workflow will update each inventory source and the automatically
created source for the constructed inventory

```
  - name: 'Refresh inventory'
    organization: 'Example'
    schedules:
      - type: 'minute'
        frequency: '5'
        description: 'Repeat every 5 minutes'
    workflow_nodes:
      - identifier: 'update-hosts'
        unified_job_template:
          name: '01_infrastructure'
          inventory:
            organization:
              name: 'Example'
          type: 'inventory_source'
        related:
          success_nodes:
            - identifier: 'update-constructed'
      - identifier: 'update-satellite'
        unified_job_template:
          name: '02_foreman'
          inventory:
            organization:
              name: 'Example'
          type: 'inventory_source'
        related:
          success_nodes:
            - identifier: 'update-constructed'
      - identifier: 'update-constructed'
        unified_job_template:
          name: 'Auto-created source for: Default'
          inventory:
            organization:
              name: 'Example'
          type: 'inventory_source'
```
