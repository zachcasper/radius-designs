# CableCo Radius Day 1 User Stories

This document is a follow-up to the CableCo Background and Radius Deployment document. It describes an end-to-end user journey for day 1 based on the CableCo deployment scenario. There will be a follow-up document for the day 2 user journey. This is not intended to be a generic feature spec. It intentionally builds upon the CableCo deployment architecture and is specific to CableCo. Throughout the document, new feature spec documents are identified which will need to be developed.

## Features

After the completion of the user stores below, feature gaps were identified and prioritized in the table below.

| Priority | Size | Feature                                                      |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | XL   | **UDT:** Ability to create resource types via `rad resource-type create` without the need to create a resource provider |
| p1       | L    | **UDT**: Child resources                                     |
| p1       | M    | **UDT:** Modify recipe registration to be specific to resource type version |
| p2       | S    | **UDT:** Parameter data validation                           |
| p1       | M    | **RBAC:**  Role definitions controlling applications, application resources, environments, secrets, and credentials in a resource group |
| p1       | M    | **RBAC:**  Role definitions granting permission to deploy resources to another resource group without CRUDL |
| p1       | L    | **RBAC:**  Assign role to an existing Kubernetes RoleBinding for a resource group |
| p1       | L    | **Terraform:** Bring your own Terraform backends             |
| p1       | M    | **External Kubernetes**: Register an external Kubernetes cluster via `rad credential register kubeconfig` within a resource group and add it to an environment |
| p1       | L    | **External Kubernetes:** Deploy Kubernetes resources to external cluster when a kubeconfig exists on an environment |
| p2       | M    | **RBAC:**  Role definitions granting permission to CRUDL resource types within a resource type namespace |
| p2       | S    | **Terraform:** Create secrets via `rad secret create`        |
| p2       | S    | **Terraform:** Configure Terraform Git repository via CLI    |
| p2       | M    | **External Kubernetes:** Modify `System.AWS/credentials` to be resource group scoped |
| p3       | L    | **Policy Management:** Extensible policy engine based on OAM syntax with out of the box policies |
| p3       | M    | **RBAC:**  External identity providers including Entra ID, AWS IAM, Okta, etc. |

## Step 1 – Installing Radius

As a Radius administrator, I need to install Radius. CableCo plans to use a single instance of Radius running in a multi-AZ Kubernetes cluster. Since I am going to use only a single tenant, I do not expect to need to configure, or have any awareness or, tenants.

> [!NOTE]
>
> Eventually, Radius should support multiple installation options in addition to Kubernetes. However, we see this far into the future so alternatives to Kubernetes are a low priority. Most likely, an alternative would be a standalone set of containers for advanced users to manually install.

**User Experience**

```bash
# Install Radius on current kubeconfig cluster
rad install kubernetes
```

**Result**

1. Radius is installed in the default kubeconfig Kubernetes cluster in the radius-system namespace
2. The radius-admin role is created in the radius-system namespace
3. The radius-admin role binding is created in the radius-system namespace with the current user added as a subject

**Exceptions**

The operation fails and informs the user interactively if:

* The radius-system namespace exists

**Changes from current implementation**

None

## Step 2 – Create resource group for applications

As a Radius administrator, I need to create a resource group for applications and application resources to be created in as well as set RBAC rules which define who can create applications in that group.

**User Experience**

```bash
# Create enterprise application A resource group 
rad group create ent-app-a
# Create developer role
rad role definition create -f developer-role-definition.yaml
# Assign the developer role to members of the enterprise developers Kubernetes RoleBinding
# Permissions only apply in the enterprise application A resource group
rad role assignment create \
  --assignee rolebinding/enterprise-developers \
  --role developer \
  --scope /planes/radius/CableCo/resourceGroup/ent-app-a
```

The contents of `developer-role-definition.yaml` is similar to:

```yaml
---
name: developer
description: Role to manage applications and application resources within a resource group
actions:
  # Grant permissions to CRUDL applications in the scoped resource group
  - Applications.Core/applications/*
  # Restrict using any resource type except those in the CableCo.App namespace
  - CableCo.App/*
```

> [!CAUTION]
>
> We assume for now that Radius will leverage Kubernetes RBAC. In the future, Radius may have its own configuration to an external identity provider via OIDC. For now, when specifying the assignee, users will prefix the assignee with `rolebinding/` to indicate that the subjects in the Kubernetes RoleBinding are assigned the specified role. This seems super confusing and needs to be further researched and validated with enterprise users.

