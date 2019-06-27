# gitops-istio

This guide walks you through setting up Istio on a Kubernetes cluster and 
automating canary deployments with GitOps pipelines.

![Progressive Delivery GitOps Pipeline](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/diagrams/flagger-gitops-istio.png)

Components:

* **Istio** service mesh
    * manages the traffic flows between microservices, enforcing access policies and aggregating telemetry data
* **Prometheus** monitoring system  
    * time series database that collects and stores the service mesh metrics
* **Flux** GitOps operator
    * syncs YAMLs and Helm charts between git and clusters
    * scans container registries and deploys new images
* **Helm Operator** CRD controller
    * automates Helm chart releases
* **Flagger** progressive delivery operator
    * automates the promotion of canary deployments using Istio routing for traffic shifting and Prometheus metrics for canary analysis

### Prerequisites

You'll need a Kubernetes cluster **v1.11** or newer with `LoadBalancer` support, 
`MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers enabled. 
For testing purposes you can use Minikube with two CPUs and 4GB of memory. 

Install Flux CLI, Helm and Tiller:

```bash
brew install fluxctl kubernetes-helm

kubectl -n kube-system create sa tiller

kubectl create clusterrolebinding tiller-cluster-rule \
--clusterrole=cluster-admin \
--serviceaccount=kube-system:tiller

helm init --service-account tiller --wait
```

Fork this repository and clone it:

```bash
git clone https://github.com/<YOUR-USERNAME>/gitops-istio
cd gitops-istio
```

### Cluster bootstrap

Install Weave Flux and its Helm Operator by specifying your fork URL:

```bash
./scripts/flux-init.sh git@github.com:<YOUR-USERNAME>/gitops-istio
```

At startup, Flux generates a SSH key and logs the public key. The above command will print the public key. 

In order to sync your cluster state with git you need to copy the public key and create a deploy key with write 
access on your GitHub repository. On GitHub go to _Settings > Deploy keys_ click on _Add deploy key_, 
check _Allow write access_, paste the Flux public key and click _Add key_.

When Flux has write access to your repository it will do the following:

* creates the `istio-system` and `prod` namespaces
* creates the Istio CRDs
* installs Istio Helm Release
* installs Flagger Helm Release
* installs Flagger's Grafana Helm Release
* creates the load tester deployment
* creates the frontend deployment and canary
* creates the backend deployment and canary
* creates the Istio public gateway 

![Helm Operator](https://raw.githubusercontent.com/stefanprodan/openfaas-flux/master/docs/screens/flux-helm.png)

The Flux Helm operator provides an extension to Weave Flux that automates Helm Chart releases for it. 
A Chart release is described through a Kubernetes custom resource named HelmRelease. 
The Flux daemon synchronizes these resources from git to the cluster, and the Flux Helm operator makes sure 
Helm charts are released as specified in the resources.

Istio Helm Release example:

```yaml
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: istio
  namespace: istio-system
spec:
  releaseName: istio
  chart:
    repository: https://storage.googleapis.com/istio-release/releases/1.2.0/charts
    name: istio
    version: 1.2.0
  values:
    gateways:
      enabled: true
      istio-ingressgateway:
        type: LoadBalancer
    mixer:
      telemetry:
        enabled: true
    prometheus:
      enabled: true
      scrapeInterval: 5s
```

Note that the Istio CRDs have been extracted from the istio-init chart and placed inside the istio-system dir.
You can update the CRDs by running the `./scripts/istio-init.sh` script.

If you make changes to the Istio configuration and push those to git, Flux will upgrade the Helm release. 
It can take up to 3 minutes for Flux to sync and apply the changes or you can use `fluxctl sync` to trigger a git sync.

### Workloads bootstrap

When Flux syncs the Git repository with your cluster, it creates the frontend/backend deployment, HPA and a canary object.
Flagger uses the canary definition to create a series of objects: Kubernetes deployments, 
ClusterIP services, Istio destination rules and virtual services. These objects expose the application on the mesh and drive 
the canary analysis and promotion.

```bash
# applied by Flux
deployment.apps/frontend
horizontalpodautoscaler.autoscaling/frontend
canary.flagger.app/frontend

