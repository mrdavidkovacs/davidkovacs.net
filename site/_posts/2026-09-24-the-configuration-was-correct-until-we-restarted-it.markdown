---
layout: post
title: "The Configuration Was Correct—Until We Restarted It"
date: 2026-09-24 10:00:00 +0200
categories: systems reliability ansible docker configuration
excerpt: "A service worked because its running configuration had been repaired manually. The next deployment recreated it from a different source of truth."
---

A family dashboard could not create its first family. The browser sent a request to the application, the application called its local API gateway, and the request failed with a server error.

The database and application containers were healthy. The failure was in the gateway configuration.

The immediate repair was small. The gateway had two stale API keys while the application used the current keys. Updating the gateway configuration and recreating only that container restored the request path.

That looked like the end of the incident. It was not.

## The configuration that was actually running

The dashboard uses Docker Compose. An Ansible deployment role prepares its stack directory from repository files and encrypted Vault values, then Docker Compose recreates containers from those files. Its API gateway reads a declarative configuration file from a bind mount. That file contains Key Auth credentials for the dashboard service and anonymous client.

The running gateway had been repaired with the current credentials. A request through the gateway returned `200`. The same request had previously been rejected with `401 Unauthorized`.

This established that the repair worked. It did not establish that the repair would survive a deployment.

The next configuration review found a different version of the gateway file in the infrastructure repository. It still contained static placeholder values. The deployment process copied that file into the stack directory before Docker Compose recreated containers.

The system therefore had two truths:

- the running container used repaired credentials;
- the declared configuration would restore placeholders on the next deployment.

A healthy container was hiding an unhealthy deployment state.

## Why restart testing matters

A restart is often treated as a basic availability check. Here it was a configuration consistency check.

The relevant failure sequence was straightforward:

1. A manual repair changed the bind-mounted gateway configuration.
2. The application worked because the running gateway used that repaired file.
3. An infrastructure deployment copied the repository version back into the stack directory.
4. Recreating the gateway made it read the repository version again.
5. Requests would fail with `401`, and the application would report a less useful `500` to the browser.

The original error was not caused by Docker Compose, Kong, or the database. It was caused by a mismatch between a corrected runtime file and an outdated declared file.

This is why "works now" is weaker evidence than "works after recreation from declared configuration." The second test exercises the configuration source, the deployment process, and the container startup path together.

## Render credentials where they are deployed

The API keys were already managed as encrypted Ansible Vault variables. The missing step was using those variables when generating the gateway file.

The static file became a Jinja template. `stack_secret_files` is the deployment role's list of rendered secret files; it renders the final `kong.yml` as a protected stack file instead of copying a file with placeholders:

{% raw %}
```yaml
stack_secret_files:
  - path: kong.yml
    mode: "0600"
    content: "{{ lookup('ansible.builtin.template', playbook_dir + '/../stacks/' + stack_name + '/files/kong.yml.j2') }}"
```

The template uses the same vault-backed values as the application environment:

```yaml
keyauth_credentials:
  - consumer: dashboard
    key: "{{ vault_dashboard_service_role_key }}"
  - consumer: anon
    key: "{{ vault_dashboard_anon_key }}"
```
{% endraw %}

The point is not that Jinja is special. The point is that there is one declared source for each credential, and the configuration consumed by the container is generated from it during deployment.

The generated gateway file is deliberately treated as a secret file. It contains credentials, even though it is configuration rather than an `.env` file.

## The second configuration regression

The same deployment exposed another version of the problem. A weather API key had been added manually to the live `.env` file. The Ansible stack definition did not contain it.

The application worked until the next deployment regenerated `.env` from `stack_env`. Then the manually added key disappeared.

The fix was the same pattern:

{% raw %}
```yaml
stack_env:
  OPENWEATHERMAP_API_KEY: "{{ vault_dashboard_openweathermap_api_key }}"
```
{% endraw %}

A generated environment file should not also be a hand-maintained configuration store. If a value needs to survive deployment, it belongs in the declared input used to generate that file.

## Verification

The change was checked at three levels before deployment:

1. The encrypted vault was checked structurally. The new key existed and was non-empty; no existing Kinboard values changed.
2. Ansible rendered the stack variables and the gateway template using the real encrypted vault.
3. Kong parsed the rendered declarative configuration successfully in the same Kong version used by the stack.

The deployment verification remains important: recreate the gateway from the rendered stack directory, then make an authenticated request through it. The negative case also matters. A gateway configured with the old placeholder key must reject the same request.

## Result and limitation

The dashboard now has a declared path from encrypted variables to both the application environment and the gateway configuration. A future deployment no longer depends on a previous manual repair remaining on disk.

This does not make every configuration change safe automatically. A template can still render an incorrect value, and a valid Kong file can still contain the wrong policy. The useful boundary is narrower: a restart now tests the configuration that deployment will actually produce, rather than preserving an accidental state from the last manual intervention.
