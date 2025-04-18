# NATS SuperCluster and Services Framework Demo

This repo builds out and runs requests against a NATS SuperCluster running in AWS and Azure. The goal of this demo:

- show how a SuperCluster can span multiple cloud environments over the internet
- give examples for how to use the Services Framework to build NATS-connected services
- demonstrate how geo-affinity keeps traffic locally-blound when possible to avoid latency

## The folder structure

- `/aks-setup`: OpenTofu and Kubernetes configuration files specific to the Azure environment running in `westus`
- `/eks-setup`: OpenTofu and Kubernetes configuration files specific to the AWS environment running in `us-east-1`
- `/k8s-configs`: the Kubernetes configuration files used to deploy services into both environments
- `/services`: the Python code for the various services that are used for the demo

## Some FYI

This is project meant for demo purposes only. While it employs authenticationa and authorization for all of the exposed endpoints that it creates, the authentication credentials are stored in this repo and are therefore insecure. Deploy this repo only in ad hoc environments that you can stand up and tear down without impacting your production environments. **Do not run this project in its current stat in production.**

## Setting up the environments

Both the Azure and AWS environments are set up using a Bash script, which can be found in their respective folders.

**NOTE: This assumes that you have the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) and [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and configured with a user/role/subscription that has the privileges to create resources in the two regions listed above. You will also need the [eksctl](https://eksctl.io/installation/) CLI installed in order to provision a service account for the AWS Load Balancer Controller.**

**NOTE: This also assumes that you have [OpenTofu](https://opentofu.org/docs/intro/install/) installed, and are running at least version 1.8.** At the time that this was created, the OpenTofu configs don't use any syntax that would impact compatibility with [Terraform](https://www.terraform.io/), so you should be able to use either IaC tool. Consider using [tenv](https://github.com/tofuutils/tenv) if you need to swap between tools or versions.

### Azure

To set up the Azure environment, `cd` into the `/aks-setup` folder and run:

```
sh aks-setup.sh
```

This will:
- run the OpenTofu configs that build out the AKS environment
- add a new context to your default kubeconfig file called `west-cluster` and set it as the current context
- run the Helm install command to build out the NATS cluster under the release name `nats-west`

### AWS

To set up the AWS environment, `cd` into the `/eks-setup` folder and run:

```
sh eks-setup.sh
```

This will:
- run the OpenTofu configs that build out the EKS environment
- add a new context to your default kubeconfig file called `east-cluster` and set it as the current context
- set the default storage class, as EKS doesn't specify one on creation
- creates an IAM service account to for the AWS Load Balancer Controller
- installs the AWS Load Balancer Controller
- run the Helm install command to build out the NATS cluster under the release name `nats-east`

#### A note about EKS

When AWS adds a new context to your default kubeconfig, it uses the AWS CLI to authenticate each request made by kubectl. If you want to use a Kubernetes UI like [Lens](https://k8slens.dev/) to interact with your clusters, you'll need to create a separate kubeconfig file. The script and K8s config file necessary to do that have been provided in `/eks-setup/kubeconfig-setup`. Assuming you have the EKS cluster set as your current context, run the following from the `/eks-setup` directory:

```
kubectl apply -f kubeconfig-setup/admin-sa.yaml
sh kubeconfig-setup/create-sa-token.sh
```

That will create a kubeconfig file tied to a ServiceAccount that has cluster-admin priviliges. You can then load that into Lens as a new kubeconfig file.

## Deploying the Services

When configured against either context, you can deploy the services in bulk by running:

```
kubectl apply -f k8s-configs
```

This will run all four mathmatical services as Deployments with 3 instances, as well as the requester Job. Note that the requester Job will deploy and possibly run before the other services are ready, so you may need to run it again to see intended results.

## NATS Gateways and Cluster Connectivity

In a production environment, you'd likely already have a domain name associated with your K8s cluster ingress that you can map to the service used by the NATS servers.

However, since this is just a demo, we avoided forcing you to provide that domain name in either cloud, and instead are just relying on the auto-generated hostname and IP of the load balancers created by each cloud provider. With that in mind, **we have to deploy the NATS Helm chart before we can know what the connectivity endpoint for the exposed load balancer is.**

If you look in `/aks-setup/aks-nats-values.yaml` and `/eks-setup/eks-nats-values.yaml`, you'll notice that the `gateway` block has been commented out, for this reason. You'll need to see what endpoint is generated by each cloud provider, fill in that endpoint in each `values.yaml` file, and run `helm upgrade...` on each release of NATS.

Again, in production, you likely wouldn't have to do this.

The quickest way to get this information is to run the following against the AWS context:

```
kubectl get svc nats-east -o json | jq .status.loadBalancer.ingress
```

That will print out a hostname that you can use as the endpoint for your gateway in AWS.

For Azure:

```
kubectl get svc nats-east -o json | jq .status.loadBalancer.ingress
```

That will print out the IP that you can use as the endpoint for your gateway in Azure.

If you don't have `jq` installed, you can find it here: https://jqlang.org/

Otherwise you can just run `kubectl get svc nats-east -o json` and sift through the output.

## Environment teardown

The Azure environment can be entirely torn down by `cd`ing into the `aks-setup/tofu-aks` folder and running:

```
tofu destroy --auto-approve
```

The AWS environment is a little more complex to tear down, and the Elastic Load Balancer can be a little finicky when you try to delete it. Additionally, the CloudFormation stack used to create the service account used by the Load Balancer Controller (provisioned by `eksctl`) needs to be removed as well. For that reason, a script has been created to tear down the AWS environment:

``
cd eks-setup
sh eks-teardown.sh
``

## Further Resources

- NATS Helm Chart: https://github.com/nats-io/k8s/blob/main/helm/charts/nats/README.md
- NATS By Example: https://natsbyexample.com/examples/services/intro/go
- Services Framwork documentation: https://docs.nats.io/using-nats/nex/getting-started/building-service
- Cluster documentation: https://docs.nats.io/running-a-nats-service/configuration/clustering
- SuperCluster documentation: https://docs.nats.io/running-a-nats-service/configuration/gateways