**Result**

1. An empty resource group is created called end-app-a
2. The developer role is created 
3. The enterprise-developers RoleBinding is validated to exist
4. The developer role is assigned to the enterprise-developers group to manage applications and CableCo-specified resources

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the radius-admin role 

* The resource group already exists

* The role definition or assignment already exists

* The enterprise-developers RoleBinding does not exist in the host Kubernetes cluster


**Changes from current functionality**

This is all new functionality which needs to be documented in a new **RBAC feature spec**.

## Step 3 – Set policies on the application resource group

As a Radius administrator, I need to create a policy which requires applications and application resources to be annotated with a cost center. I also need to restrict overriding the recipe in resource definitions. 

**User Experience**

```bash
# Create require billing code on applications policy in the CableCo tenant
rad policy create -f required-annotations.yaml
# Attach policy to the enterprise application A resource group
rad group policy register --group ent-app-a --policy require-billing-code-on-applications
```

The contents of `required-annotations.yaml` is similar to:

```yaml
---
kind: RadiusRequiredAnnotations
metadata:
  name: require-billing-code-on-applications
spec:
  match:
    kinds:
      - application
  parameters:
    message: All applications much have a billing code annotation.
    annotations:
      - key: billing-code
        # Integer
        allowedRegex: ^[0-9]*$
```



```bash
# Create require billing code on applications policy in the CableCo tenant
rad policy create -f disallow-recipe-overrides.yaml
# Attach policy to the enterprise application A resource group
rad group policy create --group ent-app-a --policy disallow-recipe-on-resources
```

The contents of `disallow-recipe-overrides.yaml` is similar to:

```yaml
---
kind: RadiusDisallowRecipeOnResources
metadata:
  name: disallow-recipe-on-resources
spec:
  match:
    kinds: [""]
  parameters:
    message: The recipe for a resource type cannot be overrided in the resource definition
```

> [!NOTE]
>
> This example is based on OPA Gatekeeper constraints. It is illustrative of a policy syntax that Radius users are likely to be familiar with. This example does not infer that OPA should be used in the implementation.

> [!NOTE]
>
> Today, the only annotations that Radius supports are via the kubernetesMetadata extension. Since Radius will support multiple different container runtime platforms, Radius needs to support a more generic metadata model.

**Result**

1. The require-billing-code-on-applications policy is created in the Radius tenant
2. The policy is attached to the ent-app-a resource group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the radius-admin role 
* The policy already exists
* The policy refers to kinds which do not exist in Radius


**Changes from current functionality**

This is all new functionality which needs to be documented in a new **policy management feature spec**.

## Step 4 – Create resource group for environments

As a Radius administrator, I need to create a resource group for environments in Radius and set RBAC rules which define who can create environments in that group.

**User Experience**

```bash
# Create enterprise non-production resource group
rad group create ent-non-prod
# Create environment administrator role
rad role definition create -f env-admin-role-definition.yaml
# Assign environment administrator role to the cloud engineering Kubernetes RoleBinding
# Permissions only apply in the enterprise non-production resource group
rad role assignment create \
  --assignee rolebinding/cloud-engineering \
  --role env-admin \
  --scope /planes/radius/CableCo/resourceGroup/ent-non-prod
```

The contents of `env-admin-role-definition.yaml` is similar to:

```yaml
---
name: env-admin
description: Role to manage environments within a resource group
actions:
  # Grant permissions to CRUDL environments and secrets in the scoped resource group
  - Applications.Core/environments/*
  - Applications.Core/secretStores/*
  # Grant permissions to configure connections to AWS, Azure, and Kubernetes in the scoped resource group
  - System.AWS/credentials/* 
  - System.Azure/credentials/*
  - System.Kubernetes/credentials/*
```

**Result**

1. An empty resource group is created
2. The env-admin role is created 
3. The cloud engineering RoleBinding is validated to exist
4. The env-admin role is granted to the cloud engineering group to manage environments within the non-prod-env resource group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the radius-admin role 

* The resource group already exists

* The role definition or assignment already exists

* The cloud-engineering RoleBinding does not exist in the host Kubernetes cluster

**Changes from current functionality**

This is all new functionality which needs to be documented in a new **RBAC feature spec**.

## Step 5 – Create the Deployer role

