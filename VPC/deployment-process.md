# Deployment Guide

A comprehensive guide on how the XVP Infra team performs deployments for regular and blue/green workflows.

As we continue to grow and learn this guide will grow to ensure we are up to date with agreed upon approaches to all of XVP.

---

## Table of Contents

- [Deployment Guide](#deployment-guide)
  - [Table of Contents](#table-of-contents)
  - [Bluegreen deployment](#bluegreen-deployment)
  - [**Build Clusters**](#build-clusters)
  - [Connect Clusters to Mesh](#connect-clusters-to-mesh)
  - [**Inactive Version Update**](#inactive-version-update)
  - [External Dependencies](#external-dependencies)
    - [Work with a member of the XCI Infra team to address these dependencies](#work-with-a-member-of-the-xci-infra-team-to-address-these-dependencies)
    - [**Deploy XVP services to clusters**](#deploy-xvp-services-to-clusters)
    - [**DNS Promotion**](#dns-promotion)
  - [Release Process](#release-process)
    - [Overview](#overview)
    - [Step 0: Planning a release](#step-0-planning-a-release)
    - [Step 1: Deploy to all lower environments](#step-1-deploy-to-all-lower-environments)
    - [**Staging Deployment**](#staging-deployment)
      - [**Cutting a release blue green**](#cutting-a-release-blue-green)
    - [If the version resource has other version(s) ahead of the one that is required to be deployed, pin the desired version in Concourse prior to creating a blue/green draft release](#if-the-version-resource-has-other-versions-ahead-of-the-one-that-is-required-to-be-deployed-pin-the-desired-version-in-concourse-prior-to-creating-a-bluegreen-draft-release)
      - [**Cutting a release refresh**](#cutting-a-release-refresh)
      - [QA Testing](#qa-testing)
    - [Step 2: Finalize Release](#step-2-finalize-release)
    - [Step 3: Create IOP Ticket](#step-3-create-iop-ticket)
      - [IOPs during an active moratorium](#iops-during-an-active-moratorium)
      - [IOP in Slack](#iop-in-slack)
    - [Step 4: Deployment Calendar \& Calendar Invites](#step-4-deployment-calendar--calendar-invites)
    - [Step 5: Deploy to Production](#step-5-deploy-to-production)
  - [Production Deployment In-Place/Refresh](#production-deployment-in-placerefresh)
  - [Production Deployment Blue/Green](#production-deployment-bluegreen)
    - [Rollback Production Deployment Refresh](#rollback-production-deployment-refresh)
    - [Helpful Dashboards \& Links for monitoring](#helpful-dashboards--links-for-monitoring)

---

## Bluegreen deployment

A Blue/Green deployment is used whenever there are new changes introduced to the infrastructure that is deemed `destructive` or `highly disruptive` that will require the XCI Infra team to stand up new infrastructure from scratch. This is to prevent in-place deployments that could cause issues and side effects such as bringing down the cluster or putting it into an unhealthy state. As such, this document aims to provide information and steps as to how to perform a blue/green deployment. This document also provides information on the #release-process and in-place/refresh deployments under #staging-deployment, note that a few things will be different depending on if the deployment type is Blue/Green or Refresh.

## **Build Clusters**

The first step is to build new EKS clusters from scratch. All Blue/Green jobs can be found in the `xvp-infra-core-eks` [pipeline](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=pipeline) in a group of jobs that follow this format: `${env}-bg` i.e. `dev-bg`

![Blue-Green-Jobs](./img/env-bg-jobs.png)

Look for the job called `build-cluster-from-release-${env}-${aws_region}`

Each build cluster job contrains `3` operations:
![Cluster-job](./img/build-cluster-job.png)

1. `eks-terraform-bluegreen-release-${aws_region}` runs the Terraform `apply` to create all of the required components to build a basic EKS cluster with all of the bells and whistles
   1. It appends the `release_version` to the cluster name that is derived from generating a `xvp-infra-core` git release in staging
       - **Note: This does not work in dev, plat-dev, and perf as we generate the release notes from the `staging` job**
       - To bump the release version, *manually* update it in Vault for these envs
   2. A version would be called `v77` and is looked up prior to building the cluster
      - If a release version does not exist, the job will fail as it will attempt to look up the value from Vault first
2. The `create_oidc_provider` task runs a script that creates an OIDC provider endpoint for the EKS cluster that is required for use by certain EKS and K8s components
3. The `k8s-terraform-bluegreen-release-${aws_region}` runs the Terraform `apply` to create all of the required K8s components such as Kiali, Vector, etc

Once the job completes, the cluster is deployed and the infrastructure fully stood up for that environment in a given region. Before deploying services to the cluster, there are some external dependencies to take care of first.

## Connect Clusters to Mesh

After the clusters in a continent are deployed, they will need to connect to each other in order to form a Istio Service Mesh.

Run the `create-apply-remote-secrets-${env}-${continent}` jobs

More info on it can be found [here](./istio-multicluster.md)

## **Inactive Version Update**

**IMPORTANT**
As part of the [Connect Clusters to Mesh](#connect-clusters-to-mesh) step, within those jobs are a Vault task to update the `inactive_version` secret.

For example, if the new clusters are running on `release_version` that is `v77`, then once the remote-secrets job runs, it will take that release version value and update the `inactive_version` to it.

- Release Version is v77
- Cluster is built and connected to mesh
- Inactive version updated to v77

## External Dependencies

Once clusters are built, there are a few changes that need to be made outside of XVP and XCI in order to call the deployment a success:

- Backwaters allowlist EKS IAM roles
  - Required for security logging
  - Reach out to #backwaters team on Slack
- Lynceus updates Prometheus scraping configs on new cluster(s)
  - Required for metrics that is essential to any deployment
  - Open PR in the [observability repo](https://github.com/comcast-observability-tenants/xvp)

### Work with a member of the XCI Infra team to address these dependencies

The team will then perform manual validations to spot check and ensure the pods, cluster, and metrics are healthy before proceeding.

### **Deploy XVP services to clusters**

Once the cluster has been validated, deploy the XVP services into the new blue/green cluster by updating their Terraform remote state versioned files to point at the new cluster.

`
Note: In order for service pods to come up healthy, config-api must be deployed first as it contains secrets and other important Java Spring information that is required for all services.
`

Once services are deployed, run their respective QA smoke tests for QA teams to validate their services.

After service validations are complete, the new blue/green cluster is now deemed ready to serve traffic to clients.

### **DNS Promotion**

There are two sets of  `Shared DNS` records for the regional records in dev and prod AWS accounts.

In `prod` AWS account where the public records live:
`xvp-eks-shared-active-${env}-${aws_region}.exp.xvp.na-1.xcal.tv`
that will resolve to the active NLB.

XVP services Route53 `public` records will hit this endpoint

```bash
Note: The reason why it lives in the prod account is due to how AWS handles Route53 A records living in the same hosted zone.
During development, there were issues found when attempting to have a service public route53 record attempting to point at an A record in a different zone. This is not allowed, hence having two sets of shared dns records to get around this issue
```

In `dev` AWS account where the protected and canary records live:
`xvp-eks-shared-active.${env}.exp.${aws_region}.aws.xvp.xcal.tv`

Each record will point at the `active` versioned cluster based on the `active_version` secret in Vault.

During a blue/green deployment, there will be an inactive and active clusters. The `inactive` cluster is the newly built cluster that is ready to serve traffic and will become the active cluster once DNS promotion completes.

More information on the general process can be found in [DNS Promotion](.//blue-green-deployment.md#dns-promotion)

![dns-promotion](./img/dns-promotion.png)

**To promote/toggle between the active and inactive clusters do the following:**

1. Run the `toggle-active-version-${env}` job for the given environment
2. It will look up the current secrets in Vault and swap the `inactive_version` and `active_version`
   - Read more about the [Vault secrets](./blue-green-eks-clusters.md#xvp-infra-core-eks-version-secret)
3. A successful run will auto-trigger the `promote-active-plan-${env}-${aws_region}` set of jobs
4. Inspect the terraform plan, it should show changes like the following:

    ```hcl
    resource "aws_route53_record" "public_active_cluster_name" {
            id             = "Z06363662MG2UIQMAQ9BL_xvp-eks-shared-active-plat-dev.exp.xvp.na-1.xcal.tv_A_active"
            name           = "xvp-eks-shared-active-plat-dev.exp.xvp.na-1.xcal.tv"
            # (6 unchanged attributes hidden)
          + alias {
              + evaluate_target_health = true
              + name                   = "xvp-eks-plat-dev-v77-ingress-64b9ae7631fea29.elb.
              us-east-2.amazonaws.com"
              + zone_id                = "ZLMOA37VPKANP"
            }
          - alias {
              - evaluate_target_health = true -> null
              - name                   = "xvp-eks-plat-dev-v76-ingress-b6e7559ca946fe48.elb.us-east-2.amazonaws.com" -> null
              - zone_id                = "ZLMOA37VPKANP" -> null
            }
            # (1 unchanged block hidden)
        }
    ```

    `Note: Terraform wants to update the shared active dns record to point from the old NLB to the new NLB that was created with the new blue/green cluster (i.e. v77 is the toggled version while v76 becomes the inactive version)`

5. Run the `promote-active-apply-${env}-${aws_region}` job once the plan looks good and Terrform will update the route53 records
6. Verify XVP services are serving traffic from the toggled active NLB.
   - i.e. If v77 was the new NLB, metrics and logs will show services resolving from the v77 cluster

---

## Release Process

### Overview

The release process for core infrastructure changes (eks, k8s, istio) has been implemented with heavy influence from the [XVP Service release process](https://internal-xvp-docs-staging.r53.aae.comcast.net/Team/xvp-release-management). The improved process not only generates automated changelog/release notes but it provides a gate that will prevent code from being deployable to Production until it has been successfully deployed to all staging regions. This gate will allow for more flexibility in the platform team's workflows eliminating the scenario where a change scheduled to be deployed to Production would prevent all merges to `main` until the deployment completes. As long as the draft release is finalized in a timely manner, additional merges will not be included in the subsequent production deployment.

### Step 0: Planning a release

First things first, you need to plan a release with the XVP Infra team and get agreement of the changeset to be deployed.

If a breaking or major change is introduced to XVP, it needs to be communicated out to all of XVP as it impacts us all. This should be considered when planning to cut a release for production deployments.

Reach out to XVP QA and bring awareness of the fact that changes are being deployed to ALL lower environments and testing is required in staging before cutting a release. If possible, plan ahead where XVP QA and services can have at least up to 48 hours notice for ample amounts of testing.

**Recommendations:**

- Use feature flags if need to roll out changes gradually
- Gather feedback from XVP service smoke/regression tests
- Let XVP tech leads know of the changes to come

### Step 1: Deploy to all lower environments

Once the PR has been closed and the feature branch merged into main, proceed to run the pipeline jobs to apply in all regions across [`dev`](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=dev), [`perf`](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=perf) and [`stg`](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=stg) environments. Once the code has successfully applied to all `stg` regions, a new [Draft Release](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks/jobs/create-git-release-draft) will be automatically created in [Github](https://github.com/comcast-xvp/xvp-infra-core/releases).

### **Staging Deployment**

As stated before, the `release_version` secret is stored in `Vault` and is updated manually in dev, plat-dev, and perf. However, for `staging`, there is an automated process to generate a Git release from which the cluster will derive its version.

More information on blue/green versioning can be found [here](./blue-green-eks-clusters.md#versioning-in-staging)

![Blue-Green-Release](./img/blue-green-release.png)

```text
Note: In a non-blue/green deployment process, the EKS and K8s jobs have to finish running first before running the create-git-release-draft job (which is kicked off automatically as described in the previous section).
In a Blue/Green deployment model, wherein the release version is needed before creating the cluster, the job is kicked off in the beginning in order to pass the release version to the build cluster job.
```

#### **Cutting a release blue green**

Prior to cutting the release it is best practice to bring the release up as an S2 for XCI-Infra Team awareness. This will ensure the team is aligned and able to coordinate multiple changes from different members.

It can also be good practice to notify XVP services and other stakeholders in #xvp-infra that a release is in the process of being created.

### If the version resource has other version(s) ahead of the one that is required to be deployed, pin the desired version in Concourse prior to creating a blue/green draft release

The `create-bluegreen-git-release-draft` kicks off the staging deployment and will generate a release. It will bump the version resource and use it for the release notes, which is the same process in the non-blue/green staging deployment workflow.

Navigate to the `xvp-infra-core` [Git releases page](https://github.com/comcast-xvp/xvp-infra-core/releases) after the job runs and you will see a `Draft` of the release that the cluster will be built from.

![bg-git](./img/bg-git-draft.png)

The next step is to finalize the release, but before hitting the `Publish` button to do so, please read the section on [preparing a release](#step-2-finalize-release). Once ready, you may click the `Publish` to make it the `latest` release.

The `xvp-infra-core-release.git` resource will have a list of published releases to choose from.

**To choose a release version perform the following steps:**

1. Open the `xvp-infra-core-release.git` resource on the `stg-bg` tab
2. Pin the published release version that needs to be deployed
3. Run the `update-vault-release-version-stg` job
4. Vault will update the `release_version` secret with the version from the specified pinned release

Once a release is cut and the Vault secret has been updated, the clusters can finally be built by following the [build clusters steps](#build-clusters)

#### **Cutting a release refresh**

Prior to cutting the release it is best practice to bring the release up in #xvp-infra for XVP organization awareness. This will ensure that teams are aligned and able to coordinate multiple changes from different members.

When cutting a release in staging for an in-place/refresh deployment please follow the guidelines in the section on [preparing a release](#step-2-finalize-release).

#### QA Testing

**Once the changes are deployed to staging, you must:**

- Reach out to XVP QA and make them aware of the changes
- Provide the draft release notes
- Specifically call out any breaking changes to infrastructure and services

We cannot schedule a production deployment and finalize release until QA has given the green light to XVP Infra team.

Once ready, you may click the `Publish` button to make it the `latest` release; note that your changes should be well-tested before this point by both QA and infra teams.

### Step 2: Finalize Release

Here you have the opportunity to edit the release notes by hand before finalizing the release. Please ensure that if a Jira ticket is included in the Pull Request Title that there is a link to the issue in Jira. Also take care that PR titles are descriptive and the summary as a whole is organized. Remember that once a release has been deployed to all production regions, an email with the release notes is sent to Senior Leadership and other key stakeholders. To do this, select the release draft from the [Releases page](https://github.com/comcast-xvp/xvp-infra-core/releases) and hit the **Edit draft** button. When complete, share the draft with xci-infra-team for approval. Once the team is in agreement, you can **Publish release** at the bottom which will tag the Git repository: `xvp-eks-shared-vX.X.X`. This makes the release available as a xvp-infra-core-release.git resource for the [production deployment jobs](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=prod)

[A good example can be found here](https://github.com/comcast-xvp/xvp-infra-core/releases/tag/xvp-infra-core-eks-v0.0.90)

### Step 3: Create IOP Ticket

After a staging deployment is complete and validated, an IOP ticket is **required** to deploy to production. This will need 2 types of approvals: peer and ccb. Please see below for more information.

Follow the process outlined in the [XVP Docs](https://internal-xvp-docs-staging.r53.aae.comcast.net/Team/xvp-release-management/#paperwork) to schedule your Production Deployment. Use the changelog that is generated in the previous step to populate the change request in IOP. Make sure to list the Concourse jobs you will be running under `Implementation Plan` and be explicit in describing the exact steps of the `Backout Plan`. Under the Reviewers tab, the Peer Reviewer Group should be `TPX-ENT-XCI-PLATFORM_INFRA` and the CCB group should be `Ent-XCI-Platform`. Remember to have the ticket Peer Reviewed by a member of the XCI Platform Infra team then also reviewed in the '#xci-ccb' channel. Once completed, please also post in '#sky-xvp-releases' if the change is being done in EU.

**NOTE:** You will need 1 IOP Ticket per deployment timeframe. For example, if you deploy to US-East-1 on Monday, then US-East-2 on Tuesday, you will need 2 separate IOP Tickets. If you deploy to EU-West-1 and AP-Southeast-2 in the morning and US-West-2 in the afternoon, then you will need 2 separate IOP tickets (morning and afternoon). Typically, the team deploys to 1 or more US regions first before deploying to the EU for stability. The team also avoids deploying to all regions in a continent on the same day (e.g., not deploying both EU-Central-1 and EU-West-1 on the same day)

More information on the IOP creation process can also be found [here](https://etwiki.sys.comcast.net/display/XCIPT/IOP+Tickets)

#### IOPs during an active moratorium

```text
Note: If there are active moratoriums but the deployment needs to go out, the IOP ticket will require SLT approval. Please notify both Eric Lochstet and Matt Izbicki to get approved.
```

#### IOP in Slack

Once a IOP is created, post it in #xvp-infra by using the `/getchange` command

Run `/getchange $CHGNUMBER` where $CHGNUMBER is the IOP ticket ID

![slack-iop](./img/slack_iop.png)

- Tag the XVP Infra team to get IOP approvals
- It lets everyone in #xvp-infra know of changes coming

### Step 4: Deployment Calendar & Calendar Invites

Once the IOP tickets are scheduled and approved, add them to the [XCI Platform Infra deployment calendar]((https://etwiki.sys.comcast.net/pages/viewpage.action?spaceKey=XCIPT&title=Deployment+Schedule+for+XCI+Infra)). Make sure to include details on the time of deployment, target region, release version, and IOP change request number. Several members from both XCI Infra and XVP look at this.

Additionally, send a calendar invite in outlook for each deployment to [xci_platforms_infrastructure@Comcast.com](xci_platforms_infrastructure@Comcast.com) group. This will send the invite to everyone on the team. Please also add any other stakeholders or anyone that should be present to validate (e.g., QA or individuals who have changes in the release). The Microsoft Teams invite box must be checked in order to create a Teams meeting.

### Step 5: Deploy to Production

***Announce the changes in #sky-xvp-releases and #xvp-infra before starting to get XVP attention as there may be confliction in starting times or an issue arises that requires the team to pause and reasses.***

If your IOP has been approved you are clear to run the production deployment on the scheduled date and time. The major change in the pipeline is that rather than `HEAD` of the main branch being deployed, the current (or pinned) tag will be used. Upon completion of the deployment to all Production regions, the changelog will be emailed out to stakeholders.

**NOTE: If you have a Jira ticket for tracking the production deployment, please attach the link to any IOP(s) related to the deployment. Additionally, if you have screenshots related to the deployment (e.g., improvement in cache hit rate), then it can be good practice to attach those both to the IOP and Jira ticket.**

## Production Deployment In-Place/Refresh

Please ensure the following steps are completed prior to deploying to production:

```text
Checklist:
1. Deployed desired changes from `main` branch to *all* lower environments
   -  Must have successful deployments in plat-dev, dev, and staging
    - Pods are in running state
    - EKS or K8s changes are working
    - Metrics and logs are returning as expected
    - QA have tested and signed off that infrastructure changes did not have a negative impact on XVP services
2. Release notes published and deployment calendar updated
3. Changes announced in #xvp-infra and tagged XVP tech leads
4. MS Teams meeting scheduled for deployment
5. IOP ticket in approved status
6. No active Sev 0/1s on day of deployment
7. Monitor dashboards, alerts, and support Slack channels
  - Slack Channels: #xvp-alerts, #xvp-infra-alerts, #xci-critical-alerts, #xvp-support-devs, #xci-ccb
  - Grafana, Blackbird, ELK
```

Once the checklist above has been verified, the production deployment can start.

1. Navigate to the `prod` tab in the `xvp-infra-core-eks` pipeline [here](https://ci.comcast.net/teams/xvp/pipelines/xvp-infra-core-eks?group=prod).

2. Verify the `xvp-infra-core-release.git` resource is pinned to the correct release.

3. Deploy to one region and let the changes sit for at least 24 hours to verify that the latest production deployment did not introduce regressions to the XVP platform.

4. Verify with QA and XVP tech leads before creating an IOP ticket to deploy to other regions.

5. Once the deployment is finished in all production regions, an email with the release notes will be sent out to all XVP Infra team, SLT, and other stakeholders. Please verify that you have received this email.

## Production Deployment Blue/Green

Please ensure the following steps are completed prior to deploying to production:

```text
Checklist:
1. Deployed desired changes from `main` branch to *all* lower environments
   -  Must have successful deployments in plat-dev, dev, and staging
2. Change(s) in the new blue/green clusters are working as expected
   - New clusters respond back healthy
   - Pods are in running state
   - EKS or K8s changes are working
   - Metrics and logs are returning
3. Release notes published and deployment calendar updated
4. MS Teams meeting scheduled for deployment
5. IOP ticket in approved status
6. No active Sev 0/1s on day of deployment
7. Monitor dashboards, alerts, and support Slack channels
  - Slack Channels: #xvp-alerts, #xvp-infra-alerts, #xci-critical-alerts, #xvp-support-devs, #xci-ccb
  - Grafana, Blackbird, ELK
```

Once the checklist above has been verified, the production deployment can start.

1. Navigate to the `prod-bg` tab in the `xvp-infra-core-eks` pipeline

    ![Prod-BG](./img/prod-bg.png)

    ```text
    Note: Once the Git Release Notes have been published, the `xvp-infra-core-release.git` resource is updated, this then auto-triggers the `update-vault-release-version-prod` secret
    ```

2. Verify the `xvp-infra-core-release.git` resource is pinned to the correct release
3. Verify the `update-vault-release-version-prod` job ran automatically
4. Verify the `release_version` secret in Vault for prod path

    - The version value should match the verion in the release notes

5. Follow the [build cluster](#build-clusters) steps

    - i.e. Run the `build-cluster-from-release-prod-${aws_region}` jobs and connect to mesh

6. Once the cluster is built and looks good, deploy services to it
7. Run service smoke tests and have QA validate

    ***IMPORTANT NOTE***

    Before running the DNS Promotion jobs, please make sure that the clusters are healthy, metrics are flowing, and XVP services are deployed and also healthy.

    If at any point throughout the production deployment there are signs of smoke, concerns raised, or an increase in errors, **stop the deployment**

    Assess the situation, and if need be, **destroy** the cluster to stop alerts and noise.

8. Run DNS Promotion jobs
9. Verify XVP services are hitting the new NLB in the new blue/green clusters
10. Run `bluegreen-release-email` to send out email to stakeholders with release notes
11. Deployment complete, close IOP ticket(s)

### Rollback Production Deployment Refresh

In case a rollback is necessary due to issues related to the latest deployment, please prioritize rolling back first and foremost. Ideally, any rollbacks occur before primetime windows.

1. Communicate in #xvp-infra that a rollback will take place

2. Navigate to the `prod` tab in the `xvp-infra-core-eks` pipeline

3. Pin the `xvp-infra-core-release.git` resource to the previous Release version

4. Depending on what is needed please revert region by region using the refresh jobs starting with US, then AP, and finally EU

5. Please verify the revert was succesful and you are no longer seeing any issues

### Helpful Dashboards & Links for monitoring

[xvp-overall-dashboard](https://dashboard.io.comcast.net/d/XGNW87Snk/xvp-overall-dashboard?orgId=409)

[xvp-infra-dashboard](https://dashboard.io.comcast.net/d/XKA_TIo7z12/xvp-infra-dashboard?orgId=409)

[Troubleshoot](troubleshoot.md)
