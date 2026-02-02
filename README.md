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

### HTTPS / Ingress TLS (when `awx_use_https` is true)

| Variable | Default | Description |
| --- | --- | --- |
| `awx_use_https` | `false` | When true, enable Ingress with TLS; `awx_controller_url` becomes `https://<hostname>`. |
| `awx_ingress_hostname` | `""` | FQDN for the AWX Ingress (required when `awx_use_https` is true). |
| `awx_ingress_tls_secret` | `""` | Name of the TLS Secret in namespace `awx` (required when `awx_use_https` is true). |
| `dns_provider` | `""` | Set to `cloudflare` to use cert-manager DNS-01 with Cloudflare; credentials in `acme_plugin_data` (from SOPS only). |
| `acme_plugin_data` | `""` | Cloudflare credentials: **dict** (`CF_Token`, `cf_token` o `api_token`) o **string** con líneas `CF_Token=...` / `api_token=...`. Cargar desde SOPS solo. |
| `cert_manager` | `false` | When true, use cert-manager **without** Cloudflare (e.g. Let's Encrypt HTTP-01). Requires cert-manager installed. |
| `awx_ingress_acme_email` | `""` | Email for Let's Encrypt account (required when using cert_manager or Cloudflare). |
| `awx_ingress_acme_server` | Let's Encrypt prod | ACME server URL (use staging for testing). |
| `awx_ingress_tls_cert_path` | `""` | Path to TLS certificate file; role creates the Secret from this and the key file. |
| `awx_ingress_tls_key_path` | `""` | Path to TLS key file (used with `awx_ingress_tls_cert_path`). |
| `awx_controller_validate_certs` | `true` when HTTPS | Whether to validate TLS in post_tasks; set to `false` for self-signed. |
| `cert_manager_version` | `v1.19.2` | cert-manager version to install when `cert_manager` or `dns_provider: cloudflare` is used. |
| `cert_manager_manifest_url` | GitHub release URL | Override manifest URL for cert-manager (default built from `cert_manager_version`). |
| `awx_ingress_wait_for_tls_secret` | `true` | When true, wait for cert-manager to create the TLS Secret before deploying the AWX CR (issuance can take several minutes). Set to `false` to skip the wait and continue; cert-manager will create the Secret asynchronously. |
| `awx_ingress_wait_tls_retries` | `36` | Number of retries when waiting for the TLS Secret (default 36 × 10 s ≈ 6 min). |
| `awx_ingress_wait_tls_delay` | `10` | Seconds between retries when waiting for the TLS Secret. |

**TLS provisioning modes:**

1. **Cloudflare** (`dns_provider: cloudflare`): Credentials in `acme_plugin_data` (dict: `CF_Token`/`api_token`, or string with lines `CF_Token=...`). Role **installs cert-manager** if needed, then creates Secret, ClusterIssuer (DNS-01), and Certificate.
2. **Autofirmado**: Do not set `dns_provider`, `cert_manager`, or cert/key paths; role generates a self-signed cert and creates the TLS Secret.
3. **cert_manager** (`cert_manager: true`): Role **installs cert-manager** in the cluster, then creates ClusterIssuer (HTTP-01) and Certificate. Hostname must be publicly reachable.
4. **Ruta a certificados**: Set `awx_ingress_tls_cert_path` and `awx_ingress_tls_key_path`; role creates the TLS Secret from those files (paths on the target host).

**Secrets created by the role:**

- **Self-signed:** The role creates the TLS Secret (`tls.crt`, `tls.key`) in namespace `awx`.
- **Path to cert/key:** The role creates the TLS Secret in namespace `awx` from the provided files.
- **Cloudflare:** The role creates the **Cloudflare API token Secret** in namespace `cert-manager` (used by cert-manager for DNS-01 challenges). cert-manager uses this Secret for both **initial issuance and renewals**; do not delete it. cert-manager then creates the TLS Secret in namespace `awx` when the Certificate is issued.
- **cert_manager HTTP-01:** cert-manager creates the TLS Secret in namespace `awx` when the Certificate is issued.

**Security:** Never store `acme_plugin_data` (Cloudflare tokens) in plain group_vars or in git. Load from SOPS (e.g. `-e @~/secure/sops/awx-ingress-cloudflare.yml`). If credentials were exposed, rotate them in Cloudflare immediately. The Cloudflare credentials Secret is required for certificate renewals; keep it in the cluster.

**Access with HTTPS (avoid 404 and SSL errors):**

- **Use the exact hostname** defined in `awx_ingress_hostname`. Open `https://<awx_ingress_hostname>` (e.g. `https://awx.example.com`). Do **not** use the server IP or another hostname: the Ingress matches requests by `Host`; if it doesn’t match, NGINX returns **404 Not Found**.
- Ensure that hostname resolves to the node or Load Balancer IP (DNS record or client `/etc/hosts`, e.g. `<node-ip>  awx.example.com`).
- **Port 8043** is used only for HTTP port-forward when `awx_use_https` is false. With HTTPS enabled, port-forward is disabled. Do **not** use `https://host:8043` — that port serves plain HTTP and will cause an SSL/protocol error.
- **ERR_CONNECTION_REFUSED from outside:** Ensure TCP 80 and 443 are reachable: host firewall (e.g. ufw; the `awx_host_prep` role can allow 80/443 when ufw is active) and **cloud/network security group** (open 80 and 443 to the node or load balancer IP).

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

### Example with HTTPS (self-signed)

```yaml
- name: Install AWX with HTTPS (self-signed cert)
  hosts: awx-rone
  gather_facts: true
  collections:
    - kubernetes.core
  roles:
    - role: awx_operator
      vars:
        awx_use_https: true
        awx_ingress_hostname: "awx.example.local"
        awx_ingress_tls_secret: "awx-tls"
        awx_controller_validate_certs: false
```

### Example with HTTPS (Cloudflare DNS-01)

Load `acme_plugin_data` from SOPS only (e.g. `-e @~/secure/sops/awx-ingress-cloudflare.yml` with `acme_plugin_data: { CF_Token: "..." }`). The role installs cert-manager in the cluster when using Cloudflare or `cert_manager: true`.

```yaml
- name: Install AWX with HTTPS (Let's Encrypt via Cloudflare)
  hosts: awx-rone
  gather_facts: true
  collections:
    - kubernetes.core
  roles:
    - role: awx_operator
      vars:
        awx_use_https: true
        awx_ingress_hostname: "awx.example.com"
        awx_ingress_tls_secret: "awx-tls"
        dns_provider: cloudflare
        awx_ingress_acme_email: "admin@example.com"
        # acme_plugin_data from SOPS (CF_Token or api_token)
```

## License

GPL-3.0-or-later

## Author Information

Mauro Rosero P. (Libre Technology)

