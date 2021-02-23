# Single Sign On (SSO)

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

 The `rules` section is just what it sounds like -- it specifies rules for redirecting traffic to specific services. When we installed Argo CD, it set up a service named `argocd-server`, and here we are pointing our custom domain to it. The `http` subsection is where the service name is specified, and `argocd-server` is accessible on the HTTPS port (port 443 - this file would also work perfectly fine if you replaced "HTTPS" with "443"). The "/" in the path section just means that when we type "cluster.example.tech" into our browser, that alone will work - if we specified another value like "/foo", we'd need to type "cluster.example.tech/foo". To do this we'd also need to configure Argo CD a little differently so best to just keep it simple. 

 The `tls` section once again is another step in getting that green lock icon to show up in Chrome, and it's important to note here that the seceretName *must* be argocd-secret because Argo CD is configured this way. 

 Once you've written all of the above into your file, run 

 ```
 k apply -n argocd argocd-ingress.yaml # or whatever your filename is
 ```

Note that the `-n argocd` bit is not necessary because we specified the namespace in our `metadata` section, but it's not a bad idea to include it when applying, just in case.

 ## Logging In

If you didn't do it in the last step, we need to use the Argo CD command line tool in this step, so install it with (I really hope you've installed Homebrew by now):

```
brew install argocd
```

