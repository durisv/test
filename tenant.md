# Deployment Guide - Shared & Tenant & Connector

## Shared

1. Edit  dw-settings.json

    ```bash
    cd /srv/spring22
    code dw-settings.json # if using VS Code, or any other preferred editor

    {
            "resource_group_prefix": "dw",
            "region": "", # azure region, same as when deploying infrastructure
            "aks": "aks",
            "region_resource_group": "{resource_group_prefix}-{region}",
            "aks_resource_group": "{resource_group_prefix}-{region}-aks-1",
            "b2c_resource_group": "{resource_group_prefix}-{region}-b2c",
            "domain": "2ring.cloud",
            "subscription_id": "", # subscription in which all azure resources reside
            "tenant_version": "9",
            "shared_version": "9",
            "acr": "tworing.azurecr.io" # acr login server
    }
    ````

1. Init shared resources
**During shared initialization you can encounter port-forward error. This is completely normal and can be ignored.**
**If you switched user to new user you will be prompted to enter sudo password during script execution.**

1. Prepare your **SendGrid API key**, you will be prompted to enter it during initialization.

    ```bash
    cd /srv/spring22/shared

    # may ask for sudo password if you switched users
    ./apply.py --init
    ```

1. Update repository after apply script.

    ```bash
    cd /srv/spring22
    git add .
    git commit -a -m "Commit message"
    git push
    ```

## Tenant

1. Create DNS Entry
    > **Note**:
    > Right now you should have your tenant_id. We need to create **A record in DNS** for our tenant_id which should point load balancer public ip or public dns name. Both can be find out on load balancer overview section in Azure.
    > **e.g. mycompany.2ring.cloud should resolve to Load balancer public ip.**

2. Check if you are logged into azure cli

   ```bash
   # outputs your current context
   az account show

   # if previous info is incorrect or you are not logged in
   az logout
   az login --tenant {azure_tenant_id} --allow-no-subscriptions --use-device-code
   az account set -s {subscription_name}
   ```

3. Create new tenant resources
   > **Note**:
   > This step will create all tenant files on disk and install database. \
   > **During creation of new tenant you can encouter port-forward error when installing database this is completely normal and can be ignored.**

   ```bash
   # {tenant_id} can consist only from alphanumerical lowercase characters
   # {db_set_index} most of the time should be 0, unless you have more db servers running
   cd /srv/spring22
   ./create_new_tenant.py --tenant-id {tenant_id} --db-set-index {db_set_index}
   ```

4. Create B2C tenant
    > **Note**:
    > Location and country code can be found here <https://docs.microsoft.com/sk-sk/azure/active-directory-b2c/data-residency>
    > locations: `'United States','Europe','Asia Pacific','Australia'`
    > country_codes: `e.g. 'US', 'CA' for United States, e.g. 'DE', 'SK' for Europe`

    ```bash
    # during this script execution you will be prompted to log in to newly created b2c
    # this will reconfigure your current az context to run without subscription which will be switched back during init_admins.py execution
    cd tenants/{tenant_id}
    ./init_b2c_tenant.py --location {location} --country-code {coutry_code}
    ```

5. Update repositories after init_b2c_tenant script.

    ```bash
    cd /srv/infrastructure
    git add .
    git commit -a -m "Commit message"
    git push

    cd /srv/spring22
    git add .
    git commit -a -m "Commit message"
    git push
    ```

6. Manually setup B2C tenant
    1. Navigate to the Azure portal and select the newly created Azure AD B2C. You might need to switch current directory to newly created b2c tenant.
    2. Select the App Registrations. Click on "Client API" app. On the left side menu, select the Manifest blade.

        ![select_application.png](./img/select_application.png)
        - Set accessTokenAcceptedVersion property to 2.
        - Click on Save.
        - More info:
        - <https://docs.microsoft.com/en-us/azure/active-directory/develop/id-tokens#claims-in-an-id-token>
        - <https://docs.microsoft.com/en-us/azure/active-directory/develop/access-tokens#token-formats-and-ownership>
        - Basically v1 and v2 differ in claims included in token and Microsoft suggets to use v2 in new applications.

            ![access_token_version.png](./img/access_token_version.png)

    3. Select the App Registrations. Click on "IdentityExperienceFramework" app. On the left side menu, select API permissions. Grant admin consent for B2C Tenant

    4. Select the App Registrations. Click on "ProxyIdentityExperienceFramework" app. On the left side menu, select API permissions. Grant admin consent for B2C Tenant

        ![admin_consent.png](./img/admin_consent.png)

    5. Update SignIn userflow

        - Navigate to Azure AD B2C tenant, click User flows and select B2C_1_SignIn

            ![ad_b2c_sign_in_template.png](./img/ad_b2c_sign_in_template.png)
        - In Application claims blade select these claims: `bu, bua, Display Name, Email Addresses, Given Name, Identity Provider Access Token, mbu, sa, Surname, ta, User's Object ID`
        - Click Save button

            ![image.png](./img/image.png)

    6. Update page layouts
        - Navigate to Azure AD B2C tenant, click User flows and select B2C_1_SignIn

            ![ad_b2c_sign_in_template.png](./img/ad_b2c_sign_in_template.png)
        - In Overview blade under Customize section, select Page layouts

            ![ad_b2c_page_layouts.png](./img/ad_b2c_page_layouts.png)

        - In page layouts blade do following steps:

            1. Select "Sign in page" row
            2. Switch "Use custom page content" to "Yes"
            3. Fill in "Custom page URI" with <<https://<tenant_id>>>.<domain>/templates/index.html
            4. Click Save button
            5. Repeat previous steps for following rows:

                - Update expired password page
                - Forgot password page
                - Change password page

                   ![ad_b2c_page_layouts_custom_page.png](./img/ad_b2c_page_layouts_custom_page.png)

        - Add additional users as B2C Global admins (Optional)

            1. In portal navigate to new Azure AD B2C tenant.
            2. Open the Users blade and click New user.
            3. Choose Invite user, fill in Email address and under Groups and roles click User.

               ![ad_b2c_new_user.png](./img/ad_b2c_new_user.png)
               ![ad_b2c_new_user_role.png](./img/ad_b2c_new_user_role.png)

            4. Find and select Global administrator role.

               ![ad_b2c_new_user_invite.png](./img/ad_b2c_new_user_invite.png)

            5. Click invite.
            6. Go to provided email address and accept invitation.

    ---

