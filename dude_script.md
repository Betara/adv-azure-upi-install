# OCP 4.6 UPI Advanced Installation Script for Azure 

Ref: YouTube Video - https://youtu.be/h2QfP9IYzeY


This script expects the SPN to have Contributor rights assigned to the Resource Group. This script and accompanying customized templates are provided to help install OpenShift 4.6+ on Azure within a Private Network with Restricted “Contributor” roles to the SPN and Managed Identity. 
```
Note: * Elevated privileges required for this step, please have your administrator / owner build these in advance along with the Resource Group, VNET and all DNS entries.
```

## PRE INSTALL SETUP

ssh-keygen -t rsa -b 4096
### Environment Variables Used 
```
export AZURE_REGION=westeurope
export CLUSTER_NAME=sandbox
export BASE_DOMAIN=hitho.de
export INFRA_ID=openshift
export PRVT_DNS="${CLUSTER_NAME}.${BASE_DOMAIN}"
export RESOURCE_GROUP=rg-euw-cluster
export VNET_RG=rg-euw-network
export VNET=vnet-westus-001
export INSTALL=~/installation
export KUBECONFIG=${INSTALL}/auth/kubeconfig
export SSH_KEY=`cat ~/.ssh/id_rsa.pub`
export STORAGE_SA=vhdsa
export MASTER_SUB=master-subnet
export WORKER_SUB=worker-subnet

export MAN_ID=openshift-identity

export INT_LB=10.0.10.110
export INT_APPS_LB=10.0.20.120

{
  "appId": "c67503ac-0311-49e7-ba4f-c6ed69b0d79f",
  "displayName": "openshift-sp",
  "password": "ZU18Q~beP0asff6TeLApWrNs5DAvVfI~LgcHVbTF",
  "tenant": "a36d4dc0-fd05-44bf-9a9b-c58d27ee96db"
}
az role assignment create --role "User Access Administrator"  --assignee-object-id $(az ad sp list --filter "appId eq 'c67503ac-0311-49e7-ba4f-c6ed69b0d79f'" | jq '.[0].id' -r)

{
  "canDelegate": null,
  "condition": null,
  "conditionVersion": null,
  "description": null,
  "id": "/subscriptions/b872424d-d9b1-4f02-9ce6-eba1e0f9e731/providers/Microsoft.Authorization/roleAssignments/28e210de-f52e-47ee-8cc8-cf1d26c5c959",
  "name": "28e210de-f52e-47ee-8cc8-cf1d26c5c959",
  "principalId": "32741da2-c85b-44d8-8f53-4bc9b55ea762",
  "principalType": "ServicePrincipal",
  "roleDefinitionId": "/subscriptions/b872424d-d9b1-4f02-9ce6-eba1e0f9e731/providers/Microsoft.Authorization/roleDefinitions/18d7d88d-d35e-4fb5-a5c3-7773c20a72d9",
  "scope": "/subscriptions/b872424d-d9b1-4f02-9ce6-eba1e0f9e731",
  "type": "Microsoft.Authorization/roleAssignments"
}

az ad app permission add --id c67503ac-0311-49e7-ba4f-c6ed69b0d79f --api 00000002-0000-0000-c000-000000000000 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role

az ad app permission grant --id c67503ac-0311-49e7-ba4f-c6ed69b0d79f --api 00000002-0000-0000-c000-000000000000 --scope /subscriptions/b872424d-d9b1-4f02-9ce6-eba1e0f9e731/resourceGroups/rg-euw-cluster

{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#oauth2PermissionGrants/$entity",
  "clientId": "32741da2-c85b-44d8-8f53-4bc9b55ea762",
  "consentType": "AllPrincipals",
  "id": "oh10MlvI2ESPU0vJtV6nYvBhdYESxOhGk6Bl8B7ba3Q",
  "principalId": null,
  "resourceId": "817561f0-c412-46e8-93a0-65f01edb6b74",
  "scope": "/subscriptions/b872424d-d9b1-4f02-9ce6-eba1e0f9e731/resourceGroups/rg-euw-cluster"
}

```
{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGVmMTVmM2E4NzQ2NDMzMGE2YzExNDIzOWY1ZTEyMzY6NEJXNUdDS0IyWEs2OTRBUVpCOFFCSUY3NFUxUEhIQlhaNE9TUE9MTjVMUzNISTFZMDY4TE8yT1EzVUJHN0s2Sw==","email":"extern.thomas.hirmer@volkswagen.de"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K29jbV9hY2Nlc3NfZGVmMTVmM2E4NzQ2NDMzMGE2YzExNDIzOWY1ZTEyMzY6NEJXNUdDS0IyWEs2OTRBUVpCOFFCSUY3NFUxUEhIQlhaNE9TUE9MTjVMUzNISTFZMDY4TE8yT1EzVUJHN0s2Sw==","email":"extern.thomas.hirmer@volkswagen.de"},"registry.connect.redhat.com":{"auth":"fHVoYy1wb29sLWVjOWYxMzI3LTE1YTctNDQ5Yy1iNTczLWVjNzI2MTUxMmEwZDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXhOR0UyTnpjNFptSXdNV0UwT0RFeVlqQXlPRFF3TTJRd09HVmhNakkzTUNKOS5zaVNuQV9aaUtNVDlLcXAxYWtIOVlGMHlGMGdXajY2TVJPa1htVXRPc0FBdDhVLXlXN3k3YVplbG5wNDRhLWJtVndqeG96NlZuLUxlMzZna2Z4YkVyYmlwOThZRFc4eWxuMGFVUDNsa0JJS2g4M3NycXk2YklRdnoyTWJPNUd6Z3FTblUzTGpYdWx0N2ZvRUZ6OXpUOTItcnpHRWZwTlNxSzdhczlmbFhWcTlkWU4tRDRLZmpuR0c5Zkhna3BmZDBrNDlvcmhiZWhjQlJlUVV1Y24wV2tocmNoMUNxS0RrM095WmlxOEpZTEtYRXBUbTc0TE0waVNYUFVXXzdDdGJoNkVkUjlLRTZIQzl4MXM3ek9CUkFORk9aM1Z3OUxyd3NKeTd1NzEyWVJCOXUzeUZzc3JrVjczZFUzZ2ppOFhmLU5oNE1CMkRMOFFvdHdnUHl1cnFUT1EweHNFR3p2Vkh0SEhBcWRjRmxLUm43RjlqWXhtQzB0MGtLM2VCU1lmTG9XdjBmVmdBQk1Fd3RCLW11WmlhUjNmVVk0RndkYVJocjVGcWNTbjRHc1h1UUc4bXBaVDFncjh4TjI2TUFQMVRHeWZOZ2R5aDRxeFFaY1NnU05jbENlS3Y1Zk5LRVd0T3ZBZnVJZURqZlR5T282V3cyVVRnbzh4c2tJZEl0dk1aZ19tRmFfRy1YWWEzd1lablRGNnF3Z0w1VFJzUGxYTmtXOTBxSXo3VkpWVHZ6UHlPZi1maktNX0FOcHJINWo1R1A0NEt6cWtZTjFfRHdaQlA3WkFDR2NUTnZQRTBLMk44eVNCUllOWGppSmJRdjlxWkJwODlVNzNyZ3Q1ZE5ncW1ycktremEtcVNwRGwtTFI4Zk5KeGJkbXd4czJra21YYXhYZEw1U3VzaHRjMA==","email":"extern.thomas.hirmer@volkswagen.de"},"registry.redhat.io":{"auth":"fHVoYy1wb29sLWVjOWYxMzI3LTE1YTctNDQ5Yy1iNTczLWVjNzI2MTUxMmEwZDpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSXhOR0UyTnpjNFptSXdNV0UwT0RFeVlqQXlPRFF3TTJRd09HVmhNakkzTUNKOS5zaVNuQV9aaUtNVDlLcXAxYWtIOVlGMHlGMGdXajY2TVJPa1htVXRPc0FBdDhVLXlXN3k3YVplbG5wNDRhLWJtVndqeG96NlZuLUxlMzZna2Z4YkVyYmlwOThZRFc4eWxuMGFVUDNsa0JJS2g4M3NycXk2YklRdnoyTWJPNUd6Z3FTblUzTGpYdWx0N2ZvRUZ6OXpUOTItcnpHRWZwTlNxSzdhczlmbFhWcTlkWU4tRDRLZmpuR0c5Zkhna3BmZDBrNDlvcmhiZWhjQlJlUVV1Y24wV2tocmNoMUNxS0RrM095WmlxOEpZTEtYRXBUbTc0TE0waVNYUFVXXzdDdGJoNkVkUjlLRTZIQzl4MXM3ek9CUkFORk9aM1Z3OUxyd3NKeTd1NzEyWVJCOXUzeUZzc3JrVjczZFUzZ2ppOFhmLU5oNE1CMkRMOFFvdHdnUHl1cnFUT1EweHNFR3p2Vkh0SEhBcWRjRmxLUm43RjlqWXhtQzB0MGtLM2VCU1lmTG9XdjBmVmdBQk1Fd3RCLW11WmlhUjNmVVk0RndkYVJocjVGcWNTbjRHc1h1UUc4bXBaVDFncjh4TjI2TUFQMVRHeWZOZ2R5aDRxeFFaY1NnU05jbENlS3Y1Zk5LRVd0T3ZBZnVJZURqZlR5T282V3cyVVRnbzh4c2tJZEl0dk1aZ19tRmFfRy1YWWEzd1lablRGNnF3Z0w1VFJzUGxYTmtXOTBxSXo3VkpWVHZ6UHlPZi1maktNX0FOcHJINWo1R1A0NEt6cWtZTjFfRHdaQlA3WkFDR2NUTnZQRTBLMk44eVNCUllOWGppSmJRdjlxWkJwODlVNzNyZ3Q1ZE5ncW1ycktremEtcVNwRGwtTFI4Zk5KeGJkbXd4czJra21YYXhYZEw1U3VzaHRjMA==","email":"extern.thomas.hirmer@volkswagen.de"}}}

