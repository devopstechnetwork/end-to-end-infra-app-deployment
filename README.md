# Overview

Here we trace the workflow of a developer deploying infrastructure and applications to Azure using Packer, GitHub, Jenkins, Terraform, Vault, Ansible, and Consul. We use our webblog application to demo this.

## Topics to Learn
1. Vault Azure Secrets Engine
2. Packer Images in Azure
3. Terraform Building VMs in Azure based on Packer Images
4. Ansible to Configure an Azure VM
5. Vault Secure Introduction
6. Vault App Role
7. Vault Dynamic Database Secrets for MongoDB
8. Vault Transit Secrets Engine
9.  Advanced CI/CD Pipeline Workflow using GitHub(VCS), Jenkins(CI/CD), Terraform(IaC), Ansible(Config Mgmt), Vault(Secrets Mgmt)
10. Consul Service Mesh

## Vault Azure Secrets Engine
Let's take a look at how we can build this. You can find details in the [docs](https://www.vaultproject.io/docs/secrets/azure). You can also follow the [step-by-step guide](https://learn.hashicorp.com/tutorials/vault/azure-secrets).

Below is a diagram of the Vault Azure Secrets Engine Workflow

![Vault Azure Secrets Engine Workflow Diagram](https://learn.hashicorp.com/img/vault-azure-secrets-0.png)

### Vault Configuration

The configuration setup below needs to be done by a Vault Admin. This [Vault policy](https://learn.hashicorp.com/tutorials/vault/azure-secrets#policy-requirements) is used with a token to run the configuration commands. We use the root token in this demo for simplicity, however, in a production setting it's not recommended to use the root token.

We are re-using our existing Vault cluster. The Vault admin configuration is located in the [infrastructure-gcp GitLab repo](https://gitlab.com/public-projects3/infrastructure-gcp/-/tree/master/terraform-vault-configuration)

#### Setup

Below an admin uses the Vault Terraform Provider. This is found in the [main.tf file](https://gitlab.com/public-projects3/infrastructure-gcp/-/blob/master/terraform-vault-configuration/main.tf)

```
resource "azurerm_resource_group" "myresourcegroup" {
  name     = "${var.prefix}-jenkins"
  location = var.location
}

resource "vault_azure_secret_backend" "azure" {
  subscription_id = var.subscription_id
  tenant_id = var.tenant_id
  client_secret = var.client_secret
  client_id = var.client_id
}

resource "vault_azure_secret_backend_role" "jenkins" {
  backend                     = vault_azure_secret_backend.azure.path
  role                        = "jenkins"
  ttl                         = 300
  max_ttl                     = 600

  azure_roles {
    role_name = "Contributor"
    scope =  "/subscriptions/${var.subscription_id}/resourceGroups/${azurerm_resource_group.myresourcegroup.name}"
  }
}
```

### Request Azure Creds Manually

```shell
vault policy write jenkins Vault/policies/jenkins_azure_policy.hcl
vault token create -policy=jenkins
VAULT_TOKEN=xxxxxx vault read azure/creds/jenkins
```

### Packer to Build a Jenkins Image in Azure

#### Steps
1. Create a Packer image in Azure with Docker installed
2. Build a Docker image that has Jenkins, Terraform, and Ansible installed

**Note**
When using the Azure creds, I couldn't use the ones generated by Vault because they are specific to the `samg-jenkins` resource group. Packer for some reason uses a random Azure resource group when building therefore it needs creds that have a scope for any resource group. I used the regular service principal creds.

### Terraform to Build a Jenkins VM in Azure and Ansible to Configure it

We use Terraform to build an Azure VM based on the Packer image we previously created. 

**Note**
Here we can use the Vault generated creds to build a VM in Azure since the creds are tied to the `samg-jenkins` resource group.

### Secure Introduction

Below are some resources that talk about Secure Introduction and Secret Zero
[HashiTalk on Vault Response Wrapping and Secret Zero](https://www.hashicorp.com/resources/vault-response-wrapping-makes-the-secret-zero-challenge-a-piece-of-cake)
[GitHub Repo for above HashiTalk](https://github.com/misurellig/hashitalks-demo)

#### Workflow

1. A Vault Admin creates a webblog app role-id
2. An admin creates a Packer VM image in Azure with the webblog app role-id
3. A Vault Admin provides Jenkins with a token with permissions to only write a wrapped Secret ID (Jenkins is a Vault trusted entity)
4. Jenkins creates a wrapped Secret ID and delivers it to the pipeline
5. The Jenkins pipeline unwraps the Secret ID and uses it with the Role ID to login into Vault
6. The pipeline then retrieves the Azure creds and passes them to Terraform
7. Terraform 

#### Create an Approle for the Jenkins Node

This is done via Terraform using the following configuration:

```shell
resource "vault_policy" "jenkins_policy" {
  name = "jenkins-policy"
  policy = file("policies/jenkins_policy.hcl")
}

resource "vault_auth_backend" "jenkins_access" {
  type = "approle"
  path = "jenkins"
}

resource "vault_approle_auth_backend_role" "jenkins_approle" {
  backend            = vault_auth_backend.jenkins_access.path
  role_name          = "jenkins-approle"
  secret_id_num_uses = "5"
  secret_id_ttl      = "300"
  token_ttl          = "1800"
  token_policies     = ["default", "jenkins-policy"]
}
```

The `jenkins_policy.hcl` file mentioned here contains the following policy:

```shell
path "auth/pipeline/role/pipeline-approle/secret-id" {
  policy = "write"
  min_wrapping_ttl   = "100s"
  max_wrapping_ttl   = "300s"
}
```

Once you configure Vault via Terraform, you can then run the two commands below to get the `role-id` and the `secret-id`. You can see more [instructions in the documentation.](https://www.vaultproject.io/docs/auth/approle)

```shell
vault read auth/jenkins/role/jenkins-approle/role-id
vault read auth/jenkins/role/jenkins-approle/secret-id
```

You can now take the `role-id` and the `secret-id` and insert them into the Jenkins Vault plugin for authentication.

#### Create an Approle for the Jenkins Pipeline

Once again we use Terraform for configuration as shown below:

```shell
resource "vault_policy" "pipeline_policy" {
  name = "pipeline-policy"
  policy = file("policies/jenkins_pipeline_policy.hcl")
}

resource "vault_auth_backend" "pipeline_access" {
  type = "approle"
  path = "pipeline"
}

resource "vault_approle_auth_backend_role" "pipeline_approle" {
  backend            = vault_auth_backend.pipeline_access.path
  role_name          = "pipeline-approle"
  secret_id_num_uses = "1"
  secret_id_ttl      = "300"
  token_ttl          = "1800"
  token_policies     = ["default", "pipeline-policy"]
}
```

You then need to read the role-id for the Jenkins policy and insert that into the jenkinsfile for the pipeline. The Jenkins node will create a wrapped secret ID for the pipeline and in fact, that's the only capability it has as defined in the Jenkins policy mentioned above. The pipeline then unwraps the secret-id and retrieves a VAULT_TOKEN that will get used for the remainder of the pipeline. Below is the command used to generate the role-id for the pipeline.

```shell
vault read auth/pipeline/role/pipeline-approle/role-id
```