As a Radius administrator, I need to set RBAC rules which define who can deploy applications and resources into these environments.

**User Experience**

```bash
# Create deployer role
rad role definition create -f deployer-role-definition.yaml
# Assign deployer role to the enterprise developers Kubernetes RoleBinding
# Permissions only apply in the enterprise non-production resource group
rad role assignment create \
  --assignee rolebinding/enterprise-developers
  --role deployer
  --scope /planes/radius/CableCo/resourceGroup/ent-non-prod
```

The contents of `deployer-role-definition.yaml` is similar to:

```yaml
---
name: deployer
description: Role granting ability to deploy to an environment
actions:
  # Grant permission to deploy resouces in any resource group to environments within the scoped resource group
  - Applications.Core/environments/deployTo
```

> [!NOTE]
>
> The `deployTo` action is a new verb which grants permission to associate a resource in another resource group to an environment in the resource in scope.

**Result**

1. The deployer role is created 
2. The enterprise developers RoleBinding is validated to exist
3. The deployer role is granted to the the enterprise developers group to allow deploying applications to environments in the scoped resource group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the radius-admin role 
* The role definition or assignment already exists
* The enterprise-developers RoleBinding does not exist

**Changes from current functionality**

This is all new functionality which needs to be documented in a new **RBAC feature spec**.

## Step 6 – Creating Resource Type Admin role

As a Radius administrator, I need to delegate the ability to manage resource types in Radius to my cloud engineering and DBA teams.

**User Experience**

```bash
# Create resource type administrator role
rad role definition create -f resource-type-admin-definition.yaml
# Assign the resource type administrator role to users in the cloud engineering Kubernetes RoleBinding
# Resource types are at the tenant level so the scope is the CableCo tenant
rad role assignment create \
  --assignee rolebinding/cloud-engineering \
  --role resource-type-admin \
  --scope /planes/radius/CableCo/ResourceTypes
```

The contents of `resource-type-admin-definition.yaml` is similar to:

```yaml
---
name: resource-type-admin
description: Role to manage resource types for CableCo
actions:
  # Grant permissions to CRUDL resource types within the CableCo.App and CableCo.Net resource providers
  - System.Resources/resourceproviders/CableCo.App/*
  - System.Resources/resourceproviders/CableCo.Net/*
```

**Result**

1. The resource-type-admin role is created 
2. The cloud engineering RoleBinding is validated to exist
3. The resource-type-admin role is granted to the cloud engineering group to operate on CableCo's resource types

**Exceptions**

The operation fails and informs the user interactively if:

* The user is not a member of the radius-admin role
* The cloud engineering RoleBinding does not exist
* The role definition or assignment already exists

**Changes from current implementation**

This is all new functionality which needs to be documented in a new **RBAC feature spec**.

## Step 7 – Create an environment

As a cloud engineer, I need to create an environment in Radius and point it to a Kubernetes cluster running in CableCo's national data center in AWS. 

**User Experience**

```bash
# Register a kubeconfig cluster and user
rad credential register kubeconfig ent-non-prod-eks-cred \
  --group ent-non-prod \
  --kubeconfig ~/.kube/config \
  --kubectx eks-cluster
# Create a secret in Radius for connecting to AWS APIs
rad credential register aws irsa \
	--iam-role myRoleARN \
  --group ent-non-prod
# Reads the kubeconfig file but not the client certificate or key
rad environment create ent-non-prod-env \
  --group ent-non-prod \
  --namespace ent-non-prod \
  --kubeconfig ent-non-prod-eks-cred
  --aws-region us-east-1 \
  --aws-account-id myAwsAccountId
```

> [!CAUTION]
> This sequence is probably broken if using federated identity.

> [!CAUTION]
>
> This sequence requires a pre-existing Kubernetes cluster since the kubeconfig is passed as an argument. We will need to support the scenario of using a recipe to create a Kubernetes cluster within the environment.

> [!NOTE]
>
> The `rad environment` arguments will change based on what type of environment is being created. In the future, for example, `rad environment` will need to access arguments for serverless platforms and for Google Cloud.

**Result**

