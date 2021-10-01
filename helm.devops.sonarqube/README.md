# sonarqube-k8s

![SonarQube Logo](img/logo.png)

A repository used for deploying SonarQube on Azure AKS cluster.

# Usage

* [Deploy Azure Storage Class](#deploy-azure-storage-class)
* [Deploy NGINX Ingress Controller](#deploy-nginx-ingress-controller)
* [Deploy cert-manager](#deploy-cert-manager)
* [Deploy SonarQube](#deploy-sonarqube)

## Deploy Azure Storage Class

To define different tiers of storage, such as Premium and Standard, you can
create a StorageClass. The StorageClass also defines the reclaimPolicy. This
reclaimPolicy controls the behavior of the underlying Azure storage resource
when the pod is deleted and the persistent volume may no longer be required.
The underlying storage resource can be deleted, or retained for use with a
future pod.

In AKS, two initial StorageClasses are created:

* **default** - Uses Azure Standard storage to create a Managed Disk. The
  reclaim policy indicates that the underlying Azure Disk is deleted when the
  persistent volume that used it is deleted.
* **managed-premium** - Uses Azure Premium storage to create Managed Disk. The
  reclaim policy again indicates that the underlying Azure Disk is deleted when
  the persistent volume that used it is deleted.

![Storage Class Diagram](img/storage-class-diagram.png)

If no StorageClass is specified for a persistent volume, the default
StorageClass is used. Take care when requesting persistent volumes so that they
use the appropriate storage you need. You can create a StorageClass for
additional needs using kubectl.

```
$ kubectl apply --filename='./storage-class.yml'
```

Verify that the storage class was provisioned with the correct attributes. We
are looking for the **ReclaimPolicy** attribute to be set to **Retain** and the
**AllowVolumeExpansion** attribute to be set to **True**.

```
$ kubectl describe storageclass managed-premium-retain

Name:            managed-premium-retain
IsDefaultClass:  No

Provisioner:           kubernetes.io/azure-disk
Parameters:            kind=Managed,storageaccounttype=Premium_LRS
AllowVolumeExpansion:  True
MountOptions:          <none>
ReclaimPolicy:         Retain
VolumeBindingMode:     Immediate
Events:                <none>
```

[Back to top](#usage)

## Deploy NGINX Ingress Controller

In order to be able to expose services inside your cluster to the outside
world, you will need an Ingress Controller. You can use a Helm chart to
customize and automate the deployment of an Ingress Controller.

Make sure you have the stable repo added to your Helm client.

```
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
$ helm repo update
```

Create a namespace to house the resources created by the nginx-ingress Helm
chart. In this tutorial I will be using ingress controller per namespace and
hence I will deploy it to the sonarqube namespace. The ingress controller will
be using a defined scope to watch for ingress on the sonarqube namespace only.

```
$ kubectl create namespace sonarqube
```

Deploy the nginx-ingress Helm chart. You can specify the version that you
desire to deploy. By specifying a version, you are able to track what version
of the chart you deployed. It is recommended to include a version when
deploying a Helm chart.

Linux:

```
$ helm install ingress-sonarqube stable/nginx-ingress \
    --namespace sonarqube \
    --values ./nginx-ingress/values.yml \
    --version 1.36.0
```

Windows (PowerShell):

```
PS> helm install ingress-sonarqube stable/nginx-ingress `
      --namespace sonarqube `
      --values ./nginx-ingress/values.yml `
      --version 1.36.0
```

Verify that the Ingress controller was deployed successfully. Note that the
service name includes the name of the chart deployed followed by
**-nginx-ingress-controller**. Replace the service name with the name of the
chart you used to deploy the Ingress controller.

Linux:

```
$ kubectl get service ingress-sonarqube-nginx-ingress-controller \
    --namespace sonarqube \
    --output=jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

Windows (PowerShell):

```
PS> kubectl get service ingress-sonarqube-nginx-ingress-controller `
      --namespace sonarqube `
      --output=jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

The command should of returned an IP address that was used to expose the
Ingress controller. Verify that you can hit the **default backend** of the
Ingress controller.

Linux:

```
$ curl --verbose ${IP_ADDRESS}/healthz
```

Windows (PowerShell):

```
PS> Invoke-WebRequest -Uri ${IP_ADDRESS}/healthz
```

You should have received a **HTTP 200** OK response code. On Linux, make sure
to use the verbose flag with curl (--verbose).

[Back to top](#usage)

## Deploy cert-manager

cert-manager is a Kubernetes addon to automate the management and issuance of
TLS certificates from various issuing sources. It will ensure certificates are
valid and up to date periodically, and attempt to renew certificates at an
appropriate time before expiry.

Create a namespace to house the resources created by the cert-manager Helm
chart.

```
$ kubectl create namespace cert-manager
```

Install the CustomResourceDefinition resources separately.

Linux:

```
$ kubectl apply \
  --validate=false \
  --filename=https://github.com/jetstack/cert-manager/releases/download/v0.14.0/cert-manager.crds.yaml
```

Windows (PowerShell):

```
PS> kubectl apply `
      --validate=false `
      --filename=https://github.com/jetstack/cert-manager/releases/download/v0.14.0/cert-manager.crds.yaml
```

Add the jetstack Helm repository.

```
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
```

Deploy the cert-manager Helm chart and issue your first certificate. You have
the choice of issuing a certificate managed by Let's Encrypt or a selfsigned
certificate. If you choose to issue a selfsigned certificate, make sure to
distribute the Certificate Authority (CA).

Make sure to update the values in **./cert-manager/sonarqube-selfsigning.yml**
to reflect the domain name that you are using. Change the  **commonName** and
**dnsNames** to reflect your domain name.

Linux:

```
$ helm install cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --values ./cert-manager/values.yml \
    --version v0.14.1

$ kubectl apply --filename='./cert-manager/sonarqube-selfsigning.yml'
```

Windows (PowerShell):

```
PS> helm install cert-manager jetstack/cert-manager `
      --namespace cert-manager `
      --values ./cert-manager/values.yml `
      --version v0.14.1

PS> kubectl apply --filename='.\cert-manager\sonarqube-selfsigning.yml'
```

Verify that the issuer and certificate was generated in the sonarqube
namespace. Make sure you wait for the certificate to be generated.

```
$ kubectl get issuer --namespace sonarqube

NAME                 READY   AGE
selfsigning-issuer   True    9s

$ kubectl get certificate --namespace sonarqube

NAME            READY   SECRET          AGE
sonarqube-tls   True    sonarqube-tls   53s
```

Fetch the selfsigned CA certificate and import it to the Trusted Root
Certification Authorities store. Refer to 
[How to manage Trusted Root Certificates in Windows 10](https://www.thewindowsclub.com/manage-trusted-root-certificates-windows)
to import the CA certificate in Windows 10.

Linux:

```
$ kubectl get secret sonarqube-tls \
    --namespace sonarqube \
    --output=jsonpath="{.data.ca\.crt}" \
    | base64 --decode
```

Windows (PowerShell):

```
PS> $certificate = kubectl get secret sonarqube-tls `
      --namespace sonarqube `
      --output=jsonpath="{.data.ca\.crt}"

PS> [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($certificate))
```

[Back to top](#usage)

## Deploy SonarQube

Add the massimo1993 Helm repository.

```
$ helm repo add massimo1993 https://massimo1993.github.io/helm-charts
$ helm repo update
```

Apply the Pod Security Policy and RBAC role.

```
$ kubectl apply --filename='./sonarqube/sonarqube-psp.yml'
$ kubectl apply --filename='./sonarqube/sonarqube-role.yml'
```

Make sure you update the **host** value in **./sonarqube/values.yml**
to reflect the hostname that you are using for your SonarQube instance.

Deploy the sonarqube Helm chart.

Linux:

```
$ kubectl apply --filename='./sonarqube/service-account.yml'

$ helm install sonarqube massimo1993/sonarqube \
    --namespace sonarqube \
    --values ./sonarqube/values.yml \
    --version 5.3.2
```

Windows (PowerShell):

```
PS> kubectl apply --filename='.\sonarqube\service-account.yml'

PS> helm install sonarqube massimo1993/sonarqube `
      --namespace sonarqube `
      --values .\sonarqube\values.yml `
      --version 5.3.2
```

Verify that the deployment was successful, make sure that the ingress has an
IP address allocated. You need to check that the **sonarqube-sonarqube-** pod
is running.

```
$ kubectl get all --namespace sonarqube
$ kubectl get ingress --namespace sonarqube

NAME                  HOSTS                ADDRESS        PORTS     AGE
sonarqube-sonarqube   sonarqube.ddns.net   192.168.65.3   80, 443   83m
```

[Back to top](#usage)

# Acknowledgments

[cert-manager Documentation](https://cert-manager.io/docs/)

[NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)

[SonarQube Documentation](https://docs.sonarqube.org/latest/)