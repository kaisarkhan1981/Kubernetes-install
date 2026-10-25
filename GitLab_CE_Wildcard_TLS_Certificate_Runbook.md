# Install a Wildcard TLS Certificate on On-Premises GitLab CE

**Step-by-step operations runbook** | Linux package (Omnibus) | GitLab HTTPS + private registry on port `5050`

## Environment

| Environment input | Planned value / placeholder |
|---|---|
| GitLab server IP | `172.31.150.56` |
| GitLab HTTPS hostname | `gitlab.example.com` (replace with your real FQDN) |
| Registry HTTPS hostname | `gitlab.example.com:5050` (same host) or `registry.example.com:5050` |
| Certificate | Existing wildcard certificate, private key, and CA chain |
| Shell identity | `tbl` with `sudo` |
| TLS termination | GitLab host NGINX; F5 alternative covered separately |

> [!IMPORTANT]
> Use a hostname covered by the wildcard certificate. For example, `*.example.com` matches `gitlab.example.com` and `registry.example.com`, but **not** `172.31.150.56` or `a.b.example.com`. These commands assume the GitLab Linux-package installation, **not Docker or Helm**.

## 1. Preflight checks and naming

1. Identify the actual hostname covered by the certificate. Use your organization's real domain rather than the `example.com` placeholders.
2. Ensure internal DNS resolves the GitLab and registry hostnames to `172.31.150.56`, or to the intended F5 virtual IP if TLS terminates at F5.
3. Check GitLab version, existing settings, listeners, and hostname resolution:

   ```bash
   sudo gitlab-rake gitlab:env:info
   sudo grep -E "^(external_url|registry_external_url|nginx\[|registry_nginx\[|letsencrypt\[)" /etc/gitlab/gitlab.rb
   sudo ss -lntp | grep -E ":(80|443|5050) " || true
   getent hosts gitlab.example.com
   ```

4. Confirm ports `443` and `5050` are reachable from GitLab clients, CI runners, and Kubernetes nodes. Confirm that the certificate is valid and unexpired.

> [!WARNING]
> Changing the external URL from HTTP/IP to HTTPS/FQDN changes clone URLs, runner endpoints, and registry image references. Coordinate the cutover and back up the existing `gitlab.rb`.

## 2. Inspect the wildcard certificate

Obtain the wildcard leaf certificate, matching **unencrypted PEM** private key, and intermediate CA bundle securely from the certificate administrator. Never commit private keys to Git or paste them into chat or tickets.

```bash
openssl x509 -in wildcard.crt -noout -subject -issuer -dates -ext subjectAltName
openssl pkey -in wildcard.key -check -noout
openssl x509 -in wildcard.crt -pubkey -noout | openssl sha256
openssl pkey -in wildcard.key -pubout | openssl sha256
```

The two SHA-256 hashes of the public keys **must match**. Verify that the certificate's Subject Alternative Names (SANs) cover both hostnames.

If the certificate is supplied as `.pfx` / `.p12`, ask the PKI administrator to export the PEM certificate, key, and chain securely. Do not expose bundle passwords in shell history.

## 3. Install certificate files

As the `tbl` user, stage the files in a restricted location. These commands assume `wildcard.crt`, `wildcard.key`, and `intermediate-chain.pem` are in the working directory.

```bash
sudo install -d -m 755 /etc/gitlab/ssl
cat wildcard.crt intermediate-chain.pem > gitlab-fullchain.crt
sudo install -o root -g root -m 644 gitlab-fullchain.crt /etc/gitlab/ssl/gitlab-fullchain.crt
sudo install -o root -g root -m 600 wildcard.key /etc/gitlab/ssl/gitlab-wildcard.key
sudo openssl x509 -in /etc/gitlab/ssl/gitlab-fullchain.crt -noout -dates
```

The fullchain should contain the **leaf certificate first**, followed by intermediate certificate(s), in the order prescribed by the CA. The root CA is generally distributed through client trust stores instead of being included in the served chain.

## 4. Configure GitLab and Registry HTTPS

Back up the GitLab configuration (this is **not** a Kubernetes or cluster backup):

```bash
sudo cp -a /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.rb.pre-tls
```

Edit `/etc/gitlab/gitlab.rb`. Choose **one** registry hostname option and remove conflicting previous settings.

### Option A: Same hostname, registry on port 5050

```ruby
external_url "https://gitlab.example.com"
registry_external_url "https://gitlab.example.com:5050"
letsencrypt['enable'] = false

# GitLab web NGINX - common Omnibus syntax
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab-fullchain.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab-wildcard.key"
nginx['redirect_http_to_https'] = true

# Container Registry NGINX
registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab-fullchain.crt"
registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab-wildcard.key"
```

### Option B: Separate registry hostname

Set this instead of the registry URL in Option A:

```ruby
registry_external_url "https://registry.example.com:5050"
```

Ensure that `registry.example.com` resolves to the appropriate server or VIP and is covered by the certificate SAN.

> [!NOTE]
> **GitLab version compatibility:** The source runbook notes a settings-syntax change for GitLab 19.2+, mentioning `gitlab_rails['nginx'][...]` and `registry['nginx'][...]` versus older top-level `nginx[...]` / `registry_nginx[...]`. Check the documentation for your **installed version** before applying these keys, and do not configure duplicate styles for the same setting.