# generated by Flagger
deployment.apps/frontend-primary
horizontalpodautoscaler.autoscaling/frontend-primary
service/frontend
service/frontend-canary
service/frontend-primary
destinationrule.networking.istio.io/frontend-canary
destinationrule.networking.istio.io/frontend-primary
virtualservice.networking.istio.io/frontend
```

Check if Flagger has successfully initialized the canaries: 

```
kubectl -n prod get canaries

NAME       STATUS        WEIGHT   LASTTRANSITIONTIME
backend    Initialized   0        2019-04-30T18:53:18Z
frontend   Initialized   0        2019-04-30T17:50:50Z
```

When the `frontend-primary` deployment comes online, 
Flagger will route all traffic to the primary pods and scale to zero the `frontend` deployment.

### Automated canary promotion

Flagger implements a control loop that gradually shifts traffic to the canary while measuring key performance indicators
like HTTP requests success rate, requests average duration and pod health.
Based on analysis of the KPIs a canary is promoted or aborted, and the analysis result is published to Slack.

A canary deployment is triggered by changes in any of the following objects:
* Deployment PodSpec (container image, command, ports, env, etc)
* ConfigMaps mounted as volumes or mapped to environment variables
* Secrets mounted as volumes or mapped to environment variables

For workloads that are not receiving constant traffic Flagger can be configured with a webhook, 
that when called, will start a load test for the target workload. The backend load test webhook configuration can be found
at `prod/backend/canary.yaml`.

![Flagger Load Testing Webhook](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/diagrams/flagger-load-testing.png)

Trigger a canary deployment for the backend app by updating the container image:

```bash
$ export FLUX_FORWARD_NAMESPACE=flux

$ fluxctl release --workload=prod:deployment/backend \
--update-image=quay.io/stefanprodan/podinfo:1.4.1

Submitting release ...
WORKLOAD                 STATUS   UPDATES
prod:deployment/backend  success  backend: quay.io/stefanprodan/podinfo:1.4.0 -> 1.4.1
Commit pushed:	ccb4ae7
Commit applied:	ccb4ae7
```

Flagger detects that the deployment revision changed and starts a new rollout:

```bash
$ kubectl -n prod describe canary backend

Events:

New revision detected! Scaling up backend.prod
Starting canary analysis for backend.prod
Advance backend.prod canary weight 5
Advance backend.prod canary weight 10
Advance backend.prod canary weight 15
Advance backend.prod canary weight 20
Advance backend.prod canary weight 25
Advance backend.prod canary weight 30
Advance backend.prod canary weight 35
Advance backend.prod canary weight 40
Advance backend.prod canary weight 45
Advance backend.prod canary weight 50
Copying backend.prod template spec to backend-primary.prod
Promotion completed! Scaling down backend.prod
```

During the analysis the canary’s progress can be monitored with Grafana. You can access Grafana using port forwarding:

```bash
kubectl -n istio-system port-forward svc/flagger-grafana 3000:80
```

The Istio dashboard URL is 
http://localhost:3000/d/flagger-istio/istio-canary?refresh=10s&orgId=1&var-namespace=prod&var-primary=backend-primary&var-canary=backend

![Canary Deployment](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/demo-backend-dashboard.png)

Note that if new changes are applied to the deployment during the canary analysis, Flagger will restart the analysis phase.

### Automated A/B testing

Besides weighted routing, Flagger can be configured to route traffic to the canary based on HTTP match conditions. 
In an A/B testing scenario, you'll be using HTTP headers or cookies to target a certain segment of your users. 
This is particularly useful for frontend applications that require session affinity.

You can enable A/B testing by specifying the HTTP match conditions and the number of iterations:

```yaml
  canaryAnalysis:
    # schedule interval (default 60s)
    interval: 10s
    # max number of failed metric checks before rollback
    threshold: 10
    # total number of iterations
    iterations: 12
    # canary match condition
    match:
      - headers:
          user-agent:
            regex: "^(?!.*Chrome)(?=.*\bSafari\b).*$"
      - headers:
          cookie:
            regex: "^(.*?;)?(type=insider)(;.*)?$"
```

The above configuration will run an analysis for two minutes targeting Safari users and those that 
have an insider cookie. The frontend configuration can be found at `prod/frontend/canary.yaml`.

Trigger a deployment by updating the frontend container image:

```bash
$ fluxctl release --workload=prod:deployment/frontend \
--update-image=quay.io/stefanprodan/podinfo:1.4.1
```

Flagger detects that the deployment revision changed and starts the A/B testing:

```bash
$ kubectl -n istio-system logs deploy/flagger -f | jq .msg

