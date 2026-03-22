One of my first experiences with Kubernetes was with using Prow. Prow is a Kubernetes CI/CD system. In the Mesosphere/D2IQ days, I had to transition the CI jobs that we were running in Jenkins on Mesos/Marathon to Prow on Kubernetes. In order to learn Kubernetes, I suggested we set up Prow to run the CI jobs for our Mesosphere/D2IQ frontend. That was the first task that I undertook that would help me understand Kubernetes . It was simple but a critical CI/CD system that handles job execution and GitHub automation through policy enforcement. Anyone who has either contributed to Kubernetes or worked with the testing-sig will have come across the weird-looking Prow robots working with your pull requests. And in the GitHub-based ChatOps, there are multiple `/slash-commands` that are utilized to interact with the state of the pull request.

The component listeners listen to these webhooks and act on the state and the command. Just like regular CI/CD systems, they trigger a job on the event, and the job usually clones the repository at the pull request HEAD and runs a script that exercises the tests and finally gives a success or failure signal.
The intelligence of running the tests is baked into the script that runs the tests. Prow does not do anything fancy that is expected from any continuous integration system. It simply runs the job and returns the results.

The first task before setting up the jobs was setting up the Kubernetes cluster and then deploying Prow.


![666](Prow-Components.png)
The various nouns in this diagram are the components of the Prow that we will see.

GitHub sends webhooks to Prow. Prow runs your tests as Kubernetes pods. Prow reports the results back to GitHub. That's it. The core pipeline flows down the center: Hook receives GitHub webhooks, creates a **ProwJob** custom resource, the **Controller Manager** picks it up and schedules a pod in the **Cluster**.

So the setup chain was simple.

I used Terraform to manage the infrastructure state. Terraform was used to create the S3 buckets to store the Terraform state. We used to have a DynamoDB table for locking. It surprises me in hindsight, and perhaps even surprised me back then, but we did use a DynamoDB table just for locking. This is to avoid multiple `terraform apply` commands acting on the same cluster. The cluster creation was controlled with users with IAM permissions for various operations.

This is how we built the infrastructure for Kubernetes. Any Kubernetes distribution can be used; at that time, I remember we were using a home-built one, which the company, D2IQ  was selling. For the worker nodes, I had provisioned very large instances — ones with 16 vCPUs and 64 GB of RAM — plenty of headroom for Prow's microservices and the test pods they'd spawn. Since Prow runs the CI job as a Kubernetes job, it needs those resources to spawn the job and run.

I had the cluster infrastructure stored in **Terraform state files**; Terraform created the S3 buckets, DynamoDB lock table, and the IAM users.
For our Kubernetes cluster, I had a **Native K8s cluster** on EC2 using a home-grown technology called Konvoy while at Mesosphere/D2IQ.
I set up the Prow application on the cluster using the Custom Resource Definitions, and since CI systems are often internal, I had to set up an **Internal domain** configured to point to Prow's Hook and Deck endpoints, and set up **cert-manager** for TLS on that internal domain.

![](Prow-State-Flow.png)
Once the Kubernetes cluster was provisioned, the cluster came up cleanly. The nodes were healthy, the API server responding, CoreDNS resolving.

I had to apply the ProwJob Custom Resource Definitions to set up the application. We created two secrets that Prow needs to talk to GitHub: an HMAC (Hash-Based Message Authentication Code, a shared secret) token for validating incoming webhooks, and a GitHub App's private key for authenticating API calls.
I deployed this full set of Prow components using the S3-backed starter manifest.

Once the Prow system was set up, the Prow services would be running. The Hook component would be waiting for webhooks, Deck serving the UI, the Prow Controller Manager ready to schedule jobs. Components like Tide watch for mergeable PRs, a component called Horologium handles periodic jobs. Systems like Crier report back to GitHub, and Sinker quietly cleans up after everyone else.

*Wiring up the domain*

When you architect a CI system for developers, as I did, the primary responsibility is to make the system reliable. Prow needs to be reachable from the public internet — specifically, GitHub needs to send HTTPS webhook requests to Hook's endpoint whenever something happens in the subscribed repositories.

That means in the Kubernetes world, you need a domain, an Ingress, and a valid TLS (Transport Layer Security) certificate.

![](Prow-data-flow.png)

The internal domain had to be provisioned using Terraform, and with the domain I set up the Ingress that routed `/hook` traffic to the Hook Service and everything else to Deck. Even these were provisioned in the Terraform state files.

We used cert-manager to handle the TLS. I pointed an internal domain at the cluster, set up an Ingress that routed `/hook` traffic to the Hook service and everything else to Deck, and used cert-manager to handle TLS. cert-manager would automatically provision and renew certificates, wiring them into the Ingress as a Kubernetes Secret. For cert-manager, we configured a ClusterIssuer, created a Certificate resource for the Prow domain, and let cert-manager do its thing.

All of these were done in Terraform state files, so that when we did a `terraform apply`, the domain was created, the certificate manager was set up, certificates were gathered, and the system was operational serving traffic over HTTPS.
Once the certificate was issued and the Ingress had valid TLS termination, we went to the GitHub App at the org-level, set the webhook URL to point at the Prow domain's `/hook` path, and plugged in the HMAC secret.

