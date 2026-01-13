# RHADS Validation Role - Development Reference

## Overview

The `rhads_validation` role performs comprehensive health checks on a Red Hat Advanced Developer Strategies (RHADS) environment deployed on OpenShift. It validates all major components and generates a detailed report saved via `agnosticd_user_info` module.

## Role Structure

```
roles/rhads_validation/
├── defaults/main.yml           # Configuration and toggles
├── tasks/
│   ├── main.yml               # Orchestrator
│   ├── check_operators.yml    # Loop through operators
│   ├── check_operator_single.yml  # Validate individual operator
│   ├── check_argocd.yml       # Loop through ArgoCD instances
│   ├── check_argocd_instance.yml  # Validate individual ArgoCD
│   ├── check_vault.yml        # Vault health
│   ├── check_keycloak.yml     # Keycloak in tssc-keycloak or keycloak ns
│   ├── check_developer_hub.yml    # RHDH/Backstage
│   ├── check_gitlab.yml       # GitLab
│   ├── check_devspaces.yml    # OpenShift DevSpaces
│   ├── check_quay.yml         # Quay registry
│   ├── check_rhacs.yml        # Red Hat Advanced Cluster Security
│   ├── check_pipelines.yml    # OpenShift Pipelines
│   ├── check_showroom.yml     # Loop through showroom users
│   ├── check_showroom_user.yml    # Validate individual showroom instance
│   └── check_routes.yml       # Collect all routes
├── library/
│   └── agnosticd_user_info.py # Module for saving user data (also in plugins/)
├── action_plugins/
│   └── agnosticd_user_info.py # Action plugin for agnosticd_user_info
└── templates/                 # (empty for now)
```

## Components Validated

| Component | Namespace(s) | What's Checked |
|-----------|-------------|----------------|
| **Operators** | Various | CSV phase = Succeeded |
| **ArgoCD** | tssc-gitops, openshift-gitops | Server deployment, route, HTTP accessibility |
| **Vault** | vault | Deployment, route, HTTP accessibility |
| **Keycloak** | tssc-keycloak, keycloak | Running pods in either namespace, routes |
| **Developer Hub** | backstage | Deployment, PostgreSQL pod, route, HTTP accessibility |
| **GitLab** | gitlab | Deployment, route, HTTP accessibility |
| **DevSpaces** | openshift-devspaces | Operator installed |
| **Quay** | quay-enterprise | QuayRegistry CR, pods, route, HTTP accessibility |
| **RHACS** | stackrox | Central deployment, route, HTTP accessibility |
| **Pipelines** | openshift-pipelines | Operator installed |
| **Showroom** | showroom-{guid}-{pool}-user{n} | Deployment, pods, route, HTTP accessibility (per user) |
| **Routes** | All validated namespaces | Route inventory |

## Required Variables

When calling the role, you must provide:

```yaml
rhpds_build_secured_dev_workflows_rhads_validation_guid: "{{ guid }}"
rhpds_build_secured_dev_workflows_rhads_validation_pool_number: "{{ pool_number }}"
rhpds_build_secured_dev_workflows_rhads_validation_username_base: "user"  # or custom base
rhpds_build_secured_dev_workflows_rhads_validation_num_users: "{{ num_users }}"
```

### Example Playbook

```yaml
---
- name: Validate RHADS Environment
  hosts: localhost
  gather_facts: false

  vars:
    guid: "qqf9f"
    pool_number: "1"
    num_users: 1
    rhpds_build_secured_dev_workflows_trusted_artifact_signer_username_base: "user"

  tasks:
    - name: Run RHADS validation
      ansible.builtin.include_role:
        name: rhpds.build_secured_dev_workflows.rhads_validation
      vars:
        rhpds_build_secured_dev_workflows_rhads_validation_guid: "{{ guid }}"
        rhpds_build_secured_dev_workflows_rhads_validation_pool_number: "{{ pool_number }}"
        rhpds_build_secured_dev_workflows_rhads_validation_username_base: "{{ rhpds_build_secured_dev_workflows_trusted_artifact_signer_username_base }}"
        rhpds_build_secured_dev_workflows_rhads_validation_num_users: "{{ num_users }}"
```

## Key Design Decisions & Fixes

### 1. Operator Validation - CSV Phase Check

**Problem**: Operators showed as "missing" even when installed because pod label selectors didn't match OLM-managed pods.

