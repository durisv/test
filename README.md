[TOC]

# New subscription checklist

 ## Enable Microsoft Defender for Cloud

Navigate to Microsoft Defender for Cloud and enable it for new subscription.

## Enable encryption at host
**note**: this will be automatically enabled during jump-server VM deployment.
```bash
az provider register --namespace Microsoft.Compute
az feature register --namespace Microsoft.Compute --name EncryptionAtHost
```
