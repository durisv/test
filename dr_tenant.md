# Disaster Recovery - Tenant

## Shared

1. Go to folder

    ```bash
    cd /srv/spring22
    direnv allow .
    ```

2. Open the dw-settings.json file and change these settings:
    > **Note**: Change only if you restoring to different subscription \
    > **Note**: Use the same values as in the default-settings.json (azure resources) file
      - subscription_id
      - region

3. Init shared resources
**During shared initialization you can encounter port-forward error. This is completely normal and can be ignored.**

    ```bash
    cd /srv/spring22/shared
    ./apply.py --init
    ```

## Tenant (repeat for all tenants)

1. Change the {tenant_id}.2ring.cloud DNS record so that it resolves to the new Application Gateway IP address.

2. Restore tenant

    ```bash
    cd /srv/spring22/tenants/{tenant_id}

    ./restore_tenant.py --db-set-index {db_set_index} --aes-key {aes_key}
    ```

3. Restore tenant database
    > **Note:** If you run the restore_db.py script with the **--restore-only** flag, the **--logical-name** parameter should follow the pattern `{tenant_id}_DW`

    - Full backup (Option 1)

        ```bash
        # 1. Files are available on Azure Portal in resource group dw-{region} -> Storage account -> File shares -> db-backup
        # 2. you can create a backup via SSMS and copy from DB pod to jump-server

        cd /srv/spring22/database/backups
        ./restore_db.py --restore-only --logical-name {tenant_DW} --backup-path {tenant_DW.bak} --db-set-index {0} --tenant-id {tenant}
        ```

    - Differential backup (Option 2)

        ```bash
        # 1. Files are available on Azure Portal in resource group dw-{region} -> Storage account -> File shares -> db-backup
        # 2. you can create a backup via SSMS and copy from DB pod to jump-server

        cd /srv/spring22/database/backups
        ./restore_db.py --restore-only --logical-name {tenant_DW} --backup-path {tenant_DW.bak} --dff-backup-path {tenant_DW.dff} --db-set-index {0} --tenant-id {tenant}
        ```

4. Recreate User Management secrets

    ```bash
    cd /srv/spring22/tenants/{tenant_id}
    ./init_b2c_tenant.py --recovery-mode --location {location} --country-code {country-code}
    ```

5. Apply tenant resources

    ```bash
    cd /srv/spring22/tenants/{tenant_id}

    ./apply.py
    ```

## Connector (repeat for all connectors)

1. Restore and apply connector

    ```bash
    cd /srv/spring22/tenants/{tenant_id}/connectors/{connector_name}

    # script will prompt you for apikey
    ./restore_connector.py

    ./apply_connector.py
    ```