**Solution**: Changed to check ClusterServiceVersion (CSV) phase instead:
```yaml
- name: Get operator CSV in namespace
  kubernetes.core.k8s_info:
    api_version: operators.coreos.com/v1alpha1
    kind: ClusterServiceVersion
    namespace: "{{ operator.namespace }}"
  register: r_operator_csv

- name: Evaluate operator health
  ansible.builtin.set_fact:
    operator_status: |-
      {%- if operator_csv is not defined or operator_csv == {} -%}
      missing
      {%- elif operator_csv.status.phase | default('') != 'Succeeded' -%}
      failed
      {%- elif operator_csv.status.phase | default('') == 'Succeeded' -%}
      healthy
      {%- else -%}
      unknown
      {%- endif -%}
```

**Operator Namespaces** (from defaults/main.yml):
- `openshift-operators`: Most operators (AllNamespaces mode)
  - devspacesoperator
  - openshift-gitops-operator
  - openshift-pipelines-operator-rh
  - quay-operator
  - rhacs-operator
- `tssc-tas`: rhtas-operator
- `tssc-tpa`: rhtpa-operator
- `tssc-keycloak`: rhbk-operator (Red Hat build of Keycloak)

### 2. Showroom Namespace Pattern

**Problem**: Initial implementation used `showroom-{{ guid }}-user{{ n }}` but actual pattern includes pool number.

**Actual Pattern**: `showroom-{{ guid }}-{{ pool_number }}-user{{ n }}`

**Example**: `showroom-qqf9f-1-user1`

**Solution**: Added pool_number to namespace construction and deployment name check:
```yaml
_showroom_namespace: "showroom-{{ rhpds_build_secured_dev_workflows_rhads_validation_guid }}-{{ rhpds_build_secured_dev_workflows_rhads_validation_pool_number }}-{{ rhpds_build_secured_dev_workflows_rhads_validation_username_base }}{{ showroom_user_index + 1 }}"

# Check for deployment named "showroom"
- name: Get Showroom deployment
  kubernetes.core.k8s_info:
    api_version: apps/v1
    kind: Deployment
    name: showroom
    namespace: "{{ _showroom_namespace }}"
```

### 3. ArgoCD Sync Status

**Problem**: ArgoCD marked as "degraded" when applications were OutOfSync.

**Solution**: Removed sync status check - applications can be OutOfSync but still healthy:
```yaml
# OLD - Too strict
{%- elif r_argocd_apps.resources | selectattr('status.sync.status', 'equalto', 'OutOfSync') | list | length > 0 -%}
out-of-sync

# NEW - Only check server availability and route accessibility
{%- elif r_argocd_http_check is defined and not r_argocd_http_check.skipped | default(false) and r_argocd_http_check.failed -%}
not-accessible
{%- else -%}
healthy
```

### 4. Keycloak Validation Logic

**Problem**: Keycloak marked as unhealthy when only one namespace had pods (used OR logic).

**Solution**: Changed to AND logic - healthy if either namespace has running pods:
```yaml
# OLD - Required both namespaces
{%- if r_keycloak_tssc_pods.resources | length == 0 or r_keycloak_tpa_pods.resources | length == 0 -%}

# NEW - Accept if either namespace has running pods
{%- if r_keycloak_tssc_pods.resources | selectattr('status.phase', 'equalto', 'Running') | list | length == 0 and r_keycloak_tpa_pods.resources | selectattr('status.phase', 'equalto', 'Running') | list | length == 0 -%}
unhealthy
```

### 5. Developer Hub PostgreSQL Detection

**Problem**: Used label selector `app=postgresql` which didn't match actual pod labels.

**Solution**: Search pod names for "postgresql" string:
```yaml
# Get all pods
- name: Get Developer Hub PostgreSQL pods
  kubernetes.core.k8s_info:
    api_version: v1
    kind: Pod
    namespace: backstage
  register: r_backstage_pods

# Filter by name
- name: Check for PostgreSQL pod
  ansible.builtin.set_fact:
    _postgres_pods: "{{ r_backstage_pods.resources | selectattr('metadata.name', 'search', 'postgresql') | list }}"

# Check if running
{%- elif _postgres_pods | selectattr('status.phase', 'equalto', 'Running') | list | length == 0 -%}
no-database
```

### 6. Quay Namespace

**Problem**: Checked `quay-registry` namespace but Quay was in `quay-enterprise`.

**Solution**: Updated namespace:
```yaml
# OLD
namespace: quay-registry

# NEW
namespace: quay-enterprise
```

### 7. HTTP Retry Logic

**Added**: 5-minute retry with 10-second intervals for slow-starting services:
```yaml
- name: Test route accessibility (with retry for 5 minutes)
  ansible.builtin.uri:
    url: "https://{{ route_host }}"
    method: GET
    validate_certs: false
    status_code: [200, 301, 302]
    timeout: 10
  register: r_http_check
  retries: 30  # 5 minutes / 10 seconds = 30 retries
  delay: 10    # Wait 10 seconds between retries
  until: r_http_check is success
  ignore_errors: true
  when: routes | length > 0
```

