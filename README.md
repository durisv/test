# Deployment Guide - Infrastructure

## 1.1 Create Infrastructure

1. **Create a DNS record for this URL: {grafana_dns_prefix}-grafana.2ring.cloud. This URL should resolve to the Load balancer IP address.**
2. Edit  infrastructure-settings.json

    ```bash
    cd /srv/infrastructure
    code infrastructure-settings.json # if using VS Code, or any other preferred editor

    {
            "lb_public_ip": "lb_public_ip",
            "resource_group_prefix": "dw",
            "region": "", # azure region, same as when deploying infrastructure
            "aks": "aks",
            "region_resource_group": "{resource_group_prefix}-{region}",
            "aks_resource_group": "{resource_group_prefix}-{region}-aks-1",
            "domain": "2ring.cloud",
            "letsencrypt_server": "https://acme-v02.api.letsencrypt.org/directory",
            "aks_pools_subnet_cidr": "10.1.0.0/16",
            "grafana_dns_prefix": "",
            "grafana_fqdn": "{grafana_dns_prefix}-grafana.{domain}",
            "acr": "tworing.azurecr.io"  # acr login server
    }
    ```

    **Note**:  If you want to disable prometheus edit `./infrastructure/.hidden/base/loki-stack/values.yaml`

    ```yaml
    prometheus:
      enabled: false # add this line
    ```

    **Note**: If you want to access Grafana portal, edit `./infrastructure/.hidden/cloud/base/loki-stack/dw-loki-stack/values.yaml`

    ```yaml
      whiteListSourceRange: 127.0.0.1 # replace with your IP address or CIDR range
    ```

    **Note**: If you have less than 3 nodes in you cluster you should set nginx-ingress-controller replicas count to 1,
    edit `./infrastructure/.hidden/cloud/base/ingress-nginx/kustomization.yaml`

    This is because of pod node affinity where we need to have at least one free node where we can schedule restarting ingress-nginx pods

    ```yaml
    valuesInline:
      controller:
        replicaCount: 2 # set to one
    ```

3. Init infrastructure resources
**During initialization you can encounter port-forward error. This is completely normal and can be ignored.**
**If you switched user to new user you will be prompted to enter sudo password during script execution.**

    ```bash
    cd /srv/infrastructure/infrastructure
    # may ask for sudo password if you switched users
    ./apply.py --init

    # The script at the end shows password for admin user (dw_admin). This password must be stored in a safe place
    ```

4. Update repository after apply script.

    ```bash
    cd /srv/infrastructure
    git add .
    git commit -a -m "Commit message"
    git push
    ```
