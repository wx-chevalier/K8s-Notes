# 基于 Cert Manager 的证书管理

cert-manager is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources, such as Let’s Encrypt, HashiCorp Vault, Venafi, a simple signing keypair, or self signed.

It will ensure certificates are valid and up to date, and attempt to renew certificates at a configured time before expiry. It is loosely based upon the work of kube-lego and has borrowed some wisdom from other similar projects e.g. kube-cert-manager.

# Deploy Cert Manager

Kubernetes 原生的证书管理控制器，可帮助从各种地方请求证书，比如 Let’s Encrypt, HashiCorp Vault, Venafi，可以请求简单的 signing keypair，或自验证证书。它会保证请求的证书有效，会在证书失效之前尝试更新。

![cert-manager-high-level-overview](https://cert-manager.io/images/high-level-overview.svg)

We need to install cert-manager to do the work with kubernetes to request a certificate and respond to the challenge to validate it. We can use helm to install cert-manager. This example installed cert-manager into the kube-system namespace from the public helm charts.

```sh
# Install the cert-manager CRDs. We must do this before installing the Helm
# chart in the next step for `release-0.12` of cert-manager:
$ kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
$ kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
$ kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

## Add the Jetstack Helm repository
$ helm repo add jetstack https://charts.jetstack.io

## Updating the repo just incase it already existed
$ helm repo update

## Install the cert-manager helm chart
$ helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.12.0 \
  jetstack/cert-manager

## Upgrade
$ helm upgrade cert-manager  --namespace cert-manager   --version v0.12.0   jetstack/cert-manager
```

Cert-manager uses two different custom resources, also known as [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)’s, to configure and control how it operates, as well as share status of its operation. These two resources are:

[Issuers](http://docs.cert-manager.io/en/latest/reference/issuers.html) (or [ClusterIssuers](http://docs.cert-manager.io/en/latest/reference/clusterissuers.html))

> An Issuer is the definition for where cert-manager will get request TLS certificates. An Issuer is specific to a single namespace in Kubernetes, and a ClusterIssuer is meant to be a cluster-wide definition for the same purpose.
>
> Note that if you’re using this document as a guide to configure cert-manager for your own Issuer, you must create the Issuers in the same namespace as your Ingress resouces by adding ‘-n my-namespace’ to your ‘kubectl create’ commands. Your other option is to replace your Issuers with ClusterIssuers. ClusterIssuer resources apply across all Ingress resources in your cluster and don’t have this namespace-matching requirement.
>
> More information on the differences between Issuers and ClusterIssuers and when you might choose to use each can be found at:
>
> https://docs.cert-manager.io/en/latest/tasks/issuers/index.html#difference-between-issuers-and-clusterissuers

[Certificate](http://docs.cert-manager.io/en/latest/reference/certificates.html)

> A certificate is the resource that cert-manager uses to expose the state of a request as well as track upcoming expirations.

# 配置 Let’s Encrypt Issuer

We will set up two issuers for Let’s Encrypt in this example. The Let’s Encrypt production issuer has [very strict rate limits](https://letsencrypt.org/docs/rate-limits/). When you are experimenting and learning, it is very easy to hit those limits, and confuse rate limiting with errors in configuration or operation.

Because of this, we will start with the Let’s Encrypt staging issuer, and once that is working switch to a production issuer.

Create this definition locally and update the email address to your own. This email required by Let’s Encrypt and used to notify you of certificate expirations and updates.

- staging issuer: [staging-issuer.yaml](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/staging-issuer.yaml)

```
   apiVersion: certmanager.k8s.io/v1alpha1
   kind: Issuer
   metadata:
     name: letsencrypt-staging
   spec:
     acme:
       # The ACME server URL
       server: https://acme-staging-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: user@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-staging
       # Enable the HTTP-01 challenge provider
       solvers:
       - http01:
           ingress:
             class:  nginx

```

Once edited, apply the custom resource:

```
$ kubectl create --edit -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/staging-issuer.yaml
issuer.certmanager.k8s.io "letsencrypt-staging" created

```

Also create a production issuer and deploy it. As with the staging issuer, you will need to update this example and add in your own email address.

- production issuer: [production-issuer.yaml](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/production-issuer.yaml)

```
   apiVersion: certmanager.k8s.io/v1alpha1
   kind: Issuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       # The ACME server URL
       server: https://acme-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: user@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-prod
       # Enable the HTTP-01 challenge provider
       solvers:
       - http01:
           ingress:
             class: nginx
$ kubectl create --edit -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/production-issuer.yaml
issuer.certmanager.k8s.io "letsencrypt-prod" created

```

Both of these issuers are configured to use the [HTTP01](http://docs.cert-manager.io/en/latest/tasks/issuers/setup-acme/http01/index.html) challenge provider.

Check on the status of the issuer after you create it:

```
 $ kubectl describe issuer letsencrypt-staging

 Name:         letsencrypt-staging
 Namespace:    default
 Labels:       <none>
 Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"certmanager.k8s.io/v1alpha1","kind":"Issuer","metadata":{"annotations":{},"name":"letsencrypt-staging","namespace":"default"},"spec":{"a...
 API Version:  certmanager.k8s.io/v1alpha1
 Kind:         Issuer
 Metadata:
   Cluster Name:
   Creation Timestamp:  2018-11-17T18:03:54Z
   Generation:          0
   Resource Version:    9092
   Self Link:           /apis/certmanager.k8s.io/v1alpha1/namespaces/default/issuers/letsencrypt-staging
   UID:                 25b7ae77-ea93-11e8-82f8-42010a8a00b5
 Spec:
   Acme:
     Email:  your.email@your-domain.com
     Private Key Secret Ref:
       Key:
       Name:  letsencrypt-staging
     Server:  https://acme-staging-v02.api.letsencrypt.org/directory
     Solvers:
       Http 01:
         Ingress:
           Class:  nginx
 Status:
   Acme:
     Uri:  https://acme-staging-v02.api.letsencrypt.org/acme/acct/7374163
   Conditions:
     Last Transition Time:  2018-11-17T18:04:00Z
     Message:               The ACME account was registered with the ACME server
     Reason:                ACMEAccountRegistered
     Status:                True
     Type:                  Ready
 Events:                    <none>
```

You should see the issuer listed with a registered account.

# Deploy a TLS Ingress Resource

With all the pre-requisite configuration in place, we can now do the pieces to request the TLS certificate. There are two primary ways to do this: using annotations on the ingress with [ingress-shim](http://docs.cert-manager.io/en/latest/tasks/issuing-certificates/ingress-shim.html) or directly creating a certificate resource.

In this example, we will add annotations to the ingress, and take advantage of ingress-shim to have it create the certificate resource on our behalf. After creating a certificate, the cert-manager will update or create a ingress resource and use that to validate the domain. Once verified and issued, cert-manager will create or update the secret defined in the certificate.

Note

The secret that is used in the ingress should match the secret defined in the certificate. There isn’t any explicit checking, so a typo will resut in the nginx-ingress-controller falling back to its self-signed certificate. In our example, we are using annotations on the ingress (and ingress-shim) which will create the correct secrets on your behalf.

Edit the ingress add the annotations that were commented out in our earlier example:

- ingress tls: [ingress-tls.yaml](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/ingress-tls.yaml)

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/issuer: "letsencrypt-staging"

spec:
  tls:
  - hosts:
    - example.example.com
    secretName: quickstart-example-tls
  rules:
  - host: example.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
```

and apply it:

```
$ kubectl create --edit -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/ingress-tls.yaml
ingress.extensions "kuard" configured
```

Cert-manager will read these annotations and use them to create a certificate, which you can request and see:

```
$ kubectl get certificate
NAME                     AGE
quickstart-example-tls   38s
```

Cert-manager reflects the state of the process for every request in the certificate object. You can view this information using the kubectl describe command:

```
 $ kubectl describe certificate quickstart-example-tls

 Name:         quickstart-example-tls
 Namespace:    default
 Labels:       <none>
 Annotations:  <none>
 API Version:  certmanager.k8s.io/v1alpha1
 Kind:         Certificate
 Metadata:
   Cluster Name:
   Creation Timestamp:  2018-11-17T17:58:37Z
   Generation:          0
   Owner References:
     API Version:           extensions/v1beta1
     Block Owner Deletion:  true
     Controller:            true
     Kind:                  Ingress
     Name:                  kuard
     UID:                   a3e9f935-ea87-11e8-82f8-42010a8a00b5
   Resource Version:        9295
   Self Link:               /apis/certmanager.k8s.io/v1alpha1/namespaces/default/certificates/quickstart-example-tls
   UID:                     68d43400-ea92-11e8-82f8-42010a8a00b5
 Spec:
   Dns Names:
     example.your-domain.com
   Issuer Ref:
     Kind:       Issuer
     Name:       letsencrypt-staging
   Secret Name:  quickstart-example-tls
 Status:
   Acme:
     Order:
       URL:  https://acme-staging-v02.api.letsencrypt.org/acme/order/7374163/13665676
   Conditions:
     Last Transition Time:  2018-11-17T18:05:57Z
     Message:               Certificate issued successfully
     Reason:                CertIssued
     Status:                True
     Type:                  Ready
 Events:
   Type     Reason          Age                From          Message
   ----     ------          ----               ----          -------
   Normal   CreateOrder     9m                 cert-manager  Created new ACME order, attempting validation...
   Normal   DomainVerified  8m                 cert-manager  Domain "example.your-domain.com" verified with "http-01" validation
   Normal   IssueCert       8m                 cert-manager  Issuing certificate...
   Normal   CertObtained    7m                 cert-manager  Obtained certificate from ACME server
   Normal   CertIssued      7m                 cert-manager  Certificate issued Successfully
```

The events associated with this resource and listed at the bottom of the describe results show the state of the request. In the above example the certificate was validated and issued within a couple of minutes.

Once complete, cert-manager will have created a secret with the details of the certificate based on the secret used in the ingress resource. You can use the describe command as well to see some details:

```
$ kubectl describe secret quickstart-example-tls

Name:         quickstart-example-tls
Namespace:    default
Labels:       certmanager.k8s.io/certificate-name=quickstart-example-tls
Annotations:  certmanager.k8s.io/alt-names=example.your-domain.com
              certmanager.k8s.io/common-name=example.your-domain.com
              certmanager.k8s.io/issuer-kind=Issuer
              certmanager.k8s.io/issuer-name=letsencrypt-staging

Type:  kubernetes.io/tls

Data
====
tls.crt:  3566 bytes
tls.key:  1675 bytes
```

Now that we have confidence that everything is configured correctly, you can update the annotations in the ingress to specify the production issuer:

- ingress tls final: [ingress-tls-final.yaml](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/ingress-tls-final.yaml)

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/issuer: "letsencrypt-prod"

spec:
  tls:
  - hosts:
    - example.example.com
    secretName: quickstart-example-tls
  rules:
  - host: example.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kuard
          servicePort: 80
$ kubectl create --edit -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/docs/tutorials/acme/quick-start/example/ingress-tls-final.yaml

ingress.extensions "kuard" configured
```

You will also need to delete the existing secret, which cert-manager is watching and will cause it to reprocess the request with the updated issuer.

```
$ kubectl delete secret quickstart-example-tls

secret "quickstart-example-tls" deleted
```

This will start the process to get a new certificate, and using describe you can see the status. Once the production certificate has been updated, you should see the example KUARD running at your domain with a signed TLS certificate.

```
 $ kubectl describe certificate

 Name:         quickstart-example-tls
 Namespace:    default
 Labels:       <none>
 Annotations:  <none>
 API Version:  certmanager.k8s.io/v1alpha1
 Kind:         Certificate
 Metadata:
   Cluster Name:
   Creation Timestamp:  2018-11-17T18:36:48Z
   Generation:          0
   Owner References:
     API Version:           extensions/v1beta1
     Block Owner Deletion:  true
     Controller:            true
     Kind:                  Ingress
     Name:                  kuard
     UID:                   a3e9f935-ea87-11e8-82f8-42010a8a00b5
   Resource Version:        283686
   Self Link:               /apis/certmanager.k8s.io/v1alpha1/namespaces/default/certificates/quickstart-example-tls
   UID:                     bdd93b32-ea97-11e8-82f8-42010a8a00b5
 Spec:
   Dns Names:
     example.your-domain.com
   Issuer Ref:
     Kind:       Issuer
     Name:       letsencrypt-prod
   Secret Name:  quickstart-example-tls
 Status:
   Conditions:
     Last Transition Time:  2019-01-09T13:52:05Z
     Message:               Certificate does not exist
     Reason:                NotFound
     Status:                False
     Type:                  Ready
 Events:
   Type    Reason        Age   From          Message
kubectl describe certificate quickstart-example-tls   ----    ------        ----  ----          -------
   Normal  Generated     18s   cert-manager  Generated new private key
   Normal  OrderCreated  18s   cert-manager  Created Order resource "quickstart-example-tls-889745041"
```

You can see the current state of the ACME Order by running `kubectl describe` on the Order resource that cert-manager has created for your Certificate:

```
$ kubectl describe order quickstart-example-tls-889745041
...
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     90s   cert-manager  Created Challenge resource "quickstart-example-tls-889745041-0" for domain "example.your-domain.com"

```

Here, we can see that cert-manager has created 1 ‘Challenge’ resource to fulfil the Order. You can dig into the state of the current ACME challenge by running `kubectl describe` on the automatically created Challenge resource:

```
$ kubectl describe challenge quickstart-example-tls-889745041-0
...

Status:
  Presented:   true
  Processing:  true
  Reason:      Waiting for http-01 challenge propagation
  State:       pending
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Started    15s   cert-manager  Challenge scheduled for processing
  Normal  Presented  14s   cert-manager  Presented challenge using http-01 challenge mechanism

```

From above, we can see that the challenge has been ‘presented’ and cert-manager is waiting for the challenge record to propagate to the ingress controller. You should keep an eye out for new events on the challenge resource, as a ‘success’ event should be printed after a minute or so (depending on how fast your ingress controller is at updating rules):

```
$ kubectl describe challenge quickstart-example-tls-889745041-0
...

Status:
  Presented:   false
  Processing:  false
  Reason:      Successfully authorized domain
  State:       valid
Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  Started         71s   cert-manager  Challenge scheduled for processing
  Normal  Presented       70s   cert-manager  Presented challenge using http-01 challenge mechanism
  Normal  DomainVerified  2s    cert-manager  Domain "example.your-domain.com" verified with "http-01" validation
```

Note

If your challenges are not becoming ‘valid’ and remain in the ‘pending’ state (or enter into a ‘failed’ state), it is likely there is some kind of configuration error. Read the [Challenge resource reference docs](http://docs.cert-manager.io/en/latest/reference/challenges.html) for more information on debugging failing challenges.

Once the challenge(s) have been completed, their corresponding challenge resources will be _deleted_, and the ‘Order’ will be updated to reflect the new state of the Order:

```
$ kubectl describe order quickstart-example-tls-889745041
...
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     90s   cert-manager  Created Challenge resource "quickstart-example-tls-889745041-0" for domain "example.your-domain.com"
  Normal  OrderValid  16s   cert-manager  Order completed successfully

```

Finally, the ‘Certificate’ resource will be updated to reflect the state of the issuance process. If all is well, you should be able to ‘describe’ the Certificate and see something like the below:

```
$ kubectl describe certificate quickstart-example-tls

Status:
  Conditions:
    Last Transition Time:  2019-01-09T13:57:52Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-09T12:57:50Z
Events:
  Type    Reason         Age                  From          Message
  ----    ------         ----                 ----          -------
  Normal  Generated      11m                  cert-manager  Generated new private key
  Normal  OrderCreated   11m                  cert-manager  Created Order resource "quickstart-example-tls-889745041"
  Normal  OrderComplete  10m                  cert-manager  Order "quickstart-example-tls-889745041" completed successfully
```

Using openssl

```
$ echo | openssl s_client -connect "letsencrypt.org":443 -servername "letsencrypt.org" -verify_hostname "letsencrypt.org" 2>/dev/null | openssl x509 -noout -startdate -enddate
notBefore=Sep 29 16:33:36 2019 GMT
notAfter=Dec 28 16:33:36 2019 GMT
```

Using Sensu/Nagios checks

- [https://github.com/sensu-plugins/sensu-plugins-ssl](https://github.com/sensu-plugins/sensu-plugins-ssl)
- https://github.com/sensu-plugins/sensu-plugins-http

Using Prometheus and Alertmanager

- [https://www.robustperception.io/get-alerted-before-your-ssl-certificates-expire](https://www.robustperception.io/get-alerted-before-your-ssl-certificates-expire)

Free services

- Let’s Encrypt provides certificate expiration emails if you have configured an email address. [https://letsencrypt.org/docs/expiration-emails/ 2](https://letsencrypt.org/docs/expiration-emails/)
- [https://certificatemonitor.org/](https://certificatemonitor.org/)
