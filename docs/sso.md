# Authentication with Argo CD and Single Sign On (SSO)

Now that you've got Argo CD installed and everything set up, one of the first hurdles to overcome is actually accessing Argo CD. 

## Ingress Nginx and CertManager

First, we need to set up an Ingress to access Argo CD's server. To make setting up the Ingress easier, we need to install *Ingress Nginx* and *CertManager*. Ingress Nginx abstracts away much of the complexity of sending "X" URL to "Y" service, and CertManager makes it easy to serve secure URls (so we get the neat looking green lock icon in the URL tab in Chrome). 

To install these, run:

```
# Ingress Nginx
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.40.2/deploy/static/provider/aws/deploy.yaml
# CertManager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml
```

## Creating an Ingress
Now we're ready to create an Ingress that will allow us to access the Argo CD instance on your cluster from an external URL (the one you set up in the last step). Start by opening a new empty file in your favorite text editor and naming it whatever you want, as long as A) it makes enough sense that you'll remember what it is later, and B) it ends in .yaml. I recommend something like `argocd-ingress.yaml`. 

Let's write it piece by piece. First the basics. The `apiVersion` field is something all K8S resources need to have to validate that the resource was written correctly. The `kind` is self explanatory. 

```
apiVersion: extensions/v1beta1
kind: Ingress
```

This next section is a little beefier. It specifies the name and namespace (please note that it's important this Ingress reside in the `argocd` namespace because it needs access to the `argocd-server`), which are pretty obvious, but service which was installed in that namespace. but the annotations are a little more confusing:

```
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

The `cert-manager.io` annotation says "use this magic generator to make a certificate to use for this ingress that will give us that green lock icon in Chrome". The `kubernetes.io/ingress.class` tells K8S that this is an Nginx ingress, and the second `kubernetes.io` annotation helps with that green lock icon again. Finally, the last three `nginx.ingress` annotations all configure the Ingress to work properly with Argo CD.

The spec section is where most of the work happens:
```
spec:
  rules:
    - host: cluster.example.tech # REPLACE ME!!
      http:
        paths:
        - backend:
            serviceName: argocd-server
            servicePort: https
          path: /
  tls:
  - hosts:
    - cluster.example.tech # REPLACE ME!!
    secretName: argocd-secret
 ```

 The `rules` section is just what it sounds like -- it specifies rules for redirecting traffic to specific services. When we installed Argo CD, it set up a service named `argocd-server`, and here we are pointing our custom domain to it. The `http` subsection is where the service name is specified, and `argocd-server` is accessible on the HTTPS port (port 443 - this file would also work perfectly fine if you replaced "HTTPS" with "443"). The "/" in the path section just means that when we type "cluster.example.tech" into our browser, that alone will work - if we specified another value like "/foo", we'd need to type "cluster.example.tech/foo". To do this we'd also need to configure Argo CD a little differently, so best to just keep it simple. 

 The `tls` section once again is another step in getting that green lock icon to show up in Chrome, and it's important to note here that the seceretName *must* be argocd-secret because Argo CD is configured this way. 

 Once you've written all of the above into your file, run 

 ```
 k apply -n argocd argocd-ingress.yaml # or whatever your filename is
 ```

Note that the `-n argocd` bit is not necessary because we specified the namespace in our `metadata` section, but it's not a bad idea to include it when applying, just in case.

 ## Logging In

First we need to change the default Argo CD admin password for security. The default password is the name of the `argocd-server` pod, which you can get onto your clipboard by running this command: 

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2 | pbcopy
```

If you didn't do it in the last step, we need to use the Argo CD command line tool in this step, so install it with (I really hope you've installed Homebrew by now):

```
brew install argocd
```

Now run: 

```
argocd login cluster.example.tech # replace URL with the URL you set up in the Ingress
```

Now choose a new password and run: 
```
argocd account update-password
```

All done! We won't use the command line much for the rest of this tutorial - instead we'll use the UI, and we're going to set up Single Sign On so that we don't even have to use the admin account anymore.

## SSO

The first app we're going to deploy with Argo CD is an app that will allow us to sign into Argo CD more easily - our own custom Dex SSO. 

Dex is an open source OIDC provider, which means it is a sort of middleman that allows you to log into an app (like Argo CD) by using your credentials from another identity provider like Google or GitHub. 

Argo CD actually installs Dex out of the box, but we're going to install our own instance so that A) we get practice with installing and managing an app with Argo CD, and B) we can use it for other things besides logging into Argo CD if we want to later. 

