# Getting Started

## Outcomes: 
By the end of this section, you'll know how to
- provision a simple K8S cluster
- interact with a K8S cluster
- navigate the cluster like a pro
- install Argo CD into any cluster

## Step 1: Install kubectl

`kubectl` is how you'll interact with your Kubernetes cluster - it's short for "kube control". `kubectl` is a command line program that runs on your *local machine*, and (for the sake of this tutorial) *not* on your cluster. To install it, run 

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
```

or refer to the [official kubectl docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

## Step 1A: Helpful tools

There are two tools available that will make your life with Kubernetes a lot easier: `kubectx` and `kubens`. These are also *local* command line tools. 

`kubectx` is short for "kube context" and makes switching between K8S contexts a snap. A context is the current cluster you're working on. Every `kubectl` command will run on the cluster defined by your current context, whether it's local (like Minikube) or remote. Without `kubectx`, you'd have to edit your `~/.kube/config` file every time you need to change contexts. Once you have it installed, you can simply run

```
kubectx NAME_OF_CONTEXT
```

to switch contexts. 

`kubens` is similar to `kubectx` but instead of helping you switch contexts, it helps you switch namespaces - it's short for "kube namespace". Namespaces are like walled gardens within a K8S cluster, and similar to contexts, every `kubectl` command will be run in the current namespace - it's sort of like a sub-context. Typically, apps are installed in their own namespace. You'll probably end up switching namespaces more than contexts. Once you install `kubens`, you can run

```
kubens ANY_NAMESPACE
```

to switch namespaces.

`kubens` is included as part of `kubectx`, so if you have Homebrew installed simply run `brew install kubectx` to install both. Otherwise, refer to their [more detailed docs](https://github.com/ahmetb/kubectx).

I also recommend shortening all of these commands since you'll be typing them so much. Below are the aliases I use, which you can add to `~/.zshrc` to get these shortcuts in every new terminal window: 

```
alias k=kubectl
alias kx=kubectx
alias kns=kubens
```

## Step 2: Set up a DigitalOcean cluster

As of early 2021, DigitalOcean offers $100 of free credit for their cloud platform if you're a student through the [GitHub student pack](https://education.github.com/pack). In my opinion DigitalOcean is easier to use for small projects like this than Google Cloud Platform or AWS, but if you prefer those platforms, this tutorial will still work - there just may be some extra steps I won't cover.

Since DigitalOcean is a constantly evolving platform, it's easy to use, and has good documentation, I also won't go into detail about how to create a cluster. Instead, here are the basic steps:

1. Create an account
2. Log in
3. Navigate to the Kubernetes section
4. Create a cluster with the minimum amount of resources
5. HIGHLY RECOMMENDED: use `doctl` to authenticate you local machine. DigitalOcean provides a step by step walk through of this.
5. Find the "Download Config" button or similar once the cluster is provisioned

Your config file should now be in your Downloads folder.

*While you're still logged into DigitalOcean:* 
You also need to create a LoadBalancer to access your cluster from the outside world. In the main "Create" menu, currently a green button located in the upper right hand corner, you can do so (as of now, it's toward the bottom of the options.) Once you've created it, point it to the Node (DigitalOcean calls it a Droplet) that was created as part of your K8S cluster -- if you've never used DigitalOcean before, it'll be the only option.

Once your LoadBalancer is created, make sure to copy its IP address and *SAVE IT FOR LATER*!

Now open a new Terminal window and:

```
# navigate to your Downloads
cd ~/Downloads
# copy the contents of the DigitalOcean config to your local kubeconfig file
cat NAME_OF_DIGIATALOCEAN_FILE >> ~/.kube/config
```

Now use kubectx to make sure you're in the right context. Run

```
kx # equivalent to kubectx
```
to list all available contexts. The DigitalOcean one should be pretty obvious, it usually starts with "do". Once you've found it, run

```
kx NAME_OF_DO_CONTEXT
# equivalent to kubectx NAME_OF_DO_CONTEXT
```

## Step 3: Register a domain name

Once again, if you're a student, the [GitHub student pack](https://education.github.com/pack) has you covered: Namecheap offers you a free ".me" domain for a year, or .TECH offers you a free ".tech" domain for a year. Otherwise, you might have to pay a few bucks to register a domain if you don't already have one. It doesn't matter which registrar you use as long as they let you add new A records (which is pretty much all of them). An A record is a record associated with your domain that says "send this domain or subdomain to this IP address" (you specify the IP address).

Please note that if you already have a domain, it doesn't matter if you're using it for something else - don't buy another, just use a subdomain! Even a sub-subdomain (like a.b.example.com) will work perfectly fine.

Once you've registered it, pick four subdomains to use for the rest of this tutorial. One of them will be for accessing Argo CD, so I recommend using the subdomain "cluster" or "argocd". Another will be used for configuring sign on, so you could use the subdomain "sso" or "auth". 

Here's an example: if you registered "example.tech", you could choose to use the base domain (example.tech), api.example.tech, auth.example.tech, and cluster.example.tech. Once you've decided, find out where you can edit the A records for your domain (Google is your friend) and add four new ones for each subdomain you chose. 

Remember the IP address you saved from DigitalOcean earlier? Use that IP address for each subdomain. 

## Step 4: Install Argo CD

Now back to setting up the cluster. 

First make sure you're still in the right context. Run `kx` to see what context you're in, and if you're in the wrong one, run `kx NAME_OF_THE_RIGHT_CONFIG` to switch to the right one. 

Now, create the Argo CD namespace:

```
k create ns argocd
# equivalent to:
# kubectl create namespace argocd
```

Then, install Argo CD to the namespace:
```
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# kubectl apply --namespace argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### What did this just do? 
`kubectl apply` means you're applying manifests to your cluster. 
The `-n` flag is short for `--namespace`, and ensures that these manifests are only getting applied to the correct namespace.
The `-f` flag says "apply this *F*ile", and the long URL is just a link to the Argo CD installation manifest file.

Okay, so know you just installed Argo CD to your cluster, but what does *that* mean?

Firstly, it installed a few CRDs. The most important one is the Application CRD, which allows you to define an Argo CD Application. This definition is what allows the Argo CD application controller to compare the live state of the cluster to the configuration defined in git. 

Secondly, it added a few pods to the `argocd` namespace. To see all of them, run
```
k get pods -n argocd
```

You should see several pods listed:
```
argocd-application-controller-0
argocd-dex-server-xxxxxxxxxx-xxxxx
argocd-redis-xxxxxxxxxx-xxxxx
argocd-repo-server-xxxxxxxxxx-xxxxx
argocd-server-xxxxxxxxxx-xxxxx
```

These all handle various functions that Argo CD needs to perform to make the Application CRD work.

*Is this document out of date? Not enough information, confusing? Refer to the much more robust, all around better source of info, the [official Argo CD docs](https://argo-cd.readthedocs.io/en/stable/getting_started/), for more.*
