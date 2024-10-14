# Blue/Green Cluster Deletion Process

- Scale up the cluster that is being deleted in case it is scaled down to the least number of nodes , used for 2,2 for dev,stg and 3,3 for prod in the node groups.
- Remove all XVP services from the inactive cluster with the EKS cleanup script. Ensure you are in the correct cluster context and run the script locally: [here](https://github.comcast.com/chause519/EKSCleanup)
  - Verify:
    - Run  kubectl commands to verify no XVP service pods are running or watch during the deletion process.
    - Confirm that the script completes without any issues.
- Verify the inactive version in the vault is set to inactive version: [here](https://or.vault.comcast.com/ui/vault/secrets/secret/show/xvp/concourse-ci/xvp-infra-core-eks-version/dev)
- Run `destroy-inactive-cluster-${env}-${region}` Concourse job(s).
- For Production, create IOP tickets usually at least 3 days after the migration of services to new clusters. (Reference ticket: CHG006436761)

## Issues we ran in to during the deletion

## Error 1: Workspace "dev" doesn't exist

```bash
Workspace "dev" doesn't exist.
You can create this workspace with the "new" subcommand
or include the "-or-create" flag with the "select" subcommand.
```

There are two ways to fix this errror:

```bash
hijack the container and create workspace.
fly hijack -t xvp --job=xvp-infra-core-eks/destroy-inactive-cluster-stg-us-west-2 sh
```

Edit the pipeline & skip the job if it completed / did everything it needed.Look up for the job in pipeline destroy-inactive-cluster-dev-us-east-2:
remove the below block for skipping

```hcl
  - params:
      terraform_source: xvp-infra-core.git/infra/terraform/xvp-exp-common/k8s
      var_files:
      - xvp-infra-core.git/infra/terraform/xvp-exp-ap-dev/eks/stg/ap-southeast-1/terraform.tfvars
      vars:
        AZURE_AD_SSO_CLIENT_ID: ((oauth2-proxy-stg.client-id))
        --------
        vault-ca-cert-pem: ((istio-ca-certs/vault-root-ca.root-ca-pem))
    put: k8s-terraform-dev_bluegreen-inactive-ap-southeast-1
```

then apply changes using:

```bash
fly -t xvp login
fly -t xvp set-pipeline -p xvp-infra-core-eks  -c xvp-infra-pipeline.yaml.
xvp-infra-pipeline.yaml file is where skip changes are added.
```

## Error 2: Failed to Delete Kubernetes Resource - external-secrets/external-secrets-vault-secret-id

```bash
Error: external-secrets/external-secrets-vault-secret-id failed to delete kubernetes resource: Internal error occurred: failed calling webhook "validate.externalsecret.external-secrets.io": failed to call webhook: Post "https://external-secrets-webhook.external-secrets.svc:443/validate-external-secrets-io-v1beta1-externalsecret?timeout=5s": no endpoints available for service "external-secrets-webhook"
```

To resolve this error, patch the validating webhook configurations and delete specific clusterSecretStores using

```bash
k get validatingwebhookconfigurations
k patch validatingwebhookconfigurations secretstore-validate --type=json -p='[{"op": "remove", "path": "/webhooks/0"}]'
k patch validatingwebhookconfigurations externalsecret-validate --type=json -p='[{"op": "remove", "path": "/webhooks/0"}]'
k delete clusterSecretStore vault-backend-v1
k delete clusterSecretStore vault-backend-v2
```

## Error 3: No Value for Required Variable - cluster_name_prefix

```hcl
│ Error: No value for required variable
│
│   on variables.tf line 40:
│   40: variable "cluster_name_prefix" {
│
│ The root module input variable "cluster_name_prefix" is not set, and has no
│ default value. Use a -var or -var-file command line argument to provide a
│ value for this variable.
```

To resolve this error update pipeline configuration file to include missing variables and compare with working job configurations

```hcl
      XVP_CLOUD_CONFIG_PASSWORD: ((cloud-config-basic-auth-password.password))
      XVP_VAULT_SECRET_ID: ((vault.secret-id))
      cluster_name_prefix: xvp-eks-shared
      service_version: ((xvp-infra-core-eks-version/dev.inactive_version))
```

## Staging

```hcl
|Error: local-exec provisioner error
│
│   with module.aws-update-role-trust.null_resource.aws_update_role_trust,
│   on aws-update-role-trust/main.tf line 19, in resource "null_resource" "aws_update_role_trust":
│   19:   provisioner "local-exec" {
│
│ Error running command './aws-update-role-trust/src/install_python.sh
│ pip3 install -r aws-update-role-trust/src/requirements.txt
│ AWS_DEFAULT_REGION=eu-west-1 python3
│ aws-update-role-trust/src/aws_update_role_trust.py -c xvp-eks-stg-v76 -r
│ CustomerManaged-aws_controller_k8s_iam-role -s ack-iam-controller -n
│ iam-ack remove
│ ': exit status 1. Output: Python3 is not installed
```

Tried to remove trust relationship using update_assume_role_policy and failed to parse policy document .the script adds/removes the trust relationship from the shared role [here](https://us-east-1.console.aws.amazon.com/iam/home?region=ap-southeast-2#/roles/details/CustomerManaged-aws_controller_k8s_iam-role?section=trust_relationships)When the last cluster in the account is removed, it throws an error because the role no longer has any trust relationship. its kind of a bug in the script/proces and its fixed.

## Scripts used during BG

Bash script to check the cluster canary actuator endpoint pointing between blue-green clusters before toggling the active version. The script takes environment and AWS region as inputs.

```bash
#!/bin/bash

# List of services
SERVICES=("disco" "ess" "linear" "mindata" "parental" "playback" "profile" "rights" "saved" "session" "sports" "sportscast" "transactional" "xifa")

# Ask user to input the environment and AWS region
read -p "Enter environment (dev, stg, prod): " ENVIRONMENT
read -p "Enter AWS region (us-east-1, us-east-2, us-west-2, eu-central-1, eu-west-1, ap-southeast-1, ap-southeast-2): " AWS_REGION

# Loop through each service and perform curl request
for SERVICE in "${SERVICES[@]}"; do
    # Set default URL
    URL="https://${SERVICE}.${ENVIRONMENT}.exp.${AWS_REGION}.aws.xvp.xcal.tv/actuator/info"

    # Check if the service is mindata or sportscast
    if [[ "$SERVICE" == "mindata" ]]; then
        URL="https://${SERVICE}-canary.${ENVIRONMENT}.exp.${AWS_REGION}.aws.xvp.xcal.tv:9471/actuator/info"
    elif [[ "$SERVICE" == "sportscast" ]]; then
        URL="https://${SERVICE}-canary.${ENVIRONMENT}.exp.${AWS_REGION}.aws.xvp.xcal.tv:44301/actuator/info"
    fi

    # Perform curl request with the determined URL
    OUTPUT=$(curl -s "$URL")

    # Parse the JSON output to extract region and name
    REGION=$(echo "$OUTPUT" | jq -r '.eks.region')
    NAME=$(echo "$OUTPUT" | jq -r '.eks.name')

    # Print the region and name
    echo "Service: $SERVICE"
    echo "Region: $REGION"
    echo "Name: $NAME"
    echo "-----------------------------------------"
done
```