1. Connectivity and authentication to the Kubernetes cluster is verified
2. Required permissions on the Kubernetes cluster is confirmed
3. Connectivity and authentication to AWS is verified
4. Required AWS IAM permissions are confirmed
5. An environment is created which points to the Kubernetes cluster and AWS account and region

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the env-admin role for the ent-non-prod resource group 
* The Radius control plane is unable to connect to the Kubernetes cluster specified in the kubeconfig
* The Radius control plane is unable to authenticate to the cluster using the client certificate and key
* The user specified has insufficient permissions on the Kubernetes cluster 
* The Radius control plane is unable to authenticate to AWS
* The user specified has insufficient AWS IAM permissions
* One of the credential names already exists

**Changes from current functionality**

New functionality which needs to be documented in the **external Kubernetes cluster feature spec** including:

1. Enhancements to the environment to support deploying applications to an external Kubernetes cluster
2. New kube credential type which reads the certificate and key from the kubeconfig

## Step 8 – Create foundational resource types

As a cloud engineer, I need to create several foundational resource types which I will deploy to an environment to prepare it to host applications.

**User Experience**

```bash
# Create the resource types in the CableCo.Net namespace
rad resource-type create -f environment.yaml
```

> [!NOTE]
>
> The term namespace is intensionally used here rather than resource provider. While the resource provider component is part of the Radius implementation and may play a part in a service provider use case (such as Azure Resource Manager), it has no meaning in the enterprise use case. Resource providers will remain part of the API and will be referenced when 

> [!NOTE]
>
> CableCo.Net is used here to distinguish infrastructure related resource types from application-related resource types in CableCo.App.

The contents of `environment.yaml` is similar to:

```yaml
--- 
namespace: CableCo.Net
resourceTypes: 
  - CableCo.Net/virtualNetworks
    versions:
      - name: v1
        metadata:
          description: A VPC or VNet with subnets created in each available AZ
        schema:
          properties:
            - cidr-block
              type: string
              description: CIDR block in the form of "10.1.1.0/24"
              # Validate proper CIDR notation
              regex: ^([0-9]{1,3}\.){3}[0-9]{1,3}($|/([1-9]|[12][0-9]|3[012]))$
              required: true
            ...
```

**Result**

1. The CableCo.Net namespace (née resource provider) is created
2. The VirtualNetworks resource type is created with the version and schema specified in the YAML file

**Exceptions**

The operation fails and informs the user interactively if:

* The user is not a member of the resource-type-admin Radius role 
* The resource type with the same version already exists

* The YAML file does not have a version specified

* The schema structure is not in valid

**Changes from current implementation**

This is new functionality which should be documented in the **UDT feature spec** including regular expression validation of parameters.

## Step 9 – Configure Terraform 

As a cloud engineer, I need to configure Terraform in the new environment to use an external Terraform backend and Git repository.

 **User Experience**

```bash
# Create a secret with the Git personal access token in the enterprise non-production resource group
rad secret create ent-non-prod-env-git-pat \
  --group ent-non-prod \
  --type generic \
  --data { pat: { value: $PAT } }
# Grant access for Terraform to the Git repository for the enterprise non-production envirnoment
rad environment update ent-non-prod-env \
  --recipe-config-terraform-authentication-pat-secret ent-non-prod-env-git-pat
# Configure Terraform to use a backend stored in an S3 bucket in the enterprise non-production envirnoment
rad environment update ent-non-prod-env \
  --terraform-backend-type s3 \
  --terraform-backend-s3-bucket cableco-terraform-bucket \
  --terraform-backend-s3-key /ent-non-prod-env
```

**Result**

1. The Git personal access token is stored in the ent-non-prod-env-git-pat secret
1. Access to the S3 bucket is confirmed
1. The environment is updated with the new Terraform configuration

**Exceptions**

The operation fails and informs the user interactively if:

* The user does not have environment admin permission in the resource group
* The ent-non-prod-env-git-pat secret already exists in the enterprise non-production resource group
* Radius does not have access to the S3 bucket specified

**Changes from current functionality**

This is all new functionality which should be documented in the **customizing Terraform feature spec** including:

1. Ability to create secrets using the Radius CLI
2. Ability to configure the Terraform block on the environment via the Radius CLI
3. Ability to customize the Terraform backend via the Radius CLI

## Step 10 – Register recipes

As a cloud engineer, I need to register a recipe in the new environment using an existing Terraform module.

 **User Experience**

```bash
# Register a Terraform module called WebService to fulfill the CableCo.App/webService@v1 resource type in the enterprise non-production environment
rad recipe register WebService \
  --environment ent-non-prod-env \
  --resource-type CableCo.App/webService \
  --resource-type-version v1 \ 
  --template-kind terraform \
  --template-path git::https://github.com/CableCo/terraform-repo \
  --template-version "1.1.0"
```