### DNS Requirements
Resolves to ${INT_LB} 
```
api.<cluster_name>.<domain_name>
api-int.<cluster_name>.<domain_name>
```
Resolves to ${INT_APPS_LB}
```
*.apps.<cluster_name>.<domain_name>
```

Note: If wildcard resolution is not permitted add this prior to installation, pointing to ${INT_APPS_LB}

```
oauth-openshift.apps.<cluster_name>.<domain_name>
console-openshift-console.apps.<cluster_name>.<domain_name>
downloads-openshift-console.apps.<cluster_name>.<domain_name>
default-route-openshift-image-registry.apps.<cluster_name>.<domain_name>
alertmanager-main-openshift-monitoring.apps.<cluster_name>.<domain_name>
grafana-openshift-monitoring.apps.<cluster_name>.<domain_name>
prometheus-k8s-openshift-monitoring.apps.<cluster_name>.<domain_name>
thanos-querier-openshift-monitoring.apps.<cluster_name>.<domain_name>
```

1. *Create Identity Acct and assign "Contributor" role to RG \
`az ad sp create-for-rbac --role Contributor -g ${RESOURCE_GROUP} --name ${MAN_ID}`

    ```
    export PRINCIPAL_ID=`az identity show -g ${RESOURCE_GROUP} -n ${MAN_ID} --query principalId --out tsv` 

    export RESOURCE_GROUP_ID=`az group show -g ${RESOURCE_GROUP} --query id --out tsv`
    ``` 

    `az role assignment create --assignee "${PRINCIPAL_ID}" --role 'Contributor' --scope "${RESOURCE_GROUP_ID}"`

