After assessing the verbose and debug output I was able to identify that many of the az aks operations rely on access to Azure Active Directory Graph (legacy). To resolve this error, you need to ensure that whichever identity you're using to run the command has permissions to use the following API:

https://graph.windows.net/Application.ReadWrite.OwnedBy

For a Service Principal, you can Add a permission here:

Azure Active Directory > App Registrations > {{service_principal}} > API Permissions

As this API requires admin consent, you need a Global Admin to grant access before it will work. This may not always be possible, so I've outlined a couple of workaround options below.

If you're using your user account, you should check that this API is available for your user identity. This should be available by default for user accounts, but I guess access may have been restricted by a Global Admin?

Workaround (Service Principal)

If you're unable to grant the above permissions to your identity (not all organisations allow API access for Service Principals) and you're using a Service Principal for your AKS Cluster, you can use manual Role Assignments to grant access to the ACR as per the following guide:

https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal

Note that the documented command uses the Application ID for your Service Principal. This also requires access to the Azure Active Directory Graph. As such, you will need to replace --assignee $SERVICE_PRINCIPAL_ID with --assignee-object-id $SERVICE_PRINCIPAL_OBJECT_ID where $SERVICE_PRINCIPAL_OBJECT_ID is the Object ID of the Service Principal, and not the Application ID which we would usually use.

For example:

az role assignment create --assignee-object-id $SERVICE_PRINCIPAL_OBJECT_ID --scope $ACR_REGISTRY_ID --role acrpull
Workaround (Managed Identity)

A better alternative is to use a Managed Identity for AKS, as documented here:

https://docs.microsoft.com/en-us/azure/aks/use-managed-identity

However I found this brings it's own challenges. Firstly, you cannot update an existing cluster to use Managed Identity (requires re-creation). Secondly, if you're using a custom subnet with --vnet-subnet-id $SUBNET_ID, you still need to provide a "dummy" value for both --service-principal and --client-secret

For example:

az aks create \
    --resource-group $RESOURCE_GROUP\
    --name $AKS_CLUSTER_NAME \
    --vm-set-type VirtualMachineScaleSets \
    --load-balancer-sku standard \
    --location northeurope \
    --kubernetes-version $AKS_VERSION \
    --network-plugin azure \
    --vnet-subnet-id $SUBNET_ID \
    --service-cidr 10.2.0.0/24 \
    --dns-service-ip 10.2.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --enable-managed-identity \
    --service-principal msi \
    --client-secret null \
    --generate-ssh-keys
Hopefully this helps someone reading this issue, but ideally it would be great to have the az aks commands updated to remove reliance on the Azure Active Directory Graph.