## 5. Apply the GitLab configuration

```bash
sudo gitlab-ctl reconfigure
sudo gitlab-ctl status
sudo ss -lntp | grep -E ":(443|5050) "
```

If reconfiguration fails, inspect `/var/log/gitlab/nginx/` and `/var/log/gitlab/registry/`. Do not delete old certificate files until verification passes.

## 6. Verify HTTPS and registry authentication

```bash
curl -Iv https://gitlab.example.com
curl -Iv https://gitlab.example.com:5050/v2/
openssl s_client -connect gitlab.example.com:443 -servername gitlab.example.com -verify_return_error </dev/null
openssl s_client -connect gitlab.example.com:5050 -servername gitlab.example.com -verify_return_error </dev/null
```

An HTTP `401 Unauthorized` response from `/v2/` with a `WWW-Authenticate` header commonly means the registry is reachable but requires authentication. Verify TLS and hostname validation first.

```bash
docker login gitlab.example.com:5050
# Supply a GitLab deploy token or scoped personal access token interactively.
```

Use `write_registry` permissions for CI image pushes and `read_registry` for Kubernetes pulls. Test a small image push/pull before changing production workloads.

## 7. Update GitLab Runner and Kubernetes clients

1. Update the GitLab Runner instance URL to the HTTPS FQDN if it previously used HTTP or an IP. Validate runner registration and jobs.
2. Use `$CI_REGISTRY_IMAGE` in GitLab CI. For example, an image reference could be `gitlab.example.com:5050/it/argocd:<commit-sha>`.
3. Update Kubernetes Deployment image references and `imagePullSecrets` to the registry DNS hostname and port. Credentials associated with `172.31.150.56:5050` do not automatically match `gitlab.example.com:5050`.

```bash
kubectl -n default create secret docker-registry gitlab-registry-secret \
  --docker-server=gitlab.example.com:5050 \
  --docker-username="<deploy-token-username>" \
  --docker-password="<read-registry-token>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

> [!CAUTION]
> Avoid typing actual token values into interactive shell history. Provide them through a protected secret store or controlled environment variable. With a private PKI, install the issuing CA into the GitLab Runner Docker daemon and the Kubernetes node container runtime trust stores. **Do not disable TLS certificate verification.**

## 8. F5 TLS termination alternative

**Only use this section if F5 terminates TLS instead of GitLab NGINX.**

1. Securely import the wildcard certificate, matching key, and CA chain to F5.
2. Attach an appropriate client SSL profile to the F5 virtual server.
3. Ensure F5 passes the correct `Host` and `X-Forwarded-*` headers.
4. Choose either verified HTTPS between F5 and GitLab or an explicitly approved internal HTTP backend.
5. Configure GitLab NGINX `listen_https` / `listen_port` to match the offload arrangement. Avoid contradictory TLS settings.

If no F5 sits in front of GitLab, skip this section. The cluster F5 VIP `172.31.150.60:6443` is for the **Kubernetes API**, not automatically GitLab HTTPS.

## 9. Rollback procedure

If the cutover fails, restore the previous GitLab configuration and coordinate any DNS or client endpoint rollback.

```bash
sudo cp -a /etc/gitlab/gitlab.rb.pre-tls /etc/gitlab/gitlab.rb
sudo gitlab-ctl reconfigure
sudo gitlab-ctl status
```

Keep the new certificates in place until the cutover has succeeded. For certificate renewal without configuration changes, replace the certificate/key files atomically with correct permissions, then reload GitLab NGINX:

```bash
sudo gitlab-ctl hup nginx
```

Re-verify both web and registry endpoints.

## 10. Troubleshooting quick reference

| Symptom | Check / corrective action |
|---|---|
| `x509: certificate valid for another name` | Use a DNS hostname covered by the wildcard SAN, not a bare IP. |
| `unable to get local issuer certificate` | Correct the leaf/intermediate chain; install the trusted internal CA on clients. |
| Registry login succeeds, image push fails | Check registry URL/port, proxy headers, token scope, and registry logs. |
| Docker reports an HTTP response to an HTTPS client | Check `registry_external_url` and reverse-proxy SSL/offload settings. |
| HTTPS UI works, registry port `5050` fails | Check registry NGINX certificate settings, firewall, and listener. |
| GitLab Runner fails after hostname change | Update runner GitLab URL, CA trust, and DNS/network resolution. |

## 11. Completion checklist

- [ ] Wildcard SANs verified for GitLab and registry hostnames.
- [ ] Certificate and private key match; key file permissions are restricted.
- [ ] GitLab HTTPS UI validated from user and runner networks.
- [ ] Registry `/v2/` TLS checked; Docker login, push, and pull validated.
- [ ] GitLab CI image references updated to the DNS endpoint.
- [ ] Kubernetes nodes trust the CA; `imagePullSecrets` and Deployments updated.
- [ ] Backup copy of `gitlab.rb` retained; certificate renewal owner and expiry date recorded.

## Official references

- [GitLab TLS configuration](https://docs.gitlab.com/omnibus/settings/ssl/)
- [GitLab Container Registry](https://docs.gitlab.com/administration/packages/container_registry/)
- [GitLab DNS settings](https://docs.gitlab.com/omnibus/settings/dns/)