2. Create private dns zone = "<cluster_name>.<domain_name>" for local lookups within the RG \
`az network private-dns zone create -g ${RESOURCE_GROUP} -n ${PRVT_DNS}`

3. *Link Private DNS Zone to VNET \
`az network private-dns link vnet create -g ${RESOURCE_GROUP} -z ${PRVT_DNS} -n ${CLUSTER_NAME}-network-link -v ${VNET} -e false`

4. Create StorageSA account \
`az storage account create -g ${RESOURCE_GROUP} --location ${AZURE_REGION} --name ${STORAGE_SA} --kind Storage --sku Standard_LRS`


## INSTALLATION PROCESS

Login with your SPN \
`az login --service-principal --username APP_ID --password PASSWORD --tenant TENANT_ID`

1. Create install-config (Optional, use included customized "install-config.yaml") \
`openshift-install create install-config —dir=$INSTALL`

2. Create manifests \
`openshift-install create manifests --dir=$INSTALL`

3. Backup worker machine-config \
`cp $INSTALL/openshift/99_openshift-cluster-api_worker-machineset-0.yaml ~/`

4. Set IngressController to "HostNetwork" / replicas: 3 \
`vi $INSTALL/manifests/cluster-ingress-default-ingresscontroller.yaml`
    ```
    apiVersion: operator.openshift.io/v1
    kind: IngressController
    metadata:
    name: default
    namespace: openshift-ingress-operator
    spec:
    endpointPublishingStrategy:
        type: HostNetwork
        replicas: 3
    ```
5. Delete machine-configs \
`rm -f $INSTALL/openshift/99_openshift-cluster-api_master-machines-*.yaml` \
`rm -f $INSTALL/openshift/99_openshift-cluster-api_worker-machineset-*.yaml`

6. Record resource group name for $INFRA_ID \
    export INFRA_ID=`grep infrastructureName $INSTALL/manifests/cluster-infrastructure-02-config.yml | awk -F":" '{gsub (" ", "", $0); print $2}'`
    