New revision detected! Scaling up frontend.prod
Waiting for frontend.prod rollout to finish: 0 of 1 updated replicas are available
Advance frontend.prod canary iteration 1/10
Advance frontend.prod canary iteration 2/10
Advance frontend.prod canary iteration 3/10
Advance frontend.prod canary iteration 4/10
Advance frontend.prod canary iteration 5/10
Advance frontend.prod canary iteration 6/10
Advance frontend.prod canary iteration 7/10
Advance frontend.prod canary iteration 8/10
Advance frontend.prod canary iteration 9/10
Advance frontend.prod canary iteration 10/10
Copying frontend.prod template spec to frontend-primary.prod
Waiting for frontend-primary.prod rollout to finish: 1 of 2 updated replicas are available
Promotion completed! Scaling down frontend.prod
```

You can monitor all canaries with:

```bash
$ watch kubectl get canaries --all-namespaces

NAMESPACE   NAME      STATUS        WEIGHT   LASTTRANSITIONTIME
prod        frontend  Progressing   100      2019-04-30T18:15:07Z
prod        backend   Succeeded     0        2019-04-30T17:05:07Z
```

### Automated rollback

During the canary analysis you can generate HTTP 500 errors and high latency to test Flagger's rollback.

Generate HTTP 500 errors:

```bash
watch curl -b 'type=insider' http://<INGRESS-IP>/status/500
```

Generate latency:

```bash
watch curl -b 'type=insider' http://<INGRESS-IP>/delay/1
```

When the number of failed checks reaches the canary analysis threshold, the traffic is routed back to the primary, 
the canary is scaled to zero and the rollout is marked as failed.

```text
kubectl -n prod describe canary/frontend

Status:
  Failed Checks:         2
  Phase:                 Failed
Events:
  Type     Reason  Age   From     Message
  ----     ------  ----  ----     -------
  Normal   Synced  3m    flagger  Starting canary deployment for frontend.prod
  Normal   Synced  3m    flagger  Advance frontend.prod canary iteration 1/10
  Normal   Synced  3m    flagger  Advance frontend.prod canary iteration 2/10
  Normal   Synced  3m    flagger  Advance frontend.prod canary iteration 3/10
  Normal   Synced  3m    flagger  Halt frontend.prod advancement success rate 69.17% < 99%
  Normal   Synced  2m    flagger  Halt frontend.prod advancement success rate 61.39% < 99%
  Warning  Synced  2m    flagger  Rolling back frontend.prod failed checks threshold reached 2
  Warning  Synced  1m    flagger  Canary failed! Scaling down frontend.prod
```

### Alerting

Flagger can be configured to send Slack notifications.
You can enable alerting by adding the Slack settings to Flagger's Helm Release:

```yaml
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: flagger
  namespace: istio-system
  annotations:
    flux.weave.works/automated: "false"
    flux.weave.works/tag.chart-image: semver:~0
spec:
  releaseName: flagger
  chart:
    repository: https://flagger.app
    name: flagger
    version: 0.16.0
  values:
    slack:
      user: flagger
      channel: general
      url: https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

Once configured with a Slack incoming **webhook**, Flagger will post messages when a canary deployment 
has been initialised, when a new revision has been detected and if the canary analysis failed or succeeded.

![Slack Notifications](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/slack-canary-notifications.png)

A canary deployment will be rolled back if the progress deadline exceeded or if the analysis reached the 
maximum number of failed checks:

![Slack Notifications](https://raw.githubusercontent.com/weaveworks/flagger/master/docs/screens/slack-canary-failed.png)

Besides Slack, you can use Alertmanager to trigger alerts when a canary deployment failed:

```yaml
  - alert: canary_rollback
    expr: flagger_canary_status > 1
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Canary failed"
      description: "Workload {{ $labels.name }} namespace {{ $labels.namespace }}"
```

### Getting Help

If you have any questions about progressive delivery:

* Invite yourself to the [Weave community slack](https://slack.weave.works/)
  and join the [#flagger](https://weave-community.slack.com/messages/flagger/) channel.
* Join the [Weave User Group](https://www.meetup.com/pro/Weave/) and get invited to online talks,
  hands-on training and meetups in your area.

Your feedback is always welcome!