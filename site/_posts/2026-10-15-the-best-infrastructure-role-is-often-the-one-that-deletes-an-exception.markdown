---
layout: post
title: "The Best Infrastructure Role Is Often the One That Deletes an Exception"
date: 2026-10-15 11:00:00 +0200
categories: systems ansible monitoring operations
excerpt: "A Wazuh ignore rule was correct for one Proxmox node. When a second node needed it, the exception had become a rule that the Ansible role needed to express once."
---

Wazuh was reporting file-integrity changes for one Proxmox cluster status file during normal operation. The file was expected to change, so I had ignored that exact path on the first node.

Then the second Proxmox node started producing the same alerts.

The fix was not to add another ignored path. The role had encoded one hostname where it should have encoded the rule that produces the path.

## Problem

The Wazuh agent watches configured files and directories for changes. One cluster runtime-status file changes whenever the cluster does normal work. That made it a bad security signal: the alert was technically correct, but operationally useless.

The original Wazuh configuration contained one literal ignore entry:

```text
/etc/pve/nodes/pve-a/lrm_status
```

That worked while the cluster had one managed node. A second node used the same layout, but with a different node name:

```text
/etc/pve/nodes/pve-b/lrm_status
```

Adding another literal line would have fixed the new alert. It would also have created a list that needed manual edits every time the cluster changed.

The actual requirement was narrower:

- ignore three known, noisy cluster files;
- derive only the node-specific part from the managed host;
- migrate the old literal exception once;
- leave later Ansible runs unchanged, without restarting Wazuh unnecessarily.

## Possible approaches

### Add one more host-specific line

This is the smallest immediate change. It is also the start of a list that nobody wants to maintain. Every new node becomes another edit and another chance to forget the matching monitoring rule.

### Ignore the whole node directory

This would be easy, but too broad. The directory can contain changes worth investigating. Reducing alert noise by hiding unrelated changes is not a useful trade.

### Use a wildcard

A wildcard avoids repeating node names, but it can silently cover more than intended when the cluster layout changes. In monitoring configuration, an overly broad ignore is often worse than an extra alert.

### Model the rule once

Every managed node has the same two shared noisy files and one node-local file. In this inventory, `inventory_hostname` is deliberately the same name Proxmox uses below `/etc/pve/nodes/`. The role can therefore construct the one variable path from the host it is configuring.

## Solution

The role keeps the fixed paths explicit and derives only the node name:

{% raw %}
```yaml
wazuh_agent_proxmox_fim_ignores:
  - /etc/pve/ha/crm_commands
  - /etc/pve/ha/manager_status
  - "/etc/pve/nodes/{{ inventory_hostname }}/lrm_status"
```
{% endraw %}

The role writes those entries with Ansible's `blockinfile`, between stable markers in the Wazuh configuration. Those markers make the ownership boundary explicit.

The migration happens only before that managed block exists:

1. remove the old host-specific `lrm_status` line;
2. insert the managed block with the inventory-derived path;
3. skip the cleanup on every later run.

That guard matters. The first version of the cleanup matched any `lrm_status` ignore line. After the managed block existed, it also matched the line that Ansible had just generated. Every subsequent run removed the managed line, added it back, and notified the Wazuh restart handler.

The resulting configuration was correct, but the role was not idempotent. A converged host still restarted its monitoring agent on every run.

A role is not finished when it reaches the desired file once. It is finished when the second run is boring.

## Validate the change before restarting the agent

The rollout uses a small, explicit sequence:

1. copy the current Wazuh configuration to a temporary backup;
2. write the client and file-integrity changes;
3. validate the Wazuh agent configuration;
4. validate the file-integrity configuration;
5. restore the backup and stop if either validation fails;
6. restart one agent, then continue to the next host.

The two checks are separate because the role changes two separate concerns:

```yaml
- ansible.builtin.command: /var/ossec/bin/wazuh-agentd -t
  changed_when: false

- ansible.builtin.command: /var/ossec/bin/wazuh-syscheckd -t
  changed_when: false
```

`wazuh-agentd -t` checks the agent-side client configuration. `wazuh-syscheckd -t` checks the file-integrity section that contains the new ignore rules.[1][2]

The playbook also uses `serial: 1`. A package action or handler restart affects one agent at a time, not every monitoring agent in the inventory together.

## Verification

The first node exercised the migration path: the old literal ignore was removed, the managed block was inserted, both validation commands passed, and the agent restarted successfully.

The second node exercised the fresh-host path: the agent package was installed, enrolled, validated, and started. Afterwards both agents reported as connected, and the Wazuh manager listed both as `Active`.

That checks more than whether the YAML parsed. It verifies the generated configuration, the running agent, and the manager-side view of the agent.

## Why this solution holds up

The role owns three known paths and nothing broader:

1. two shared cluster files;
2. one node-local runtime-status file derived from the configured host;
3. one migration path from the historical literal exception.

It does not ignore an entire directory. It does not need a hostname list. It does not restart a healthy agent after every converged run.

## Limitation

This is intentionally specific to the current, documented path pattern. If the cluster gains more dynamic paths than these three known exceptions, the next step is an explicit reviewed data structure for those paths. It is not a wildcard for the entire directory.

## References

1. [Wazuh: verifying local configuration](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/verifying-configuration.html)
2. [Wazuh: `wazuh-syscheckd` reference](https://documentation.wazuh.com/current/user-manual/reference/daemons/wazuh-syscheckd.html)
