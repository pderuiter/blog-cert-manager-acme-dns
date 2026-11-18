# blog-cert-manager-acme-dns

Companion code for the blog post
[Provisioning TLS Certificates with Traefik, cert-manager and ACME-DNS in Kubernetes](https://webuildyourcloud.com/provisioning-tls-certificates-with-traefik-cert-manager-and-acme-dns-in-kubernetes/).

It builds the setup from the post on a single-node k3s cluster (an Ubuntu 24.04 VM):
- Traefik (Helm chart 42.4.1) as the ingress controller, behind a LoadBalancer Service.
- cert-manager v1.21.3, with its CRDs installed by the chart.
- ACME-DNS v2.0.2. Its configuration lives in a ConfigMap and its account database on a PVC. DNS is exposed on a
  LoadBalancer, while the HTTP API stays cluster-internal.
- The ACME-DNS account registration, stored as `acmedns-secret` in the `cert-manager` namespace.
- The `letsencrypt-acmedns` ClusterIssuer.

The playbook prints the records you need in your main DNS: the NS delegation and glue for `auth.example.com`, plus one
`_acme-challenge` CNAME per registered domain.

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | One Proxmox VE VM from the Ubuntu 24.04 cloud image (bpg/proxmox provider). Outputs an Ansible inventory. |
| `ansible/roles/k3s` | k3s v1.37.2+k3s1 (bundled Traefik disabled, ServiceLB enabled), Helm v3.19.0, and a virtualenv with the kubernetes Python client for `kubernetes.core`. |
| `ansible/roles/traefik_ingress` | `helm install traefik traefik/traefik --set service.spec.type=LoadBalancer`. |
| `ansible/roles/cert_manager` | `helm install cert-manager jetstack/cert-manager --set crds.enabled=true`. Its `issuer` tasks create the ClusterIssuer and any `cert_manager_certificates`. |
| `ansible/roles/acme_dns` | Section 3 of the post: the namespace, the DNS and API Services, `config.cfg`, the PVC and the Deployment. The `records` in `config.cfg` are filled in with the LoadBalancer IP. Section 4 is automated too: one `POST /register` per domain in `acme_dns_accounts`, stored as `{"<domain>": <account>}` in `acmedns-secret`. Domains that are already registered are kept. The Services and the Deployment use server-side apply: their port lists have both 53/UDP and 53/TCP, and a client-side patch merges container ports on the port number alone, folding the UDP entry into the TCP one. |
| `ansible/site.yml` | k3s, Traefik, cert-manager, ACME-DNS, then the ClusterIssuer. |
| `tests/verify.yml` | Sections 5 and 6: a `Certificate`, a backend (`my-service`) and the `Ingress`. It waits for `Ready`, then checks the certificate Traefik serves and calls the backend through it. |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM).
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`.
- SSH access with sudo to the machine. The Terraform cloud-init user is `ubuntu`.
- A public IP on the VM (or port forwards), reachable on 53/udp+tcp for ACME-DNS and on 80/443 for Traefik.
- Access to your main DNS zone, to add the delegation and the CNAME(s).

## Quick start

```bash
# 1. Machine (optional; any Ubuntu 24.04 host works)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IP, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Cluster, ACME-DNS and the issuer
cd ../ansible
ansible-galaxy collection install -r requirements.yml
$EDITOR group_vars/all/vars.yml   # acme_dns_domain, acme_dns_accounts, cert_manager_acme_email
ansible-playbook site.yml
```

The last task prints the records to add, for example:

```
auth.example.com.  IN NS  ns1.auth.example.com.
ns1.auth.example.com.  IN A  192.0.2.40
_acme-challenge.example.com.  IN CNAME  4411d03a-5aaa-419b-9bc9-a5d55a94c8df.auth.example.com.
```

After they are in your main DNS, request certificates. Either add them to `cert_manager_certificates` and run
`site.yml` again, or run the end-to-end check:

```bash
ansible-playbook ../tests/verify.yml -e verify_domain=example.com
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and put your host in the `k8s` group.

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `acme_dns_domain` | `auth.example.com` | Subdomain delegated to ACME-DNS |
| `acme_dns_nsname` / `acme_dns_nsadmin` | `ns1.auth.example.com` / `admin.example.com` | `nsname` / `nsadmin` in `config.cfg` |
| `acme_dns_public_ip` | empty (the LoadBalancer IP) | IP for the A records in `config.cfg`. Set it when the VM sits behind NAT |
| `acme_dns_accounts` | `[example.com]` | Domains to register an ACME-DNS account for |
| `acme_dns_image` | `joohoi/acme-dns:v2.0.2` | ACME-DNS image |
| `cert_manager_acme_email` | `your-email@example.com` | ACME account e-mail |
| `cert_manager_acme_server` | Let's Encrypt production | Use the staging directory while testing |
| `cert_manager_acme_ca_bundle` | empty | Base64 CA bundle for a private ACME CA |
| `cert_manager_certificates` | `[]` | `{name, namespace, secret_name, dns_names}` entries to request |
| `cert_manager_chart_version` | `v1.21.3` | cert-manager chart |
| `traefik_ingress_chart_version` | `42.4.1` | Traefik chart (Traefik v3.7) |
| `k3s_version` | `v1.37.2+k3s1` | k3s release |

The ACME-DNS API runs over plain HTTP inside the cluster
(`cert_manager_acme_dns_host: http://acme-dns.acme-dns.svc.cluster.local`), as in the post. It is not exposed outside
the cluster.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container (k3s v1.37.2+k3s1). The second run reported
`changed=0`; an existing registration is reused, not repeated. For the end-to-end check, Pebble replaced Let's Encrypt
(`cert_manager_acme_server` + `cert_manager_acme_ca_bundle`), and a BIND container acted as the main DNS for a test
domain, with the printed NS, glue and CNAME records. `tests/verify.yml` then passed:
- The Certificate went `Ready`.
- Traefik served the certificate issued by the Pebble CA for the domain.
- The backend answered through the Ingress.

ansible-lint (production profile), yamllint, `terraform validate` and tflint are clean. The Terraform code has not been
applied against a real Proxmox cluster.

## License

MIT