### 8. Skipped Task Handling

**Problem**: Tasks failed with "dict has no attribute 'failed'" when HTTP check was skipped.

**Solution**: Always check `.skipped` before accessing `.failed`:
```yaml
{%- elif r_http_check is defined and not r_http_check.skipped | default(false) and r_http_check.failed -%}
not-accessible
```

### 9. Loop and Block Syntax

**Problem**: Ansible doesn't support `loop` directly on `block`.

**Solution**: Extracted block content to separate task files and used `include_tasks`:
```yaml
# OLD - Doesn't work
- block:
    - name: Check something
  loop: "{{ items }}"

# NEW - Works
- name: Check each item
  ansible.builtin.include_tasks: check_item_single.yml
  loop: "{{ items }}"
  loop_control:
    loop_var: item
```

### 10. Cluster API URL

**Problem**: Required external `openshift_api_url` variable.

**Solution**: Get it dynamically from cluster:
```yaml
- name: Get cluster information
  kubernetes.core.k8s_cluster_info:
  register: r_cluster_info
  ignore_errors: true

- name: Set cluster API URL
  ansible.builtin.set_fact:
    _cluster_api_url: "{{ r_cluster_info.connection.host | default('N/A') }}"
```

### 11. agnosticd_user_info Module

**Problem**: Module needed to be in collection's plugin directory, not just role's library.

**Solution**:
1. Copied module to `plugins/modules/agnosticd_user_info.py`
2. Copied action plugin to `plugins/action/agnosticd_user_info.py`
3. Also kept in role's `library/` and `action_plugins/` for backwards compatibility
4. Used FQCN in tasks: `rhpds.build_secured_dev_workflows.agnosticd_user_info`

## Output Format

The role saves validation results to `{action}-user-data.yaml` for each user:

```yaml
users:
  user1:
    rhads_validation_status: "healthy"  # or "failed"
    rhads_validation_summary: |
      RHADS Environment Validation Report
      ====================================
      Status: HEALTHY

      Summary:
      - Total Components Checked: 19
      - Healthy: 19
      - Degraded: 0
      - Failed: 0
    rhads_validation_total: 19
    rhads_validation_healthy: 19
    rhads_validation_degraded: 0
    rhads_validation_failed: 0
    rhads_validation_issues: ""  # empty if healthy
```

## Testing

### Build and Install Collection

```bash
cd /root/rhpds.build-secured-dev-workflows
ansible-galaxy collection build --force
ansible-galaxy collection install rhpds-build_secured_dev_workflows-*.tar.gz --force
```

### Run Validation

```bash
ansible-playbook /tmp/test_rhads_validation.yml -e "guid=qqf9f" -e "pool_number=1" -e "num_users=1" -vv
```

### Check Results

```bash
cat provision-user-data.yaml
```

## Troubleshooting

### Module Not Found Error

**Error**: `couldn't resolve module/action 'agnosticd_user_info'`

**Fix**: Ensure module is in collection's `plugins/modules/` and use FQCN:
```yaml
rhpds.build_secured_dev_workflows.agnosticd_user_info:
```

### Undefined Variable Errors

**Error**: `'guid' is undefined` or `'openshift_api_url' is undefined`

**Fix**: Pass required variables when calling role or use role-scoped variables with defaults.

### False Positive Health Checks

**Debug Steps**:
1. Check actual resource on cluster: `oc get <resource> -n <namespace>`
2. Verify namespace is correct
3. Check label selectors match actual resources
4. Review status field paths in Jinja2 templates

### Skipped Task Errors

**Error**: `dict has no attribute 'failed'`

**Fix**: Always check if task was skipped before accessing result attributes:
```yaml
r_task is defined and not r_task.skipped | default(false) and r_task.failed
```

## Future Enhancements

- [ ] Add validation for TAS (Trusted Artifact Signer) components
- [ ] Add validation for TPA (Trusted Profile Analyzer) components
- [ ] Add detailed component dependency checks
- [ ] Add performance metrics collection
- [ ] Add JSON output option for programmatic consumption
- [ ] Add webhook/notification support for validation failures
- [ ] Add retry/wait logic for components still deploying

## References

- AgnosticD v2: https://github.com/agnosticd/
- Core Workloads Pattern: https://github.com/agnosticd/core_workloads
- OLM CSV Docs: https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/
- Kubernetes Core Collection: https://docs.ansible.com/ansible/latest/collections/kubernetes/core/
