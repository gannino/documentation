# Walk-through of the Skyscrapers' Kubernetes cluster

## Requirements

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [helm](https://github.com/kubernetes/helm)

## Authentication

Once the cluster is setup:

1. head to the url `https://kubesignin.<cluster-domain-name>/login`
1. select the GitHub login for your company
1. authorize the Kubesignin application in GitHub
1. at this point you should have been redirected to a webpage containing a token (and some metadata). This token will be valid for one hour and will allow you to authenticate to the cluster. Ensure that there's an `email` claim in the metadata, otherwise you'll need to set a public email address in GitHub.
1. run `kubectl config set-credentials <username> --token=<token>` to set the token in kubectl config.
1. run `kubectl config set-cluster <cluster-name> --server https://api.<cluster-domain-name> --certificate-authority=<certificate_file>` to set the cluster info in kubectl config. You can find the certificate data in the documentation in your repo.
1. run `kubectl config set-context <anything-you-want> --cluster <cluster-name> --user <username>` to set the context in kubectl config.
1. run `kubectl config use-context <anything-you-want>` to select the context created above. Contexts are useful when you want to manage multiple clusters.

Now you should be able to interact with the cluster. Try `kubectl get pods` for example (*Note that this command might return successfully but not display anything. This means that you don't have Pods deployed in the `default` namespace*)

## Deploying applications & services on Kubernetes: the Helm Package Manager

After a roll out of a Kubernetes cluster, it could be tempting to start executing numerous `kubectl create` commands to get stuff deployed on the cluster.

Running such commands is a good idea to learn how deployments are done on Kubernetes, but it **is not the appropriate tool** to construct fully self contained application deployments. The Kubernetes community came up with a separate tool for that:

[The Helm Package Manager](https://helm.sh/)

With Helm, you create self contained packages for a specific piece of a deployment, e.g. a web stack. Such packages are called `Charts` in Helm terminology.

You probably want such a stack to be deployed in a specific Kubernetes namespace, with a specific configuration (`ConfigMap`), defining a Kubernetes `Service` referring to a `Deployment`. But if you want to have this setup reproducible, you need a way to parameterize this.
By using a Template engine, a [Go function library](http://masterminds.github.io/sprig/) and the use of a `Values.yaml` file, you can build a template of a specific piece and re-use that for multiple deployments.

The [Helm documentation](https://docs.helm.sh/) is quite good, but this section highlights 2 specific features that should be used from the start to create good reusable Charts:

* [Dependencies](https://docs.helm.sh/chart-best-practices/#requirements)
* [Lifecycle Hooks](https://docs.helm.sh/developing-charts/#chart-lifecycle-hooks)

Good examples always help out a lot. Here is a list of existing git Charts repositories:

* [Kubernetes Charts](https://github.com/helm/charts/)
* [Skyscrapers Charts](https://github.com/skyscrapers/charts)

The above Chart repositories contain Charts that serve as building blocks for bigger composite installations.

## Ingress

### http traffic (ports 80 and 443)

At the moment, http (ports 80 and 443) ingress to the cluster is done as follows:

```
public ELB -> Nginx -> pods
```

To make your deployment accessible from the outside world through http(s), you need to create an `Ingress` object with the following annotation: `kubernetes.io/ingress.class: "nginx"`. This will tell the Nginx ingress controller to route traffic to your services. You can find more information on how the Nginx ingress controller works and some examples in the official documentation: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

### Other traffic

If your application needs to be accessible through other ports or through TCP, you'll need to create your own ingresses. Normally you'll want to create a `Service` of type `LoadBalancer`. With that, Kubernetes will automatically create an ELB that will route traffic to your pods on the needed ports. An example of this kind of `Service` can be found here: https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer

## DNS

To automatically setup the DNS records in Route53 for you applications, you'll need to annotate the `Ingress` or `Service` objects with the following annotation:

```yaml
external-dns.alpha.kubernetes.io/hostname: nginx.example.org.
```

The only requirement for this to work is that the DNS zone for the domain is hosted in Route53 in the same AWS account as the cluster.

## IAM Roles

Your deployments can be assigned with specific IAM roles to grant them fine-grained permissions to AWS services. To do that just add the following annotation to `.spec.template.metadata.annotations` in your `Deployment` objects:

```yaml
iam.amazonaws.com/role: arn:aws:iam::123456789:role/kube2iam/something
```

You can find some examples in the `kube2iam` official documentation: https://github.com/jtblin/kube2iam#kubernetes-annotation

**Note**: when you create a new IAM role for a Kubernetes deployment, make sure you create it under the `/kube2iam/` path, otherwise your deployments won't be able to assume it. Also you'll need to add a [trust relationship](http://docs.aws.amazon.com/directoryservice/latest/admin-guide/edit_trust.html) to your role so the k8s nodes roles can assume it. Normally the k8s nodes role will have the following name scheme: `arn:aws:iam::<aws-account-id>:role/nodes.<cluster-domain-name>`

## Automatic SSL certificates

As with the DNS records, SSL certificates can also be automatically fetched and setup for applications deployed on the Kubernetes cluster. To do that you'll need to add the following annotation to the `Ingress` object:

```yaml
kubernetes.io/tls-acme: "true"
```

And also you'll need to add a `tls` section in the spec of the `Ingress` object, like the following:

```yaml
tls:
- secretName: yourapplicationname-lego-tls
  hosts:
  - example.application.com
```

With the `hosts` array of the `tls` section you're telling the cluster (`kube-lego` is the service that handles that) which domains need to be in the certificate.

## Cronjobs

Kubernetes can run cronjobs for you. More information/examples about cronjobs can be found [here](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

### Clean up

Starting from Kubernetes 1.7 the scheduled jobs don't get automatically cleaned up. So make sure that you add the following two lines to the `spec` section of your cronjob.

```yaml
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
```

This will clean up all jobs except the last 3, both for successful and failed jobs.
