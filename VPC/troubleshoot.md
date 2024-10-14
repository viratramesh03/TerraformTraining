# Troubleshoot

In addition to the XVP triage document found [here](https://github.com/comcast-xvp/xvp/blob/main/docs/Platform/k8s/triage.md) this growing document outlines issues that engineers may run into during development and deployment of infrastructure-related items.

## "Failed to assign an IP address to container" error

During a deployment of shared components into a EKS cluster Terraform may return an error message such as:

```bash
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container
"bec8509502cb3484d39770552267d97f2eefc86d3b572aa2dc9e08a088d572cc" network for pod
"aws-load-balancer-controller-6666448b7d-xmw94": networkPlugin cni failed to set up pod
"aws-load-balancer-controller-6666448b7d-xmw94_kube-system" network: add cmd: failed to assign an
IP address to container
```

which indicates that EKS was unable to assign an IP address to a pod for the shared node group. Terraform will error out and abort the apply job with the error message above.

### Verify IP usage

To view how many IPs are being used in a node one can start an interactive bash session on the `aws-node` pod:

```bash
kubectl exec $AWS_NODE_NAME -n kube-system --stdin --tty -- /bin/sh
```

Then run the following curl command:

```bash
curl http://localhost:61679/v1/enis | python -m json.tool
```

which will output (a snippet):

```bash
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2618    0  2618    0     0  2556k      0 --:--:-- --:--:-- --:--:-- 2556k
{
    "AssignedIPs": 9,
    "ENIs": {
        "eni-04efb702fd4d77f5f": {
            "DeviceNumber": 1,
            "ID": "eni-04efb702fd4d77f5f",
            "IPv4Addresses": {
                "198.18.32.83": {
                    "Address": "198.18.32.83",
                    "IPAMKey": {
                        "containerID": "d6bd083252235b6ef4479ce06c50de78c14c2e2e2e0958e66308c9fc1e578106",
                        "ifName": "unknown",
                        "networkName": "_migrated-from-cri"
                    },
                    "UnassignedTime": "0001-01-01T00:00:00Z"
                },
                "198.18.33.72": {
                    "Address": "198.18.33.72",
                    "IPAMKey": {
                        "containerID": "4f7c32f4af80ca2d59022f85023e0321116b9a94feea4438aaab3c147e4b692f",
                        "ifName": "unknown",
                        "networkName": "_migrated-from-cri"
                    },
                    "UnassignedTime": "0001-01-01T00:00:00Z"
                },
                "TotalIPs": 10
```

In the output above you see that there is a total of 10 IPs available for the node but `9` is already in use, so in this example if Terraform told the Kubernetes control plane to schedule 2 new pods it would only create one and throw an error because at that point it has exhausted all available IPs.

More info on troubleshooting node group found [here](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md#ipamd-debugging-commands)

You can inspect the pods on the cluster and see if there are opportunities to clean up old/unused pods and/or see if there is a way to pack the pods on another node.

## Destroying Istio throws `context deadline exceeded` error

If you ever need to run the `istio-terraform-destroy-$env` job to clean up or for other reasons the job may time out with a `context deadline exceeded` message and turn red. The finalizer for the `istio-system` namespace runs into issues during the delete.

To unblock and perform the delete, one must find out what is preventing the delete to complete. In this instance the Terraform job will show

```bash
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m0s elapsed]
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m10s elapsed]
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m20s elapsed]
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m30s elapsed]
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m40s elapsed]
kubernetes_namespace.istio_system_ns: Still destroying... [id=istio-system, 4m50s elapsed]
```

and will time out after 5 minutes with the error message. This shows that some resource it preventing the namespace from being deleted.

To find if a resource is not available you can look for the `apiservice` it is associated with,

```bash
kubectl get apiservice | grep False
```

if the command above returns a service that is `False` and mentions `istio` you can remove it by:

```bash
kubectl delete apiservice $APISERVICE_NAME
```

then re-run the terraform destroy job. If it still gives you issues then you can run the following command to edit the finalizer.

**NOTE: Editing the finalizer is not a best practice but is known to fix the issue.**

More information on it can be found [here](https://github.com/kubernetes/kubernetes/issues/60807#issuecomment-524772920)

```bash
(
NAMESPACE=istio-system
kubectl proxy &
kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >temp.json
curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
)
```

then re-run the job and the destroy job will turn green. Then you can go ahead and re-apply istio to add it to the EKS cluster

## Launch containers in the cluster

Sometimes, it may be useful to launch an interactive environment in the cluster for troubleshooting.

### Run `alpine` in shared namespace with Istio Injection (envoy proxy)

```bash
kubectl run -n shared --rm -it --image=alpine --annotations="sidecar.istio.io/inject=true" --labels="sidecar.istio.io/inject=true" -- alpine
# When the container launches you can install utilities:
/ $ apk update && apk add curl tcptraceroute
```

### Run `busybox` interactively on specific node

```bash
kubectl run -it --rm tr-busybox --image=busybox  --overrides='{ "apiVersion": "v1", "spec": { "nodeName": "ip-96-102-29-113.us-east-2.compute.internal" } }' -- sh
```

### Shell into existing pod

```bash
kubectl exec --stdin --tty <pod-name> -- /bin/bash
```

## Terraform Plan Error: Failed to retrieve output

![Screenshot](./img/terraform-failed-to-retrieve-output.png)

### Error Message Detailed

```terraform
Failed To Run Terraform Plan! Plan Error: Failed to retrieve output.
```

### Description

You may encounter the above error while running terraform plan-apply jobs in the service pipelines. Generally speaking, when this error occurs it is a sign that there is an issue with the terraform state file located in the corresponding s3 bucket.

### Solution

The most straightforward way to resolve this error is to run the terraform apply job only (omitting the plan). When the apply runs, it will resolve the terraform state file and you should be able to continue with the plan apply jobs as usual.

### Steps to implement solution

1. In the command line, run "fly -t xvp login"
2. Then run "fly -t xvp get-pipeline -p $PIPELINE-NAME-HERE > $FILENAME.yaml"
3. Open the file in a code editor and comment out the plan job only for the region and environment where the job is failing. If you are unsure about what to remove ask another member of the team to confirm.
    1. Example - In reference to the above screenshot, you would search for the "terraform-plan-apply-stg-tsf-us-east-2" job and remove the put step with a parameter of "plan_only: true"
    2. Note - Also make sure to comment out any references of the plan job from the groups otherwise you may see errors such as:

        ```terraform
        no jobs match 'terraform-plan-plat-dev-us-east-2' for group 'plat-dev'
        ```

    3. Note - Make sure to comment out the "passed" step of the terraform apply job (if it contains the plan job) otherwise you may see errors such as:

        ```terraform
        jobs.terraform-apply-plat-dev-us-east-2.plan.do[0].in_parallel.steps[1].get(version).passed: unknown job 'terraform-plan-plat-dev-us-east-2'
        ```

4. Pause the "update-pipeline" job of the pipeline that is having this issue (ensuring you don't accidentally push any unwanted changes)
5. After saving your changes to the yaml file, run "fly -t xvp set-pipeline -p $PIPELINE-NAME-HERE -c $FILENAME.yaml"
6. Run a new build of the previously failing job, it should pass this time
7. Once you are finished, unpause the "update-pipeline" job from step 4 and run a new build for that job, which will undo your changes from step 5 and restore the pipeline config.

## Increased latency between services

XVP services communicating intra-cluster _should_ result in latency values below 10ms, however sometimes we see latency increase to 5-10s in a specific region. The cause of this is an issue with an individual Istio control plane pod becoming "out-of-sync". This results in Istiod providing "remote" endpoints to "local" clusters.

### Istiod out of sync

Normally, when you describe a pods "endpoints", you'll see a combination of `198.18.x.x:8080` endpoints, which represent local services, and `96.x.x.x:15443` endpoints which represent the remote services exposed via EastWest Gateway:

For example, here are the mindata pods:

```bash
$ k get pods -n xvp-mindata-api-prod -owide
NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE                                           NOMINATED NODE   READINESS GATES
canary-release-54dfbb4d95-4pqcx   2/2     Running   0          36h   198.18.18.55    ip-96-102-243-92.us-west-2.compute.internal    <none>           <none>
canary-release-54dfbb4d95-59rbb   2/2     Running   0          36h   198.18.13.197   ip-96-102-243-33.us-west-2.compute.internal    <none>           <none>
canary-release-54dfbb4d95-6whjt   2/2     Running   0          19h   198.18.1.69     ip-96-102-242-206.us-west-2.compute.internal   <none>           <none>
canary-release-54dfbb4d95-d7fw7   2/2     Running   0          17h   198.18.2.65     ip-96-102-242-206.us-west-2.compute.internal   <none>           <none>
canary-release-54dfbb4d95-ggkm4   2/2     Running   0          28h   198.18.8.232    ip-96-102-243-26.us-west-2.compute.internal    <none>           <none>
canary-release-54dfbb4d95-njrn2   2/2     Running   0          27h   198.18.19.192   ip-96-102-243-100.us-west-2.compute.internal   <none>           <none>
```

And then you can see Session has endpoints for all of these local pods, as well as the mindata service in remote clusters:

```bash
$ istioctl proxy-config endpoints canary-release-84c6cbf454-5qt52.xvp-session-api-prod | grep mindata-api-prod-rollout | grep 8080
198.18.1.69:8080                                        HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local # <-- Local
198.18.13.197:8080                                      HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
198.18.18.55:8080                                       HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
198.18.19.192:8080                                      HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
198.18.2.65:8080                                        HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
198.18.8.232:8080                                       HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
96.103.168.209:15443                                    HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local # <-- Remote
96.103.169.6:15443                                      HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
96.103.169.73:15443                                     HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
96.106.36.207:15443                                     HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
96.106.37.27:15443                                      HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
96.106.37.79:15443                                      HEALTHY     OK                outbound|8080||xvp-mindata-api-prod-rollout.xvp-mindata-api-prod.svc.cluster.local
```

When this issue occurs, you will see "local" endpoints that do not map to "local" pods. This causes routing issues as traffic is sent to IP addresses with no route to them.

You will also see this happen with local pods that are not actually in the cluster and can follow the exact same steps to resolve the issue. In this example we found a us-east-2 pod with its in private ip in us-east-1 route which will do the exact same thing and follow the same symptoms at a smaller scale where when calling the service occasionally you will see a request take a very long time since the request will just time out.

```bash
$ istioctl proxy-config endpoints canary-release-57bcb655ff-spjxx.xvp-session-api-stg | grep session-api-stg-rollout | grep 8080
198.18.38.66:8080                                       HEALTHY     OK                outbound|8080||xvp-session-api-stg-rollout.xvp-session-api-stg.svc.cluster.local
198.18.39.113:8080                                      HEALTHY     OK                outbound|8080||xvp-session-api-stg-rollout.xvp-session-api-stg.svc.cluster.local
198.18.73.16:8080                                       HEALTHY     OK                outbound|8080||xvp-session-api-stg-rollout.xvp-session-api-stg.svc.cluster.local <- us-east-2 Pod
96.102.239.243:15443                                    HEALTHY     OK                outbound|8080||xvp-session-api-stg-rollout.xvp-session-api-stg.svc.cluster.local
96.103.202.241:15443                                    HEALTHY     OK                outbound|8080||xvp-session-api-stg-rollout.xvp-session-api-stg.svc.cluster.local
```

### Identifying Istiod out of sync

When a service reports higher than usual latency, we'll want to identify if this issue is the cause. We'll first want to look at the proxy status for the source service, Session in this example:

```bash
$ istioctl proxy-status | grep session
canary-release-84c6cbf454-29m2x.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-4czcc.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-7g4ck     1.16.3
canary-release-84c6cbf454-4znsf.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-5h2gd.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-7g4ck     1.16.3
canary-release-84c6cbf454-5qb5q.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-7g4ck     1.16.3
canary-release-84c6cbf454-5rq8j.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
```

While everything here is `SYNCED`, which is a good sign, we also notice there are two different `istiod` pods performing the syncs. So we'll want to analyze a pod from each to see if there are any discrepancies:

```bash
$ istioctl proxy-config endpoints canary-release-84c6cbf454-29m2x.xvp-session-api-prod | wc -l
    1711
$ istioctl proxy-config endpoints canary-release-84c6cbf454-4czcc.xvp-session-api-prod | wc -l
    2511
```

As you can see, the `canary-release-84c6cbf454-4czcc` pod has **significantly** more endpoints than the `canary-release-84c6cbf454-29m2x`, indicating there is a discrepancy. Since the `4czcc` pod was synced by `istiod-688dbf8f67-7g4ck`, this identifies the "bad" `istiod` pod.

### Fixing Istiod out of sync

To resolve this issue, simply delete the "bad" istiod pod.

```bash
$ k delete pods -n istio-system istiod-688dbf8f67-7g4ck
pod "istiod-688dbf8f67-7g4ck" deleted
```

Since we have multiple `istiod` replicas, there is no downtime associated with recycling one pod. All of the pods will be synced with the "good" pod initially, and allow the new pod time to come up and share the load

```bash
$ istioctl proxy-status | grep session
canary-release-84c6cbf454-29m2x.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-4czcc.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3 # <-- Synced with "old" istiod pod
canary-release-84c6cbf454-4znsf.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-5h2gd.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-5qb5q.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
canary-release-84c6cbf454-5rq8j.xvp-session-api-prod           prod-us-west-2     SYNCED     SYNCED     SYNCED     SYNCED     NOT SENT     istiod-688dbf8f67-ps6qn     1.16.3
```

### Prometheus-agent Helm Failures

When building a new cluster be careful with Prometheus-agent Helm Failures as well as the cluster autoscaler failing.
This happens when the OIDC is not deleted properly after the cluster is destroyed it is best practice to delete the OIDC after you delete your cluster
[found here](https://github.com/cloud-cre/AWSOIDCUpdater/tree/main/utility) and making sure that actually gets done.

However this for if you do not delete the old OIDC before building the new cluster since it will screw a little bit up.

First thing you will want to do is delete the old OIDC if it ask you to delete extra roles please do that. They will be recreated later with the new oidc.

```bash
python3 OIDCUtility.py --account-number <12 digit aws account number> --region <EKS Cluster Region> --cluster-name <Cluster Nam or OIDC ID> --operation delete --profile saml
```

The next step will make sure there is a new working OIDC whe can do that by using the create operation

```bash
python3 OIDCUtility.py --account-number <12 digit aws account number> --region <EKS Cluster Region> --cluster-name <Cluster Name> --operation create --profile saml
```

After that we can head back to concourse and re run your specified build job once again.

Once that is done you can delete the old cluster-autoscaler-aws-cluster-autoscale & ebs-csi-controller

Then you should see all pods coming back happy and healthy

### Prometheus-Metrics Failure Causing Scaling Issues

When Prometheus instances fail and Keda can no longer pull from those metrics, it can cause failures in pod scaling. This can lead to pods scaling up, scaling down, and terminating which can cause latency and dropped connections. To resolve this issue we want the services to stay stable, preferably scaled at the max amount of pods each service requires. This issue has occurred historically when a metric with a large degree of cardinality has been introduced to an xvp-service causing excessive querying and storage of additional metrics.

If this situation occurs, you may see the following example error messages when retrieving pod or service logs:

```bash
Events:
  Type     Reason            Age                From           Message
  ----     ------            ----               ----           -------
  Warning  KEDAScalerFailed  21m (x9 over 22m)  keda-operator  prometheus query api returned error. status: 502 response: <html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
</body>
</html>
  Warning  KEDAScalerFailed  91s (x1053 over 166m)  keda-operator  prometheus query api returned error. status: 503 response: Service Unavailable
``````

```bash
Message:  At least one trigger is falling back on this scaled object
    Reason:   FallbackExists
    Status:   True
    Type:     Fallback
    Status:   Unknown
    Type:     Paused
```

```bash
Message: unable to get external metric xvp-disco-api-prod/s1-prometheus-prometheus/&LabelSelector{MatchLabels:map[string]string{scaledobject.keda.sh/name: scaled-canary-release,},MatchExpressions:[] │
│ LabelSelectorRequirement{},}: unable to fetch metrics from external metrics API: rpc error: code = Unknown desc = error when getting metric values no matching metrics found for s1-prometheus-prometheus
```

```bash
xvp-disco-api-prod           59s         Warning   FailedCreatePodSandBox    pod/canary-release-5b4bd6b7c7-9cpbc              Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox : plugin type="aws-cni" name="aws-cni" failed (add): add cmd: failed to assign an IP address to container
xvp-disco-api-prod           59s         Warning   FailedCreatePodSandBox    pod/canary-release-5b4bd6b7c7-r4lrw              Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox : plugin type="aws-cni" name="aws-cni" failed (add): add cmd: failed to assign an IP address to container
```

In order to scale the service to the max amount of pods it requires, we can can take a few immediate actions to avoid service disruption. We should plan to manually scale service pods to their maximum amount.

**PLEASE NOTE: This amount should be confirmed with service teams and we should also request and receive permission from managers/leaders if we are performing manual actions in live environments like `production`**.

To scale an individual services pods we can use the following command:

```bash
REPLICAS=3
NAMESPACE=xvp-ping-api-plat-dev
kubectl patch scaledObject -n $NAMESPACE scaled-canary-release --patch "{\"spec\":{\"minReplicaCount\": $REPLICAS ,\"maxReplicaCount\": $REPLICAS, \"fallback\": { \"replicas\": $REPLICAS}}}" --type=merge
```

To verify these changes have been successfully applied, we can run the following command to output the new scaling configuration of each service:

```bash
kubectl get scaledobjects scaled-canary-release -n xvp-{{service}}-api-prod -o yaml
```

Additionally, you can watch the pods scale up by running the command:

```bash
kubectl get pods -n xvp-{{service}}-api-prod -w
```

Keep in mind that scaling out the services to this extent can put strain on the cluster nodes and should be monitored to make sure the max available nodes has not been reached and are not throttling the service pods. To get the metrics for the cluster nodes we can use the following command:

```bash
kubectl describe nodes -l component=xvp-services-api-prod | grep -A 3 "Resource .*Requests .*Limits"
```

We may also want to check the number of nodes in each region via the AWS console. To find this metric we can go to Amazon EKS -> Clusters -> xvp-eks-prod-{{version}} -> Compute -> Node Groups -> xvp-services-api-prod-{{name}} -> Details. Here we can find `Desired Size`, `Minimum Size`, and `Maximum Size` of the node group. As long as we remain within these limits, we should be able to scale the service pods to the max amount required.