It works. Open a test PR, and within seconds the Hook receives the webhook, the trigger plugin creates the ProwJob, the Controller Manager scheduled a pod, the Crier reported results back to GitHub. The green checkmark appearing on the PR status means all is good, and life is good.

Kubernetes Upgrades

Kubernetes upgrades are talked about as a frequent source of pain points. Even when you use a Managed Kubernetes Provider, when you have made complex changes to your cluster and operate business-critical workloads, teams are seldom confident to upgrade their Kubernetes clusters without a good plan in place. So, in my Kubernetes cluster, as we updated our Kubernetes version and Prow version, we had to upgrade the components too. As I explained above, the internal domain over HTTPS was set up with a separate Terraform plan.
And when we upgraded cert-manager, it started to fail. That's when the Hook endpoint stopped receiving webhooks from GitHub.

### The Failure Chain

Here's what happened:

The cert-manager upgrade caused the webhook pod to restart. During the upgrade, cert-manager's own internal webhook — which validates cert-manager CRDs — went through a transition period where it was unavailable. This was due to a bug in the version of cert-manager we were upgrading to.
When cert-manager's cainjector or controller had issues during the upgrade, the Certificate resource couldn't renew and the Secret wasn't properly updated, as the TLS certificate for Prow's Ingress was tied to cert-manager. During the upgrade, the Ingress lost its valid TLS certificate, and without a valid cert, the HTTPS endpoint that GitHub was hitting for webhooks started returning TLS errors.

GitHub webhook deliveries started failing — this can be seen in the GitHub interface showing that webhook deliveries had failed — and all CI/CD stopped. No webhooks meant no presubmit jobs, no postsubmit jobs, no `/retest` commands, nothing. Prow was running perfectly fine internally, but it was invisible to the outside world.

![](The-Failure-Chain.png)



Everything _inside_ the cluster looked healthy:

```bash
# All pods running?

kubectl get pods -n prow
kubectl get pods -n cert-manager

# ProwJobs being created?  No new ones
kubectl get prowjobs -n prow --sort-by=.metadata.creationTimestamp

# Hook logs showing any incoming webhooks? Nothing
kubectl logs -l app=hook -n prow -f

# Certificate status?
kubectl describe certificate prow-tls -n prow
# This is where I finally saw the problem
```

The `kubectl describe certificate` output showed the certificate was in a failed state — cert-manager couldn't issue or renew it because the upgrade had left its internal state inconsistent.

```bash
# Check cert-manager's own health
kubectl get pods -n cert-manager
kubectl logs -l app.kubernetes.io/name=webhook -n cert-manager
```

The webhooks were unhealthy, so I had to delete and reinstall cert-manager cleanly.

I was doing everything using Terraform. Manually mutating state when everything else is maintained in Terraform was going to be tricky. Terraform does not like that. Also, certain versions and dependencies of cert-manager were a problem, so I had to find the right combination of versions and encode it.

```bash
# If the webhook is unhealthy, you may need to
# delete and reinstall cert-manager cleanly

kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v<OLD>/cert-manager.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v<NEW>/cert-manager.yaml
```

Once everything was stable, I had to wait for the cert-manager rollout to stabilize and force the certificate renewal.

```bash
# Wait for everything to stabilize
kubectl -n cert-manager rollout status deployment/cert-manager-webhook

# Force certificate renewal
kubectl delete secret prow-tls-secret -n prow
# cert-manager will recreate it from the Certificate resource

# Verify
kubectl describe certificate prow-tls -n prow
# Should show "Certificate is up to date and has not expired"
```

Once cert-manager was stable, the GitHub webhook deliveries should be back and hooks succeeding again.

```bash
kubectl logs -l app=hook -n prow -f
# Should see incoming webhook events
```

This downtime affected the productivity of my group for multiple hours. And fixing something when things were down was a challenging one.

*Some of the Important Lessons I have learned from this experience are*

Cert-manager upgrades are not zero-downtime for your services.

When you upgrade cert-manager, its internal admission webhook restarts. During that window, if any Certificate resources need to be reconciled, things can break. If your Ingress TLS depends on cert-manager and the timing is bad, your endpoints go dark.
The best way would be to decouple the TLS from cert-manager for critical webhook endpoints during the upgrade.

The Prow Hook endpoint is the single point of entry.

Everything in Prow depends on Hook receiving webhooks. If Hook is unreachable, whether due to TLS issues, DNS problems, or network configuration, the CI/CD pipeline goes silent. This is an important design consideration for anyone designing an application architecture on Prow.

Having a "break glass" procedure for webhook endpoints.

As in, you can manually create a TLS secret and attach it to your Ingress without cert-manager. When cert-manager is the thing that's broken, you need a way to get your endpoints back up while you fix it:

```bash
# Create a self-signed cert to restore connectivity
openssl req -x509 -nodes -days 30 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=prow.your-internal-domain.com"

kubectl create secret tls prow-tls-secret \
  --cert=/tmp/tls.crt --key=/tmp/tls.key \
  -n prow --dry-run=client -o yaml | kubectl apply -f -
```


This was a firefighting story that helped me realize how infrastructure and complex applications stand on some very simple dependencies.

![](xkcd-prow-upgrade.png)