**Result**

1. Access to the Terraform module is confirmed
1. The Terraform module is registered to fulfill the CableCo.App/webService@v1 resource type (only v1)

**Exceptions**

The operation fails and informs the user interactively if:

* The user does not have environment administrator permissions on the  enterprise non-production resource group
* The enterprise non-production environment does not exist
* Radius does not have access or the URL is invalid to the Terraform module

**Changes from current functionality**

The only change is tying recipes to the version of a resource type. The rationale for this is to enable schema changes on the resource type without breaking the recipe. This should be documented in the **UDT feature spec**.

## Step 11 – Deploy foundational resources

As a cloud engineer, I need to prepare the environment with common infrastructure before being able to host applications.

**User Experience**

```bash
# Create the virtual network in the enterprise non-production environment
rad deploy ent-non-prod-env.bicep --environment ent-non-prod-env 
```

The contents of `ent-non-prod-env.bicep` is similar to:

```
resource network 'CableCo.Net/VirtualNetworks@v1' = {
  name: 'network'
  properties: {
     cidr-block: '10.68.1.120/28'
  }
}
```

**Result**

1. The CIDR block parameter is validated to match the regular expression on the resource type parameter
1. A virtual network is deployed to the enterprise non-production environment

**Exceptions**

The operation fails and informs the user interactively if:

* The network resource already exists
* The CIDR block parameter does not match the regular expression 

**Changes from current functionality**

Resource type parameter validation via regular expressions. This should be documented in the **UDT feature spec**.

## Step 12 – Create composite resource types

As a member of the Enterprise Application Architecture team, I need to create a WebService resource type including the resource schema, metadata such as documentation, and the version.

**User Experience**

```bash
# Create the resource type in the CableCo.App namespace
rad resource-type create CableCo.App/webservice -f webservice.yaml
```

The contents of `webservice.yaml` is similar to:

```yaml
--- 
namespace: CableCo.App 
resourceTypes: 
  - CableCo.App/webServices: 
    versions:
      - name: v1
        metadata:
          description: Standard CableCo web service 
          documentation: |
            # CableCo Web Service
            ## How and why to use a Web Service
            ...
            ## Example
            ...
            ## Input Parameters
            * Short name which will be used as the DNS name
            * Container image
            * Health check configuration
            * Minimum and maximum CPU and memory requirements
            * Minimum and maximum replicas 
            * Autoscaling metric
            ## Environment Variables
            ...
            ## Change Log
            ... |
        schema:  
          properties:
            - image
              type: string
              description: URI of the container image
              required: true
            - command
              type: string
              description: Command to execute within the container at launch
              required: false
            - args
              type: string
              description: Command arguements
              required: false
            - workingDir
              type: string
              description: Working directory when executing the command
              required: false
            - minCPU
              type: string
              description: Minimum number of vCPUs for the container; e.g., 0.5 or 500m for half a vCPU
              required: false
            - maxCPU
              ...
            - readinessProbe
              ...
            - livenessProbe
              ...
            - minReplicas
              ...
            - maxReplicas
              ...
            - autoscalerMetric
              ...
        # Embedded resources within this resource type
        resources:
          - Applications.Core/containers@v1
            name: $name-service
            properties:
            container:
              image: $image
              ports:
                http:
                  containerPort: 80
                  protocol: TCP
              command: $command
            }  
          - Application.Core/gateways@v1
            ...
          - Applications.Core/secretStores@v1
            ...
          - Applications.Core/autoscalers@v1
            ...
          - Applications.Datastores/redisCaches@v1
            name: redis
            properties:
              host: hostName
              port: port
              username: myusername
              secrets:
                password: redisPassword
          # Embedded connections
        connections: 
          - $name-service:
             source: redis
```

**Result**

1. The CableCo.App namespace (née resource provider) is created
2. The WebService resource type is created with the version and schema specified in the YAML file

**Exceptions**

The operation fails and informs the user interactively if:

* The user is not a member of the resource-type-admin Radius role 
* The resource type with the same version already exists

* The YAML file does not have a version specified

* The schema structure is not in valid

**Changes from current implementation**

This is very similar to the existing **UDT feature spec** with the exception of:

* Simplified YAML schema
* Embedded resources and passing parameters to the embedded resources
* Embedded connections