7. Multi factor authentication (Optional)
    >
    >    **Per user**
    >
    >    1. Prepare your Azure AD B2C tenant
    >  <https://docs.microsoft.com/en-us/azure/active-directory-b2c/conditional-access-user-flow?pivots=b2c-user-flow#prepare-your-azure-ad-b2c-tenant>
    >    2. Add a Conditional Access policy
    >  <https://docs.microsoft.com/en-us/azure/active-directory-b2c/conditional-access-user-flow?pivots=b2c-user-flow#add-a-conditional-access-policy>
    >    3. Enable multi-factor authentication (Conditional)
    >   <https://docs.microsoft.com/en-us/azure/active-directory-b2c/conditional-access-user-flow?pivots=b2c-user-flow#enable-multi-factor-authentication-optional>
    >
    >    **Per tenant**
    >
    >    1. Enable multi-factor authentication (Always on)
    >   <https://docs.microsoft.com/en-us/azure/active-directory-b2c/conditional-access-user-flow?pivots=b2c-user-flow#enable-multi-factor-authentication-optional>
    >

8. Continue tenant deployment

    ```bash
    # script will ask for details about 2 new users that will created (system and tenant admin)
    cd /srv/spring22/tenants/{tenant_id} # if not already here
    ./init_admins.py

    # previous script should switch us back to original subscriptions, we can verify that by running
    az account show --query id # should print id of our active subscription

    ./apply.py
    ```

## Connector

1. Navigate to admin tool and manually add connector by uploading package.
Fill in all required fields (timezone, connector settings, license, etc).
**Generate new api key and save it for next script.**

2. Create and apply new connector

    ```bash
    cd /srv/spring22/tenants/{tenant_id}
    # script will prompt you for apikey
    ./create_new_connector --name {connector_name} --type {connector_type - wxcc,genesys} --connector-version {version}
    cd ./connectors/{connector_name}
    ./apply_connector.py
    ```
