---
layout: post
title: "The Best Infrastructure Role Is Often the One That Deletes an Exception"
date: 2026-10-15 11:00:00 +0200
categories: systems ansible monitoring operations
excerpt: "A monitoring exception was correct for one cluster node. When a second node needed it, the exception had become a rule and the role needed to model it once."
---

A monitoring configuration on one node had accumulated a small exception for a cluster-managed runtime file. It was sensible at the time. The file changed frequently for operationally normal reasons, so treating every change as a security event created noise rather than signal.

Then a second node needed the same exception.

That was the point where the original line stopped being an exception. It had become a rule, disguised as a hostname.

## Problem

The first version of the configuration ignored one exact path. It worked because there was one relevant machine. The second machine had the same directory structure, but its node-specific path was different.

The tempting fix was to add another literal path. That would have solved the immediate problem, but it would also have created a list that needed updating every time the cluster changed.

The actual requirement was narrower:

- ignore a small set of known, operationally noisy cluster files;
- derive only the node-specific part from the managed host;
- remove the old literal exception instead of keeping both versions forever;
- avoid turning every later Ansible run into a configuration change and service restart.

## Possible approaches

### Keep adding host-specific lines

This is the smallest immediate change. It is also the beginning of a list that nobody wants to own. Every new node becomes another edit, another review, and another opportunity to miss one place where the rule was copied.

### Ignore an entire directory

This is operationally easy, but too broad. A directory that contains one noisy runtime file can also contain changes that are worth watching. Reducing alert noise by hiding unrelated changes is not a useful trade.

### Use a broad wildcard

A wildcard is better than repeating node names, but it still needs careful thought. If the cluster layout changes, a broad expression can silently cover more than intended. In monitoring configuration, silence is often harder to notice than a false positive.

### Model the invariant

The useful invariant was simple: every managed node has the same two shared noisy files and one node-local noisy file. Ansible already knows the current inventory hostname, so the role can derive the one variable segment from that fact.

That produces one narrow rule for every node rather than a growing historical list.

## Solution

The final role keeps the fixed paths explicit and derives only the node name:

{% raw %}
```yaml
monitoring_fim_ignores:
  - /etc/cluster/ha/commands
  - /etc/cluster/ha/status
  - "/etc/cluster/nodes/{{ inventory_hostname }}/runtime-status"
```
{% endraw %}

The role writes those paths as an Ansible-managed block in the monitoring agent configuration. The marker matters. It gives the role an ownership boundary: on the first run it removes the old literal line, then adds the managed block. On later runs it sees its own marker and leaves the managed entry alone.

That last detail is easy to miss. A cleanup expression that matches both the old line and the new generated line is technically correct once, but non-idempotent forever. The next run removes the managed line, adds it back, and restarts the agent for no operational reason.

A configuration-management role is not finished when it reaches the desired file once. It is finished when the second run is boring.

## Validation is part of the change

The role makes a temporary backup before editing the configuration. It restores that backup and stops if validation fails. This is deliberately unglamorous. A monitoring agent that keeps running with its old configuration is preferable to an agent that fails on its next restart because a generated file was invalid.

The validation is split by responsibility:

```yaml
- ansible.builtin.command: /var/ossec/bin/wazuh-agentd -t
  changed_when: false

- ansible.builtin.command: /var/ossec/bin/wazuh-syscheckd -t
  changed_when: false
```

The first command validates the agent-side client configuration. The second validates the file-integrity section that was changed. Wazuh documents both local configuration validation and the file-integrity daemon separately.[1][2]

The rollout also runs one managed node at a time. A package update or a handler restart should not take every monitoring agent down together simply because the inventory happened to contain two hosts.

Finally, the result was checked from both sides: the agents were running and connected after the rollout, and the manager reported both as active.

## Why this solution holds up

The role is still small. It does not introduce a generic cluster abstraction, a data model for every possible monitored path, or a wildcard broad enough to surprise us later.

It does encode the real rule:

1. the common paths are common;
2. the node-local path belongs to the current inventory host;
3. the historical exception is migrated once;
4. subsequent runs converge without a restart.

That is usually the threshold worth aiming for in infrastructure code. Generalise the part that has demonstrably repeated. Leave everything else alone.

## Limitation

This is intentionally specific to these known file-integrity exceptions. If the cluster gains another dynamically named path, it should be added as a reviewed rule with the same narrowness. The answer is not to turn the whole directory into an ignore pattern just to avoid writing one more explicit line.

## References

1. [Wazuh: verifying local configuration](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/verifying-configuration.html)
2. [Wazuh: `wazuh-syscheckd` reference](https://documentation.wazuh.com/current/user-manual/reference/daemons/wazuh-syscheckd.html)
