># Implementing Azure Oauth2 Integration with Kafka UI RBAC

<br/>

* 1. [Prerequisites](#Prerequisites)
* 2. [Standard Variables](#StandardVariables)
* 3. [Create Entra Groups for Kafka UI Roles](#CreateEntraGroupsforKafkaUIRoles)
* 4. [Add Users to the Entra Groups](#AddUserstotheEntraGroups)
* 5. [Create and configure a Service Principal for the Kafka UI](#CreateandconfigureaServicePrincipalfortheKafkaUI)
* 6. [Update the App Registration Manifest](#UpdatetheAppRegistrationManifest)
* 7. [Update the App Roles on the App Registration](#UpdatetheAppRolesontheAppRegistration)
* 8. [Remove existing App Role Assignments on the Service Principal](#RemoveexistingAppRoleAssignmentsontheServicePrincipal)
* 9. [Add the App Role Assignments to the Service Principal](#AddtheAppRoleAssignmentstotheServicePrincipal)
	* 9.1. [KafkaUI.Admins](#KafkaUI.Admins)
	* 9.2. [KafkaUI.ReadOnly](#KafkaUI.ReadOnly)
* 10. [Configure the Kafka UI YAML File for Kubernetes](#ConfiguretheKafkaUIYAMLFileforKubernetes)
* 11. [Deploy the Kafka UI App to the AKS Cluster](#DeploytheKafkaUIApptotheAKSCluster)

<br/>

TL;DR - The Kafbat Team are busting their tails making Kafbat better so I decided to write-up a Step-by-Step process on how to configure Azure Oauth2 Integration with Kafka UI RBAC running on an AKS Cluster.

<br/>

> **IMPORTANT!: The instructions below are for educational purposes only. Careful consideration should be taken before using any of this in a Production Envrionment. I am not liable for where or how you decide to use this content. I am not a cat.**

<br/>

> NOTE: Please be aware of the following before continuing.

- The instructions below are for Kafka UI RBAC associated with Azure Entra Groups.
- The instructions below use Azure CLI.
- The instructions below have only been tested in a Linux Environment. The documentation below should, **in theory**, work in a Windows Subsystem for Linux Environment.
- The instructions below may not work if you don't have at least **Contributor** Access or higher on the target Azure Subscription.
- Anywhere the term **Service Principal** is used, you can view the same information in the Azure Portal by looking for **Enterprise Application**.

<br/>

<br/>

##  1. <a name='Prerequisites'></a>Prerequisites

Make sure you have the following before continuing.

- Access to Azure CLI in a **bash** Environment.
  - The following linux tools installed.
    - jq
    - uuidgen
- Access to an existing Azure Event Hub.
- Access to existing, or the ability to create, Security Groups in Azure.
- Access to existing, or the ability to create, Users in Azure.

<br/>

##  2. <a name='StandardVariables'></a>Standard Variables

Variables that will be utilized throughout the documentation are listed below. Set them according to your Environment or search/replace the existing variables with new values as you see fit.

```bash
aksName="dev-aks"
aksFqdn="dev-aks.contoso.local"
azSubId="9cb2f87a-130b-4c04-a7ad-82e3a217f989"
azTenantId="6bfa42d5-d581-41a9-8cee-f855005cfe63"
eventHubsKvName="kv-dev-event-hubs"
eventHubsNamespaceName="eh-dev-event-hubs-ns-001"
eventHubsRgName="rg-dev-event-hubs"
kafkaUiSpName="sp-dev-kafka-ui"
kafkaUiSpCredsSecretName="sp-dev-kafka-ui-password"
office365Fqdn="contoso"
```

</br>

##  3. <a name='CreateEntraGroupsforKafkaUIRoles'></a>Create Entra Groups for Kafka UI Roles

Below, two Entra Groups are being created, one for Kafka UI **Admins** and one for Kafka UI **Read-Only** Users. You can read more on creating your own customized Roles [here](https://ui.docs.kafbat.io/configuration/rbac-role-based-access-control).


```bash
# Creating Kafka UI Admins Group.
az ad group create \
--display-name sec-dev-kafka-ui-admins \
--mail-nickname sec-dev-kafka-ui-admins \
--description "Members have Admin Access to the Kafka UI at https://my-private-aks-cluster.local/kafka-ui/" \
--only-show-errors \
--output none

# Creating Kafka UI Read-Only Group.
az ad group create \
--display-name sec-dev-kafka-ui-ro \
--mail-nickname sec-dev-kafka-ui-ro \
--description "Members have Read-Only Access to the Kafka UI at https://my-private-aks-cluster.local/kafka-ui/" \
--only-show-errors \
--output none
```

<br/>

Next, run the following command to retrive the Object IDs of each Group you've created.

```bash
az ad group list \
--query "[?contains(displayName,'sec-dev-kafka-ui')].{displayName:displayName, objectId:id}" \
--output table | sort -n
```

</br>

You should get back a similar response.

```text
DisplayName                         ObjectId
--------------------------  ------------------------------------
sec-dev-kafka-ui-admins     ee66854f-9f96-4dc1-a53f-2bf891b35026
sec-dev-kafka-ui-read-only  691b9aa1-cc4d-4f0b-8592-09dfe00b9daf
```

<br/>

Next, set the values respectively to the following variables below as they will be used later.

```bash
# Object ID of the Kafka UI Admins Entra Security Group.
kafkaUiAppRoleAdminsSecGroupId="ee66854f-9f96-4dc1-a53f-2bf891b35026"

# Object ID of the Kafka UI Read-Only Entra Security Group.
kafkaUiAppRoleReadOnlySecGroupId="691b9aa1-cc4d-4f0b-8592-09dfe00b9daf"
```

<br/>

Make note of the **Object IDs** above as they will be used later on.

<br/>

##  4. <a name='AddUserstotheEntraGroups'></a>Add Users to the Entra Groups

Add Users to their required group for the Role you want assigned for them in the [Azure Portal](https://portal.azure.com). You can, of course, use Azure CLI (or other tools) to perform this activity.

</br>

##  5. <a name='CreateandconfigureaServicePrincipalfortheKafkaUI'></a>Create and configure a Service Principal for the Kafka UI

Run the following command to create a Service Principal for the Kafka UI.

```bash
az ad sp create-for-rbac \
--role "Reader" \
--name $kafkaUiSpName \
--years 5 \
--scopes "/subscriptions/$azSubId/resourceGroups/$eventHubsRgName/providers/Microsoft.EventHub/namespaces/$eventHubsNamespaceName" \
--only-show-errors \
--output none
```

</br>

Next, retrieve the Object Id of the Service Principal.

```bash
spId=$(az ad sp list \
--all \
--display-name $kafkaUiSpName \
--query [].id \
--output tsv)
```

</br>

Next, reset the Password for the Service Principal.

```bash
spPassword=$(az ad sp credential reset \
--display-name $kafkaUiSpName \
--id $spId \
--years 5 \
--query password \
--only-show-errors \
--output tsv)
```

<br/>

Add the Kafka UI Service Principal Credentials to a Secret in a Key Vault.

```bash
az keyvault secret set \
--name "$kafkaUiSpCredsSecretName" \
--vault-name "$eventHubsKvName" \
--value "$spPassword" \
--output none
```

<br/>

##  6. <a name='UpdatetheAppRegistrationManifest'></a>Update the App Registration Manifest

The following values on the App Registration must be updated.

- Identifier URIs
- Access Token Accepted Version (requestedAccessTokenVersion)
- API Permissions
- Group Membership Claims
- Optional Claims
- Redirect URIs

<br/>

In order to achieve this, with minimal errors, the **az rest** command is used to make API REST calls to update the App Registration Manifest, with the exception of the **Identifier URIs**.

<br/>

<br/>

Retrieve the Application ID of the Service Principal.

```bash
adAppId=$(az ad app list \
--display-name $kafkaUiSpName \
--query [].id \
--output tsv)
```

<br/>

Update the Application URI of the Service Principal.

```bash
az ad app update \
--id $adAppId \
--identifier-uris https://$office365Fqdn.onmicrosoft.com/$kafkaUiSpName
```

<br/>

Next, create a new file called **kafka-ui-access-token-accepted-version.json** and populate it with the following values.

```bash
cat <<- EOF > kafka-ui-access-token-accepted-version.json
{
    "api": {
        "requestedAccessTokenVersion": "2"
    }
}
EOF
```

<br/>

Next, Update the **Access Token Accepted Version** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body @kafka-ui-access-token-accepted-version.json
```

<br/>

Next, create a new file called **kafka-ui-api-permissions.json** and populate it with the following values.

```bash
cat <<- EOF > kafka-ui-api-permissions.json
{
    "requiredResourceAccess": [
        {
            "resourceAppId": "00000003-0000-0000-c000-000000000000",
            "resourceAccess": [
                {
                    "id": "14dad69e-099b-42c9-810b-d002981feec1",
                    "type": "Scope"
                },
                {
                    "id": "64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0",
                    "type": "Scope"
                },
                {
                    "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
                    "type": "Scope"
                },
                {
                    "id": "5b567255-7703-4780-807c-7be8301ae99b",
                    "type": "Role"
                }
            ]
        }
    ]
}
EOF
```

<br/>

The mapping of these permissions can be found in the [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference#all-permissions).

Below are the Scope/Roles and their IDs that are in use above.

| Name            | Permission Type | Description                                                                                                               | Identifier                           |
|-----------------|-----------------|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| profile         | Delegated       | Allows the app to see your users' basic profile (e.g., name, picture, user name, email address)                           | 14dad69e-099b-42c9-810b-d002981feec1 |
| email           | Delegated       | Allows the app to read your users' primary email address                                                                  | 64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0 |
| User.Read       | Delegated       | Allows users to sign-in to the app, and allows the app to read the profile of signed-in users.                            | e1fe6dd8-ba31-4d61-89e7-88639da4683d |
| Group.Read.All  | Delegated       | Allows the app to read group properties and memberships, and read conversations for all groups, without a signed-in user. | 5b567255-7703-4780-807c-7be8301ae99b |

<br/>

Next, Update the **API Permissions** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body kafka-ui-api-permissions.json
```

<br/>

Next, create a new file called **kafka-ui-group-membership-claims.json** and populate it with the following values. 

```bash
cat <<- EOF > kafka-ui-group-membership-claims.json
{
    "groupMembershipClaims": "ApplicationGroup"
}
EOF
```

<br/>

Next, Update the **Group Membership Claims** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body kafka-ui-group-membership-claims.json
```

<br/>

Next, create a new file called **kafka-ui-optional-claims.json** and populate it with the following values. 

```bash
cat <<- EOF > kafka-ui-optional-claims.json
{
    "optionalClaims": {
        "idToken": [
            {
                "name": "groups",
                "source": null,
                "essential": false,
                "additionalProperties": []
            }
        ],
        "accessToken": [
            {
                "name": "verified_primary_email",
                "source": null,
                "essential": false,
                "additionalProperties": []
            },
            {
                "name": "email",
                "source": null,
                "essential": false,
                "additionalProperties": []
            },
            {
                "name": "upn",
                "source": null,
                "essential": false,
                "additionalProperties": []
            },
            {
                "name": "groups",
                "source": null,
                "essential": false,
                "additionalProperties": []
            }
        ],
        "saml2Token": [
            {
                "name": "groups",
                "source": null,
                "essential": false,
                "additionalProperties": []
            }
        ]
    }
}
EOF
```

<br/>

Next, Update the **Optional Claims** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body kafka-ui-optional-claims.json
```

<br/>

Next, create a new file called **kafka-ui-redirect-uris.json** and populate it with the following values. 

```bash
cat <<- EOF > kafka-ui-redirect-uris.json
{
    "web": {
        "redirectUris": [
          "https://$aksFqdn/kafka-ui/login/oauth2/code/azure"
        ]
    }
}
EOF
```

> NOTE: The first part of the **redirectUris** value (*https://dev-aks.contoso.local/kafka-ui/*) is the location where your **Kafka UI** App is being hosted. Adjust these values according to your environment.

<br/>

Next, Update the **Redirect URIs** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body kafka-ui-redirect-uris.json
```

<br/>





##  7. <a name='UpdatetheAppRolesontheAppRegistration'></a>Update the App Roles on the App Registration

Next, we need to disable and remove the App Roles, if they exist, before attempting to update then from templates further down. The reason for doing this is [here](https://stackoverflow.com/questions/43517110/deleting-an-applications-approle-in-azure-active-directory).

<br/>

Next, run the command to retrieve the existing JSON Manifest of the App Registration and save it to a variable where **App Roles** are disabled.

```bash
setAppRolesToDisabled=$(az ad app show --id $adAppId | jq -r '.appRoles[].isEnabled = false')
```

<br/>

Next, apply the updated JSON Manifest against the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body "$setAppRolesToDisabled"
```

<br/>

Next, run the command to retrieve the existing JSON Manifest of the App Registration and save it to a variable where **App Roles** are removed.

```bash
removeExistingAppRoles=$(az ad app show --id $adAppId | jq -r '.appRoles=[]')
```

<br/>

Next, apply the updated JSON Manifest against the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body "$removeExistingAppRoles"
```

<br/>

Next, generate GUIDs for the **KafkaUI.Admins** and **KafkaUI.ReadOnly** App Roles.

```bash
kafkaUiAdminsAppRoleGuid=$(uuidgen)
kafkaUiReadOnlyAppRoleGuid=$(uuidgen)
```

<br/>


Next, create a new file called **kafka-ui-app-roles.json** and populate it with the following values. The GUIDs you just generated for the **KafkaUI.Admins** and **KafkaUI.ReadOnly** App Roles will be added through variable expansion.

<br/>

```bash
cat <<- EOF > kafka-ui-app-roles.json
{
    "appRoles": [
        {
            "allowedMemberTypes": [
                "User"
            ],
            "description": "Users have Admin access to the Kafka UI",
            "displayName": "Kafka UI Admins",
            "id": "$kafkaUiAdminsAppRoleGuid",
            "isEnabled": "true",
            "origin": "Application",
            "value": "KafkaUI.Admins"
        },
        {
            "allowedMemberTypes": [
                "User"
            ],
            "description": "Users have Read-Only access to the Kafka UI",
            "displayName": "Kafka UI Read-Only",
            "id": "$kafkaUiReadOnlyAppRoleGuid",
            "isEnabled": "true",
            "origin": "Application",
            "value": "KafkaUI.ReadOnly"
        }
    ]
}
EOF
```

<br/>

Next, Update the **App Roles** on the App Registration.

```bash
az rest \
--method PATCH \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/applications/$adAppId" \
--body kafka-ui-app-roles.json
```

<br/>

##  8. <a name='RemoveexistingAppRoleAssignmentsontheServicePrincipal'></a>Remove existing App Role Assignments on the Service Principal

This section can be skipped if you are only going to deploy this once. However, if you are going to include this process as part of a CI/CD Pipeline, you want to remove any existing App Role Assignments so you aren't creating duplicates every time your CI/CD Pipeline is ran.

<br/>

Next, retrieve any existing **App Role Assignments** configured for the Service Principal.

```bash
existingAppRoles=$(az rest \
--method GET \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/servicePrincipals/$spId/appRoleAssignedTo" \
--query "value[].join(',', [id,principalDisplayName])" \
--output tsv | sed 's/ /_/g')
```

<br/>

Next, delete any existing **App Role Assignments** for the Service Principal.

```bash
for role in $existingAppRoles
do
    roleId=$(echo $role | awk -F ',' '{print $1}')
    principalDisplayName=$(echo $role | awk -F ',' '{print $2}')

    az rest \
    --method DELETE \
    --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$spId/appRoleAssignedTo/$roleId" \
    --output none && \
    echo "Deleted App Role Assignment for [$principalDisplayName], Role ID [$roleId]."
done
```

<br/>

##  9. <a name='AddtheAppRoleAssignmentstotheServicePrincipal'></a>Add the App Role Assignments to the Service Principal

In this section, the **KafkaUI.Admins** and **KafkaUI.ReadOnly** App Role Assignments are added to the Service Principal.

<br/>

###  9.1. <a name='KafkaUI.Admins'></a>KafkaUI.Admins

Next, retrieve the App Role IDs of the **KafkaUI.Admins** App Role.

```bash
kafkaUiAppRoleAdminsId=$(az ad app show \
--id $adAppId \
--query "appRoles[?value=='KafkaUI.Admins'].id" \
--output tsv)
```
<br/>

Next, create a new file called **kafka-ui-app-role-assignment-admins.json** and populate it with the following values. 

```bash
cat <<- EOF > kafka-ui-app-role-assignment-admins.json
{
    "principalId": "$kafkaUiAppRoleAdminsSecGroupId",
    "resourceId": "$spId",
    "appRoleId": "$kafkaUiAppRoleAdminsId"
}
EOF
```

<br/>

Next, add the Kafka UI **Admins** App Role Assignment to the Service Principal.

```bash
az rest \
--method POST \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/servicePrincipals/$spId/appRoleAssignedTo" \
--body kafka-ui-app-role-assignment-admins.json \
--output none
```

<br/>

###  9.2. <a name='KafkaUI.ReadOnly'></a>KafkaUI.ReadOnly

Next, retrieve the App Role IDs of the **KafkaUI.ReadOnly** App Role.

```bash
kafkaUiAppRoleReadOnlyId=$(az ad app show \
--id $adAppId \
--query "appRoles[?value=='KafkaUI.ReadOnly'].id" \
--output tsv)
```

<br/>

Next, create a new file called **kafka-ui-app-role-assignment-read-only.json** and populate it with the following values. 

```bash
cat <<- EOF > kafka-ui-app-role-assignment-read-only.json
{
    "principalId": "$kafkaUiAppRoleReadOnlySecGroupId",
    "resourceId": "$spId",
    "appRoleId": "$kafkaUiAppRoleReadOnlyId"
}
EOF
```

<br/>

Next, add the Kafka UI **Read-Only** App Role Assignment to the Service Principal.

```bash
az rest \
--method POST \
--headers "Content-Type=application/json" \
--uri "https://graph.microsoft.com/v1.0/servicePrincipals/$spId/appRoleAssignedTo" \
--body kafka-ui-app-role-assignment-read-only.json \
--output none
```

<br/>

##  10. <a name='ConfiguretheKafkaUIYAMLFileforKubernetes'></a>Configure the Kafka UI YAML File for Kubernetes

Retrive the **primaryConnectionString** for the Event Hubs Namespace.

```bash
eventHubsNamespaceConnectionString=$(az eventhubs namespace authorization-rule keys list \
--namespace-name $eventHubsNamespaceName \
--resource-group $eventHubsRgName \
--authorization-rule-name RootManageSharedAccessKey \
--query primaryConnectionString \
--only-show-errors \
--output tsv)
```

<br/>

Retrieve the **Application Id** of the Service Principal.

```bash
kafkaUiAppRegAppId=$(az ad app list \
--display-name $kafkaUiSpName \
--query [].appId \
--output tsv)
```

<br/>

Retrieve the Kafka UI App Client Password from the Azure Key Vault where you added it earlier. If you didn't store it earlier, you can just continue to use the value in the **$spPassword** variable.

```bash
spPassword=$(az keyvault secret show \
--name "$kafkaUiSpCredsSecretName" \
--vault-name "$eventHubsKvName" \
--query value \
--output tsv)
```

<br/>

<br/>

Next, create a new file called **kafka-ui.yaml** and populate it with the following values. The Variables below will be replaced through variable expansion.

<br/>

```bash
cat <<- EOF > kafka-ui.yaml
---
# [kafka-ui] - Service
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui
  namespace: kafka-ui
  labels:
    app: kafka-ui
    service: kafka-ui
spec:
  type: ClusterIP
  selector:
    app: kafka-ui
  ports:
  - name: http-kafka-ui
    port: 8080
    targetPort: 8080
---
# [kafka-ui] - Virtual Service
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kafka-ui-route
  namespace: kafka-ui
spec:
  hosts:
  - '*'
  gateways:
  - istio-system/dev-apps-gateway
  http:
  - match:
    - uri:
        prefix: /kafka-ui/
    rewrite:
      uri: /kafka-ui/
    route:
    - destination:
        host: kafka-ui.kafka-ui.svc.cluster.local
        port:
          number: 8080
---
# [kafka-ui] - Destination Rule
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: kafka-ui-destination-rule
  namespace: kafka-ui
spec:
  host: kafka-ui.kafka-ui.svc.cluster.local
---
# [kafka-ui] - Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kafka-ui
  namespace: kafka-ui
---
# [kafka-ui] - Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui-deployment
  namespace: kafka-ui
  labels:
    app: kafka-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      serviceAccountName: kafka-ui
      containers:
      - name: kafka-ui
        #image: "provectuslabs/kafka-ui:latest"
        image: "kafbat/kafka-ui:v1.0.0"
        env:
        - name: SPRING_CONFIG_ADDITIONAL-LOCATION
          value: "kui-config/config.yml"
        - name: LOGGING_LEVEL_ROOT
          value: "INFO"
        - name: SERVER_SERVLET_CONTEXT_PATH
          value: "/kafka-ui"
        ports:
        - containerPort: 8080
        - name: liveness-port
          containerPort: 8080
        volumeMounts:
        - name: kafka-ui-configmap
          mountPath: /kui-config
      volumes:
        - name: kafka-ui-configmap
          configMap:
            name: kafka-ui-configmap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-ui-configmap
  namespace: kafka-ui
data:
  config.yml: |-
    logging:
      level:
        io.kafbat.ui.service.rbac.extractor: TRACE
    kafka:
      clusters:
      - name: $eventHubsNamespaceName
        bootstrapServers: $eventHubsNamespaceName.servicebus.windows.net:9093
        properties:
          sasl:
            jaas:
              config: org.apache.kafka.common.security.plain.PlainLoginModule required username='\$ConnectionString' password='$eventHubsNamespaceConnectionString';
            mechanism: PLAIN
            kerberos:
              service:
                name: kafka
          security:
            protocol: SASL_SSL
    auth:
      type: OAUTH2
      oauth2:
        client:
          azure:
            clientId: "$kafkaUiAppRegAppId"
            clientSecret: "$spPassword"
            scope:
            - openid
            - email
            client-name: azure
            provider: azure
            redirect-uri: https://$aksFqdn/kafka-ui/login/oauth2/code/azure
            authorization-grant-type: authorization_code
            issuer-uri: https://login.microsoftonline.com/$azTenantId/v2.0
            jwk-set-uri: https://login.microsoftonline.com/$azTenantId/discovery/v2.0/keys
            user-name-attribute: email
            custom-params:
              type: oauth
              roles-field: roles
    rbac:
      roles:
        - name: "KafkaUI.Admins"
          clusters:
            - $eventHubsNamespaceName
          subjects:
            - provider: oauth
              type: role
              value: "KafkaUI.Admins"
          permissions:
            - resource: applicationconfig
              actions: all
            - resource: clusterconfig
              actions: all
            - resource: topic
              value: ".*"
              actions: all
            - resource: consumer
              value: ".*"
              actions: all
            - resource: schema
              value: ".*"
              actions: all
            - resource: connect
              value: ".*"
              actions: all
            - resource: ksql
              actions: all
            - resource: acl
              actions: [ "view" ]
        - name: "KafkaUI.ReadOnly"
          clusters:
            - $eventHubsNamespaceName
          subjects:
            - provider: oauth
              type: role
              value: "KafkaUI.ReadOnly"
          permissions:
            - resource: clusterconfig
              actions: [ "view" ]
            - resource: topic
              value: ".*"
              actions: 
                - VIEW
                - MESSAGES_READ
            - resource: consumer
              value: ".*"
              actions: [ view ]
            - resource: schema
              value: ".*"
              actions: [ view ]
            - resource: connect
              value: ".*"
              actions: [ view ]
            - resource: acl
              actions: [ view ]
---
EOF
```

<br/>

##  11. <a name='DeploytheKafkaUIApptotheAKSCluster'></a>Deploy the Kafka UI App to the AKS Cluster

<br/>

Apply the Kafka UI Configuration to the AKS Cluster

```bash
kubectl apply -f ./kafka-ui.yaml
```

<br/>

Restart the Kafka UI Pod to ensure ConfiMap updates are applied.

```bash
kubectl delete po --selector=app=kafka-ui --namespace kafka-ui
```

<br/>

Profit.

<br/>

<br/>
. . .
