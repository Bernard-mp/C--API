# Deploying the app
 
<!-- TOC -->
* [Deploying the app](#deploying-the-app)
  * [Overview](#overview)
    * [Steps to decide what to deploy](#steps-to-decide-what-to-deploy)
  * [Infrastructure](#infrastructure)
  * [Deploying infrastructure using CI/CD](#deploying-infrastructure-using-cicd)
    * [Deploying infra for the FIRST TIME](#deploying-infra-for-the-first-time)
    * [Deploying infra from CI/CD](#deploying-infra-from-cicd)
  * [Deploying app from CI/CD](#deploying-app-from-cicd)
  * [Deploying Okta configuration](#deploying-okta-configuration)
    * [Okta preview environments](#okta-preview-environments)
    * [Okta SASS environments](#okta-sass-environments)
  * [Executing database SQL scripts](#executing-database-sql-scripts)
    * [Steps](#steps)
  * [Granting access to Azure resources](#granting-access-to-azure-resources)
    * [Steps](#steps-1)
    * [Revoking access to Azure resources](#revoking-access-to-azure-resources)
    * [Azure environment Access levels](#azure-environment-access-levels)
  * [Deploying (infrastructure + app) locally from dev machine](#deploying-infrastructure--app-locally-from-dev-machine)
    * [Prerequisites](#prerequisites)
    * [Permissions to run infrastructure scripts](#permissions-to-run-infrastructure-scripts)
    * [Steps](#steps-2)
  * [Cleanup](#cleanup)
    * [From a powershell prompt](#from-a-powershell-prompt)
    * [From the github workflow](#from-the-github-workflow)
  * [Troubleshooting `provision-azure-resources.ps1`](#troubleshooting-provision-azure-resourcesps1)
<!-- TOC -->
 
> **Summary listing** of key information for each of the environments that have already been provisioned can be found [here](https://teams.microsoft.com/l/entity/0d820ecd-def2-4297-adad-78056cde7c78/_djb2_msteams_prefix_2617345832?context=%7B%22subEntityId%22%3Anull%2C%22channelId%22%3A%2219%3AWc0ST6g_UBkBMQWC6AHRNv4Tf_mbY7T-C63qyRuPbIc1%40thread.tacv2%22%7D&groupId=25822982-9537-4f49-b5ca-c7b16ca88249&tenantId=e04e9f50-006e-4eaa-ab0b-e804b0c7b7d1&allowXTenantAccess=false)
 
## Overview
 
At a high level deployment consists of:
 
1. Setting up the AKS cluster to support pod-identity (see section ["Deploying infra for the FIRST TIME"](#deploying-infra-for-the-first-time))
2. Deploying the infrastructure required for the app (see section ["Deploying infra from CI/CD"](#deploying-infra-from-cicd))
3. Deploying Okta configuration (eg authorization server) (see section ["Deploying Okta configuration"](#deploying-okta-configuration))
4. Deploying the app into the infrastructure (see section ["Deploying app from CI/CD"](#deploying-app-from-cicd))
5. Grant access to the teams members to the resources in Azure for the environment (see section ["Granting access to Azure resources"](#granting-access-to-azure-resources))
 
This repo contains various powershell scripts (see [tools directory](../tools)) that can be run from the command-line to automate the deployment tasks above 
and [github workflows](../.github/workflows) that automate CI/CD pipelines for the same deployments.
 
For more information on how these github workflows for the project were set up: [create-github-actions-infrastructure-pipeline](create-github-actions-infrastructure-pipeline.md)
 
### Steps to decide what to deploy
 
> **TIP**: there is a teams session recording [here](https://mrisoftware-my.sharepoint.com/:v:/p/christian_crowhurst/EQ4rPg89oxxBnlVSlNCw3hMBHlxW719aBkuQUR-46M_Crg?e=yPpdda&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZyIsInJlZmVycmFsQXBwUGxhdGZvcm0iOiJXZWIiLCJyZWZlcnJhbE1vZGUiOiJ2aWV3In19)
>          that deep-dives into deciding what to deploy, and then performing that deployment
Each sprint/month or whatever is the release cadence for AIG at the time, changes will be potentially made to infrastructure-as-code scripts, backend, frontend app and
okta desired state. The following section describe the steps to take that will identity what deployment needs to be run.
 
1. Identity the type of release being deployed:
   * hotfix
   * standard / regular cadence release
2. For standard / regular cadence release, create the release branches. EG:
   * backend repo release/2023.03 (see [guide](https://github.com/MRI-Software/data-services-gateway/blob/master/docs/branch-and-deployment-strategy.md))
   * frontend repo release/release-aig-2023.03
3. For standard feature release you might need to manually run the frontend app build pipeline
   * you will need to do this whenever changes have been committed to master and you cut a release branch, but do NOT make futher pushes to that release branch
   * see instructions: [CI/CD_Pipeline > Manual builds](https://dev.azure.com/mrisoftware/MRI_Platform/_git/mri-platform-ag-client-admin?path=/docs/CI-CD_Pipeline.md&version=GBmaster&_a=preview&anchor=manual-builds)
4. Determine which deployments are required:
   * Infrastructure
     - look for the latest github release that starts 'infra-release' eg `infra-release-2023.02-374`
     - use [this link](https://github.com/MRI-Software/data-services-gateway/releases?q=infra-release&expanded=true) to filter on such release
     - is this release been marked as published or is it still a pre-release? If it's still a pre-release you know that a deployment is required for the infrastructure
     - if this release is already published, then look for a the existing github actions workflow run in [Infrastructure Deploy Production Release](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-deploy-release.yml) corresponding to that release eg `infra-release-2023.02-374`
     - have all the environments have been deployed to? If no, then a deployment to those environments are still pending
   * AIG backend app
     - look for the latest github release that starts 'app-release' eg `app-release-2023.02-1331`
     - use [this link](https://github.com/MRI-Software/data-services-gateway/releases?q=app-release&expanded=true) to filter on such release
     - is this release been marked as published or is it still a pre-release? If it's still a pre-release you know that a deployment is required for the app
     - if this release is already published, then look for a the existing github actions workflow run in [Application Deploy Production Release](https://github.com/MRI-Software/data-services-gateway/actions/workflows/app-deploy-release.yml) corresponding to that release eg `app-release-2023.02-1331`
     - have all the environments have been deployed to? If no, then a deployment to those environments are still pending
   * AIG frontend app - browse to [the release board](https://dev.azure.com/mrisoftware/MRI_Platform/_release?_a=releases&view=mine&definitionId=35) and look for the most recent release for the current release branch
   * Okta desired state for backend api
     - changes to folder 'tools/okta' since the last release deployment
     - team will have raised a SNOW ticket for the desired state files to be run in mrisass.okta.com
5. Start the deployment(s) identified in step above. You will do that for **each environment in turn** from demo (maybe) then staging through to production. Typically you want to perform the deployments in the following order:
   1. Infrastructure (see [guide](#deploying-infra-from-cicd))
   2. AIG backend app (see [guide](#deploying-app-from-cicd))
   3. Okta desired state for backend api (see [guide](#deploying-okta-configuration))
   4. AIG frontend app (see [guide](https://dev.azure.com/mrisoftware/MRI_Platform/_git/mri-platform-ag-client-admin?path=/docs/deploy-aig-spa.md&_a=preview))
 
 
## Infrastructure
 
> Note: Image represents deployment to the dev environment. 
> Other environments will have the same resources but with different names, plus production and qa environments will also have failover instances for SQL and AKS pods
 
> [Online (editable) link](https://app.creately.com/d/mK0TmHIohMI/edit)
 
![provisioning-infrastructure](./provisioning-infrastructure.jpg)
 
> Also see output from [print-product-convention-table.ps1](../tools/infrastructure/print-product-convention-table.ps1)
 
## Deploying infrastructure using CI/CD
 
> **TIP**: to discover the configuration values used during deployment run: `./tools/infrastructure/get-product-conventions.ps1`
 
### Deploying infra for the FIRST TIME
 
1. Every environment: Enable Pod identity (aka managed identity for pods)
   1. Manually run the workflow [Infrastructure Enable AKS Pod-identity](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-enable-aks-pod-identity.yml), selecting the name of the environment to deploy to (for example dev)
      ![run workflow](./assets/infra-enable-aks-pod-id-run-workflow.png)
   2. For all environments except dev you will need to [approve deployment](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
2. Deploy to dev and qa environment:
   1. Touch any file in tools/infrastructure on the `master` (via a PR), *or* manually run [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) workflow
   2. Deploy to dev: [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) workflow will trigger *automatically* to deploy infrastructure to dev (ie you don't need to do anything)
   3. Deploy to qa: once deployed to the dev environment, the [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) workflow will queue up a deployment for the infrastructure to the qa environment.
      ![queued deployment](./assets/infra-ci-queued.png)
      This deployment will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
3. Deploy to demo, staging, prod-xxx environments:
   1. Go to the [Releases list](https://github.com/MRI-Software/data-services-gateway/releases) in the github repo
   2. Find the pre-release that you want to deploy, it will start with 'infra-master-' or 'infra-release-'
      ![infra release](./assets/infra-release.png)
   3. To deploy the release, select the 'Edit' option, **_uncheck_** 'Set as pre-release', and then select 'Update release'. This will start the execution of the deployment
 
      ![infra edit release](./assets/infra-edit-option.png)
 
      ![infra prerelease option](./assets/infra-prerelease-option.png)
 
   4. Approve the deployment to demo and/or staging, and then to production:
      1. Open the [Infrastructure Deploy Production Release](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-deploy-release.yml) workflow run just started that has the name of the release you're just published above
      2. [Approve](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments) the environment(s) listed in the UI to allow the deployment to continue for each of those respective environments
         ![queued deployment](./assets/infra-release-queued.png)
         **IMPORTNT**: the option to deploy to staging and prod environments will be enabled only when the branch that triggered the initial workflow is a release branch (eg release/2022.01)
4. Every environment: Add pod identity
   1. Manually run the workflow [Infrastructure Add AKS Pod-identity](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-add-aks-pod-identity.yml), selecting the name of the environment to deploy to (for example demo)
   2. For all environments except dev you will need to [approve deployment](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
 
### Deploying infra from CI/CD
 
1. Trigger build by _either_:
   * Touching any file in tools/infrastructure on the `master` branch (via a PR)
   * Manually running [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) workflow
   * Create a `release/*` branch (eg release/2022.01). Note: this is the only method to deploy to staging and prod environments as explained in [Branch and deployment strategy](branch-and-deployment-strategy.md)
2. Deploy to dev: [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) will trigger *automatically* to deploy infrastructure to dev environment (ie you don't need to do anything)
3. Deploy to qa: once deployed to the dev environment, the [Infrastructure CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-ci-cd.yml) workflow will queue up a deployment for the infrastructure to the qa environment.
   ![queued deployment](./assets/infra-ci-queued.png)
   This deployment will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
4. Deploy to demo, staging, prod-xxx environments:
   1. Go to the [Releases list](https://github.com/MRI-Software/data-services-gateway/releases) in the github repo
   2. Find the pre-release that you want to deploy, it will start with 'infra-master-' or 'infra-release-'
      ![infra release](./assets/infra-release.png)
   3. To deploy the release, select the 'Edit' option, _**uncheck**_ 'Set as pre-release', and then select 'Update release'. This will start the execution of the deployment
 
      ![edit release](./assets/infra-edit-option.png)
 
      ![prerelease option](./assets/infra-prerelease-option.png)
 
   4. Approve the deployment to demo and/or staging, and then to production:
      1. Open the [Infrastructure Deploy Production Release](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-deploy-release.yml) workflow run just started that has the name of the release you're just published above
      2. [Approve](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments) the environment(s) listed in the UI to allow the deployment to continue for each of those respective environments
 
         **IMPORTNT**: the option to deploy to staging and prod environments will be enabled only when the branch that triggered the initial workflow is a release branch (eg release/2022.01)
 
         ![queued deployment](./assets/infra-release-queued.png).
 
         Example of the approval dialog:
 
         ![approve deployment](./assets/infra-release-approve.png)
 
## Deploying app from CI/CD
 
> **CRITICAL**: ensure that any changes required to front-end SPA are also deployed as described in [agora-insights-gateway-frontend docs](https://dev.azure.com/mrisoftware/MRI_Platform/_git/mri-platform-ag-client-admin?path=/docs/deploy-aig-spa.md&_a=preview)
 
> **TIP**: to discover the configuration values used during deployment run: `./tools/infrastructure/get-product-conventions.ps1`
 
1. Trigger build by _either_:
   * Touching any file in tools/infrastructure on the `master` branch (via a PR)
   * Manually running [Application CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/app-ci-cd.yml) workflow
   * Create a `release/*` branch (eg release/2022.01). Note: this is the only method to deploy to staging and prod environments as explained in [Branch and deployment strategy](branch-and-deployment-strategy.md)
2. Deploy to dev: [Application CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/app-ci-cd.yml) will trigger *automatically* to deploy app to dev environment (ie you don't need to do anything)
3. Deploy to qa: once deployed to the dev environment, the [Application CI/CD](https://github.com/MRI-Software/data-services-gateway/actions/workflows/app-ci-cd.yml) workflow will queue up a deployment for the app to the qa environment.
   ![queued deployment](./assets/app-ci-queued.png)
   This deployment will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
4. Deploy application to demo, staging, prod-xxx environments:
   1. Go to the [Releases list](https://github.com/MRI-Software/data-services-gateway/releases) in the github repo
   2. Find the pre-release that you want to deploy, it will start with 'app-master-' or 'app-release-'
      ![app release](./assets/app-release.png)
   3. To deploy the release, select the 'Edit' option, **_uncheck_** 'Set as pre-release', and then select 'Update release'. This will start the execution of the deployment
 
      ![edit release](./assets/infra-edit-option.png)
 
      ![prerelease option](./assets/infra-prerelease-option.png)
 
   4. Approve the deployment to demo and/or staging, and then to production:
      1. Open the [Application Deploy Production Release](https://github.com/MRI-Software/data-services-gateway/actions/workflows/app-deploy-release.yml) workflow execution just started that has the name of the release you're just published above
      2. [Approve]((https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)) the environment(s) listed in the UI to allow the deployment to continue for each of those respective environments
         **IMPORTNT**: the option to deploy to staging and prod environments will be enabled only when the branch that triggered the initial workflow is a release branch (eg release/2022.01)
 
         ![queued deployment](./assets/app-release-queued.png).
         Example of the approval dialog:
 
         ![approve deployment](./assets/app-release-approve.png)
 
## Deploying Okta configuration
 
There is a github workflow ([Okta Set Desired State](https://github.com/MRI-Software/data-services-gateway/actions/workflows/okta-set-desired-state.yml)) that will deploy the okta configuration required for API authorization.
Frustratingly, it will fail when run for any environment. This is because the okta api-key that is used to authenticate the workflow against okta REST API does not
have the permissions it requires. Specifically, it does not have the permission to create/modify an authorization server that has inline-hooks configured.
 
Therefore, as an alternative to using our github workflow, there is a powershell script that can be run manually as explained in the next sections. This script performs the 
same task of our github workflow - that of creating/updating the Okta configuration required by our API.
 
### Okta preview environments
 
For dev, qa and release environments the powershell script can be run by developers who are admins in the oktapreview account.
 
1. Open a pwsh core command prompt (it's important that this is powershell core 6.2 or above)
2. In the pwsh prompt: `./tools/okta/run/run-non-prod.ps1`
   * You may be prompted with Azure Artifacts PAT token. If you don't have one, create one with at least Packaging:Read scope. See guide: https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?tabs=preview-page#create-a-pat
   * You will be prompted for your okta api token for the Okta Preview account. If you don't already have an api token, create one using the guide: https://developer.okta.com/docs/guides/create-an-api-token/create-the-token/
 
### Okta SASS environments
 
For demo, staging and production environment we need to send a pwsh script as a SNOW ticket.
The process of raising the SNOW ticket is explained in [Okta README](../tools/okta/run/README.md)
 
 
## Executing database SQL scripts
 
On the occasion that you need to execute a SQL script against the AIG database, manually run the
[Database Execute SQL Script](../.github/workflows/db-deploy-sql-script.yml) github workflow as explained below.
 
### Steps
 
1. If not already done so, commit the sql script file to the repo:
   * Typically add these sql script file to the [`tools/db-scripts`](../tools/db-scripts) folder
   * For specific fixes that need to be applied to the database, add these files to the [`tools/db-scripts/fixes`](../tools/db-scripts/fixes) folder
   * Make sure to add a comment section at the top of the script providing an overview and a business justification
     (this will be important for compliance reasons and to help the approver of the workflow decide whether to approve the execution of the script
2. Open the github workflow [Database Execute SQL Script](../.github/workflows/db-deploy-sql-script.yml)
3. Select the "Run workflow" button
4. In the dialog:
   1. Select the Environment containing the AIG db to target
   2. Enter the relative path in this repo to the sql script file to execute
5. For all environments except dev, the workflow run will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments). Approvers:
   * QA environment: members of the [Enterprise Data Services](https://github.com/orgs/MRI-Software/teams/enterprise-data-services)
   * Other environment: members of the [Enterprise Data Services - Production approver](https://github.com/orgs/MRI-Software/teams/enterprise-data-services-production-approver/members)
 
## Granting access to Azure resources
 
### Steps
 
1. Decide on the access level for each person (see 'Access levels' section below)
2. Open the github workflow [Infrastructure Grant Azure Environment Access](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-grant-azure-env-access.yml)
3. Select the "Run workflow" button
   ![run workflow](./assets/infra-grant-access-run-workflow.png)
4. In the dialog:
   1. Select the Environment to which to grant access (select 'all' to expedite the process considerably)
   2. Select the Access level that is appropriate for the person (see below for description of each access level)
   3. In 'A comma delimited list of User principal names to grant' add the email address of the person(s) to grant access
   4. Select 'Run workflow' button
5. For all environments except dev, the workflow run will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
   * See example workflow run screenshots below
   * Approvers:
     * QA environment: members of the [Enterprise Data Services](https://github.com/orgs/MRI-Software/teams/enterprise-data-services)
     * Other environment: members of the [Enterprise Data Services - Production approver](https://github.com/orgs/MRI-Software/teams/enterprise-data-services-production-approver/members)
 
Once approved, the Azure RBAC permissions applicable to the selections above will be provisioned to the resources in Azure.
 
To check existing access, go to the Azure AD group in the portal. The naming convention of these groups is as follows:
 
* development team: sg.role.development.aig._\[env]_. EG:
  * dev environment: [sg.role.development.aig.dev](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/27054828-d480-4702-aaa3-fe906caed400)
* GPS: sg.role.supporttier1.aig._\[env]_. EG:
  * staging environment: [sg.role.supporttier1.aig.staging](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/23718c10-635b-478d-8ad8-378c2b29dc94)
  * prod-emea environment: [sg.role.supporttier1.aig.prodemea](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/1ab38486-1e2d-4064-a84a-02ea403246f7)
  * prod-apac environment: [sg.role.supporttier1.aig.prodapac](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/2720c540-798e-4896-9927-ce65339fec62)
  * prod-na environment: [sg.role.supporttier1.aig.prodna](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/b2760156-02e7-402d-a15c-5f2b47e7b1b5)
* App Admin: sg.role.supporttier2.aig._\[env]_. EG:
  * staging environment: [sg.role.supporttier2.aig.staging](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/1f0eb829-a70a-4df2-8d8e-38d9c673568d)
  * prod-emea environment: [sg.role.supporttier2.aig.prodemea](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/1ab38486-1e2d-4064-a84a-02ea403246f7)
  * prod-apac environment: [sg.role.supporttier2.aig.prodapac](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/c4c90961-0d83-4c04-8edb-33026153a210)
  * prod-na environment: [sg.role.supporttier2.aig.prodna](https://portal.azure.com/#view/Microsoft_AAD_IAM/GroupDetailsMenuBlade/~/Overview/groupId/e24440af-3b26-4e84-9d3b-405aa545013c)
 
_Example workflow run:_
![run workflow](./assets/infra-grant-access-run-workflow-run-1.png)
 
![run workflow](./assets/infra-grant-access-run-workflow-run-2.png)
 
### Revoking access to Azure resources
 
1. Open the github workflow [Infrastructure Revert Azure Environment Access](https://github.com/MRI-Software/data-services-gateway/actions/workflows/infra-revert-azure-env-access.yml)
2. Select the "Run workflow" button\
   ![run workflow](./assets/infra-revoke-access-run-workflow.png)
3. In the dialog:
   1. Select the Environment to which to revoke access
   2. Select the Access level that's to be revoked for the person
   3. In 'User to revoke' add the email address of the person to revoke access
   4. Select 'Run workflow' button
4. For all environments except dev, the workflow run will need to be reviewed then [approved in github](https://docs.github.com/en/actions/managing-workflow-runs/reviewing-deployments)
 
### Azure environment Access levels
 
1. development
   * dev, qa: 
     * admin access to AIG Azure SQL db
     * contributor access to Azure resources (_including_ access to keyvault)
     * contributor access to power-bi client workspaces
   * demo:
      * data read/write access to AIG Azure SQL db
      * contributor access to Azure resources (no access to keyvault)
      * contributor access to power-bi client workspaces
   * staging and prod: 
     * data read access to AIG Azure SQL db
     * contributor access to Azure monitor, read access to all other Azure resources (no access to keyvault)
     * no access to power-bi client workspace
2. GPS / support-tier-1
   * demo, staging and prod environments: 
     * data read access to AIG Azure SQL db
     * read access to Azure (no access to keyvault)
     * contributor rights to power-bi client report workspaces
     * viewer rights to power-bi client dataset workspaces
3. App Admin / support-tier-2
   * demo, staging and prod environments:
      * contributor access to AIG Azure SQL db
      * contributor access to Azure (_including_ access to keyvault)
      * admin rights to power-bi client report workspaces
      * admin rights to power-bi client dataset workspaces
 
To provide a comprehensive list of permissions per environment execute [print-product-convention-table.ps1](../tools/infrastructure/print-product-convention-table.ps1),
specifically, the example with the description "Returns tables describing all Azure RBAC and Azure ADD security group membership"
 
## Deploying (infrastructure + app) locally from dev machine
 
> **CRITICAL**: creating the infrastructure from your local dev machine using the provision script below (./tools/infrastructure/provision-azure-resources.ps1)
> will likely cause the Infrastructure CI/CD pipeline to fail for the environment that you deployed to locally.
> This is because the AAD groups will be created with your identity as the owner, and NOT the service principal that the
> pipeline uses. This will cause the pipeline to fail when it tries to add a group member
 
### Prerequisites
 
* [az-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) (**minimum vs 2.39.0**), required to:
    * enable/add pod identity
    * run dev scripts
* [Azure bicep cli](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/install#install-manually) (**minimum vs 0.24.24**)
* powershell core (tested on v7.2)
* docker engine to run the dev script with the flag `-DockerPush`
 
### Permissions to run infrastructure scripts
 
You are unlikely to have permissions to run the infrastructure provisioning scrips (steps 1-3 below) from your dev machine :-(
In practice the only way to run these scripts from a dev machine is:
 
1. To have your own Azure subscription where you are the owner, AND
2. The Azure subscription is linked to a developer Azure AD tenant created using the Microsoft 365 developer program. See the following on how to get this setup:
    1. sign-up for the MS 365 developer program: <https://developer.microsoft.com/en-us/microsoft-365/dev-program>
    2. linking your VS subscription to your office365 dev tenant: <https://laurakokkarinen.com/how-to-use-the-complimentary-azure-credits-in-a-microsoft-365-developer-tenant-step-by-step/>
    3. things to be aware of when moving your VS subscription to another AD tenant: <https://docs.microsoft.com/en-us/azure/role-based-access-control/transfer-subscription>
 
 
### Steps
 
1. (Once-only) Modify product conventions to avoid conflicts for those azure resource whose names are globally unique:
   1. open [get-product-conventions.ps1](../tools/infrastructure/get-product-conventions.ps1)
   2. set `ProductName` (line 20) to make it globally unique (adding your initials eg `-cc` as a prefix should be sufficient)
   3. comment out the line `Get-ResourceConvention @conventionsParams -AsHashtable:$AsHashtable`
   4. comment-in the block of code that starts `# If you need to override conventions, ...`
   5. set the `RegistryName` to make it globally unique (adding your initials eg `cc` as a prefix should be sufficient)
2. (Once-only) Setup shared infrastructure:
   1. Provision AKS. See "Permissions to run infrastructure scripts" above for reason this is necessary
      ```pwsh
      # 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
      ./tools/infrastructure/add-aks-cluster.ps1 -InfA Continue -EnvironmentName dev -CreateAzureContainerRegistry -Login -SubscriptionId xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
      ````
   2. Enable Pod identity (aka managed identity for pods):
      ```pwsh
      # 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
      ./tools/infrastructure/enable-aks-pod-identity.ps1 -InfA Continue -EnvironmentName dev -Login -Subscription xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
      ````
3. (When changed) Provision Azure resources:
   ```pwsh
      # 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
      ./tools/infrastructure/provision-azure-resources.ps1 -InfA Continue -EnvironmentName dev -Login -Subscription xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
      ````
    * NOTE: if this script fails try running it again (script is idempotent)
    * **IMPORTANT**: If a secondary (failover) Azure SQL server is provisioned - see troubleshooting section below
4. (Once-only) Add Pod identity for API app:
   ```pwsh
      # 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
      ./tools/infrastructure/add-aks-pod-identity.ps1 -InfA Continue -EnvironmentName dev -Login -Subscription xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
      ````
5. Build App: `./tools/dev-scripts/build.ps1 -DockerPush -InfA Continue`
    * **IMPORTANT**: You will need to have docker engine installed and running on your machine in order to build and push the images
6. Deploy App: 
   ```pwsh
      # IMPORTANT: You will likely need to connected to the office VPN in order to satisfy the firewall rules configured in the Azure SQL db
      # 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
      ./tools/dev-scripts/deploy.ps1 -InfA Continue -Login -Subscription xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
      ````
7. Test that it worked:
    * browse to the "Api health Url" printed to the console
    * Import the postman [collection](../tests/postman/api.postman_collection.json) and [environment](../tests/postman/api-local.postman_environment.json),
      change the baseUrl postman variable to the "Api Url" printed to the console. Run the requests in the collection
 
 
## Cleanup
 
To remove all Azure resources and AKS pod identity run the deprovision-azure-resources.ps1 script from a powershell prompt (assuming you have permissions),
or running the github workflow [Infrastructure Uninstall](../.github/workflows/infra-uninstall.yml)
 
### From a powershell prompt
 
```pwsh
# 'CC - Visual Studio Enterprise' subscription id: 402f88b4-9dd2-49e3-9989-96c788e93372
./tools/infrastructure/deprovision-azure-resources.ps1 -InfA Continue -Environment xxx -UninstallAksApp -DeleteAADGroups -Login -Subscription xxxxxxxx-xxxx-xxxxxxxxx-xxxxxxxxxxxx
```
 
### From the github workflow
 
1. Go to the Actions tab in the github repo
2. Manually run the workflow 'Infrastructure Uninstall', selecting the name of the environment to uninstall from
 
 
## Troubleshooting `provision-azure-resources.ps1`
 
When running `provision-azure-resources.ps1`, you might receive an error with the following message:
 
```cmd
Login failed for user '<token-identified principal>'
```
 
To resolve the problem try re-running the provisioning script again (it's safe to do so). This still may not work if the script
is provisioning a secondary Azure SQL server as part of a failover group. In this case, try waiting for somewhere between 15-60 minutes and re-run the script.
It appears that the creation of the replicated database takes sometime and is cause of the problem.
