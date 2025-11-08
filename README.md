# awx_operator

Role that installs the AWX Operator on a MicroK8s cluster, deploys an AWX instance, and configures optional helpers such as persistent port-forwarding for ClusterIP services.

## Requirements

- MicroK8s installed and configured on the target host (typically via `awx_host_prep`).
- `kubernetes.core` collection available on the control node.
- Ansible 2.15+.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `awx_operator_namespace` | `awx` | Namespace where the operator and AWX custom resource live. |
| `awx_operator_version` | `2.19.1` | AWX Operator version to deploy. |
| `awx_operator_release_url` | GitHub tarball URL | Tarball source for the operator manifests. |
| `awx_operator_bundle_path` | `/opt/awx-operator-{{ version }}` | Path where the operator sources will be extracted. |
| `awx_operator_admin_user` | `{{ ansible_user }}` | System account used to run `kubectl`/MicroK8s commands. |
| `awx_operator_kubeconfig` | `/home/{{ awx_operator_admin_user }}/.kube/config` | Kubeconfig used to manage the cluster. |
| `awx_instance_name` | `awx` | AWX custom resource name. |
| `awx_instance_spec` | Minimal spec with `ClusterIP` service | Spec applied to the AWX CR (override for custom configuration). |
| `awx_operator_display_status` | `true` | When true, prints pod/service diagnostics during deployment. |
| `awx_operator_port_forward_enabled` | `true` | Automatically configure a systemd service to keep a port-forward alive for ClusterIP deployments. |
| `awx_operator_port_forward_port` | `8043` | External port for port-forwarding. |
| `awx_operator_access_host` | `{{ ansible_host }}` | Host/IP used in access messages. |

Refer to `defaults/main.yml` for the complete list.

## Dependencies

- `kubernetes.core` collection.
- MicroK8s with required add-ons (e.g., DNS, storage, ingress).

## Example Playbook

```yaml
- name: Install AWX operator and instance
  hosts: awx-rone
  gather_facts: true
  collections:
    - kubernetes.core
  roles:
    - role: awx_operator
      vars:
        awx_instance_spec:
          service_type: NodePort
```

## License

GPL-3.0-or-later

## Author Information

Mauro Rosero P. (Libre Technology)