7. Create ignition files \
`openshift-install create ignition-configs --dir=${INSTALL}`

8. Create private api load balancing
    ```
    az deployment group create -g ${RESOURCE_GROUP} \
    --template-file "adv_infra.json" \
    --parameters privateDNSZoneName="${PRVT_DNS}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters serviceSubnetName="${WORKER_SUB}" \
    --parameters internalLoadBalancerIP="${INT_LB}" \
    --parameters internalAppsLoadBalancerIP="${INT_APPS_LB}"
    ```
9. Export storage account key and vhd_url \
export ACCOUNT_KEY=`az storage account keys list -g ${RESOURCE_GROUP} --account-name ${STORAGE_SA} --query "[0].value" -o tsv` \
export VHD_URL=`curl -s https://raw.githubusercontent.com/openshift/installer/release-4.6/data/data/rhcos.json | jq -r .azure.url`

10. Create a storage container for the rhcos.vhd \
`az storage container create --name vhd --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY}`

11. Copy rhcos.vhd into the container \
`az storage blob copy start --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} --destination-blob "rhcos.vhd" --destination-container vhd --source-uri "${VHD_URL}"`
12. While Loop, wait until completed before proceeding
    ```
    status="unknown"
    while [ "$status" != "success" ]
    do
    status=`az storage blob show --container-name vhd --name "rhcos.vhd" --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -o tsv --query properties.copy.status`
    echo $status
    done
    ```
13. Create the storage container to host the bootstrap.ign file \
`az storage container create --name files --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} --public-access blob`

14. Copy bootstrap.ign into the container \
`az storage blob upload --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c "files" -f "$INSTALL/bootstrap.ign" -n "bootstrap.ign"`

15. Export the RHCOS VHD \
export VHD_BLOB_URL=`az storage blob url --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c vhd -n "rhcos.vhd" -o tsv`

16. Deploy rhcos image def_storage.json \
`az deployment group create -g ${RESOURCE_GROUP} --template-file "def_storage.json" --parameters vhdBlobURL="${VHD_BLOB_URL}" --parameters baseName="${INFRA_ID}"`

17. Creating the bootstrap \
export BOOTSTRAP_URL=`az storage blob url --account-name ${STORAGE_SA} --account-key ${ACCOUNT_KEY} -c "files" -n "bootstrap.ign" -o tsv` \
export BOOTSTRAP_IGNITION=`jq -rcnM --arg v "3.1.0" --arg url ${BOOTSTRAP_URL} '{ignition:{version:$v,config:{replace:{source:$url}}}}' | base64 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_bootstrap.json" \
    --parameters bootstrapIgnition="${BOOTSTRAP_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters identityName="${MAN_ID}"
    ```

18. Creating the masters \
export MASTER_IGNITION=`cat $INSTALL/master.ign | base64 -w0 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_masters.json" \
    --parameters masterIgnition="${MASTER_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters privateDNSZoneName="{PRVT_DNS}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters masterSubnetName="${MASTER_SUB}" \
    --parameters identityName="${MAN_ID}"
    ```
19. Monitor bootstrap process \
`openshift-install wait-for bootstrap-complete --dir=$INSTALL  --log-level info`

20. Creating workers \
export WORKER_IGNITION=`cat $INSTALL/worker.ign | base64 -w0 | tr -d '\n'`
    ```
    az deployment group create -g ${RESOURCE_GROUP} --template-file "adv_workers.json" \
    --parameters workerIgnition="${WORKER_IGNITION}" \
    --parameters sshKeyData="${SSH_KEY}" \
    --parameters baseName="${INFRA_ID}" \
    --parameters virtualNetworkResourceGroup="${VNET_RG}" \
    --parameters virtualNetworkName="${VNET}" \
    --parameters identityName="${MAN_ID}" \
    --parameters serviceSubnetName="${WORKER_SUB}"
    ```
21. Approve csr's \
`watch "oc get csr | grep Pending"` \
`oc adm certificate approve ...... `

22. Remove Bootstrap Resources
    ```
    az vm stop -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
    az vm deallocate -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap
    az vm delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap --yes
    az disk delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap_OSDisk --no-wait --yes
    az network nic delete -g ${RESOURCE_GROUP} --name ${INFRA_ID}-bootstrap-nic --no-wait
    az storage blob delete --account-key ${ACCOUNT_KEY} --account-name ${STORAGE_SA} --container-name files --name bootstrap.ign
    ```
23. Add workers to both backend load balancer pools and wait for services to come online. \
`watch oc get co`