The easiest way to install Dex into your cluster is with Helm. Argo CD can even manage the Dex Helm chart for us in an Application. However, the Dex Helm chart is deprecated, so we're going to use the Helm chart to generate a Kustomize app that Argo CD will manage. This gives us more control and stability. This also makes more sense in the case of the Dex chart - which generates a pretty small amount of YAML - whereas with a huge chart, it wouldn't make much sense.

When you install a Helm app, you configure it with a `values.yaml` file. Throughout this tutorial, I've used uppercase letters like REPLACE_THIS_TEXT to indicate where you should put your own information instead of the dummy info. Helm does something similar. It has a bunch of template files with the YAML definitions of K8S resources, but if you tried to apply these files, it wouldn't work -- there are many spots in the template (similar to REPLACE_THIS_TEXT) that need to be filled either by the default values or ones you specify in `values.yaml`. When you run `helm install` or `helm template`, that's what happens. 

What we need to do is specify all the right values to make our SSO work, and then have Helm generate all the necessary YAML, and save them to repository. The values file is pretty long so buckle up!

As of the time of writing, all of the Dex values that are configurable for the Helm chart are [listed on GitHub](https://github.com/helm/charts/tree/master/stable/dex).

```
dex:
  grpc: false
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-prod
      kubernetes.io/tls-acme: "true"
    path: /
    hosts:
      - auth.example.tech
    tls:
    - secretName: dex-tls
      hosts:
        - auth.example.tech
```

First, we need to disable GRPC. GRPC is an Internet protocol like HTTPS is, but we don't need it for our SSO in this tutorial so we can turn it off. 

Next, we can tell Helm that we want an Ingress and it will automatically generate one for us. You'll notice this section looks similar to the Ingress we set up above. The annotations are the same, and the `path`, `hosts`, and `tls` sections should be self explanatory after reading the earlier Ingress section.

Now it's time to configure Dex itself in the `config` section: 

```
  config:
    issuer: https://auth.example.tech
    staticClients:
    - id: argo-cd
      name: Argo CD
      redirectURIs:
      - https://cluster.example.tech/auth/callback
      secret: argocd-sso-secret
```

The `issuer` value makes sure the Dex code has knowledge of how it's seen by the outside world. 

The `staticClients` section defines clients that will use this Dex instance to authenticate. Since Argo CD will be doing this, we have to set it up here. Argo CD uses the `/auth/callback` endpoint to do its business once it receives word that someone logged in, so we must tell Dex to redirect there once it's logged in. Finally, Dex generates a secret for each static client, so specify a name for that secret in this section.

```
    frontend: 
      issuer: Example SSO
      issuerUrl: https://example.tech
```

The `frontend` section is somewhat for vanity. The `issuer` you specify here will be displayed in the UI, and the `issuerUrl` you specify should be the base domain name that you registered.

Finally, the most complex part is the `connectors` section. This is where you specify how users should be authenticated. Dex offers [a long list of connectors](https://dexidp.io/docs/connectors/), but in this tutorial I'm going to use GitHub.

There are a few things you must configure through GitHub. First, create an organization. The option to do this is in the same location where you would create a repository. Next, navigate to your new organization, and visit the "Teams" section. Here you'll find a button to create a new team. Name it whatever you'd like (I recommend "admin"), and make sure to add yourself (your own GitHub account) to the team. 

Next, navigate to your *organization's* settings. This is a different settings page from the one for your personal GitHub account. Then scroll way down and click on the menu item called "Developer Settings". There will be a section called "OAuth Apps", and we need to create one. Click the button to do so. 

The "Application Name" can be whatever you'd like, but the "Homepage URL" should be the subdomain you designated for SSO (https://auth.example.tech for example), and the "Authorization callback URL" should be the same subdomain but with "/callback" tacked on (e.g. https://auth.example.tech/callback)

Once the OAuth App has been created, you should see a "Client ID". Save this ID somewhere. Next, generate a new "Client Secret" (this should be a button on the same page). Save this secret too, and be careful with it, because you only get to see it on this page once.

Now we're ready to set up the GitHub connector: 

```
    connectors:
    - config:
        clientID: GITHUB_CLIENT_ID
        clientSecret: GITHUB_CLIENT_SECRET
        redirectURI: https://auth.example.tech/callback
        orgs:
        - name: YOUR_ORG
      id: github
      name: GitHub
      type: github
      groups: ['YOUR_ORG']
```