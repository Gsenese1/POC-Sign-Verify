# Proof of Concept on OCI Image Signature

## Project Description

This READMI file explains the main steps of this POC test on container image security, it contains code snippets and links to guide you through it.

The project demonstrates how any OCI container image can be signed and published in Kubernetes clusters, along with blocking unsigned images in namespaces or sending warnings.
Integration with Sysdig events UI will complete the process.


## Download, Installations and Configurations

The list of open-source tools with links I have used for this POC directly on my personal computer are:

- **[Linux](https://www.linux.org/pages/download/)** distribution shell
- **[Minikube](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-intro/)**, to create a Kubernetes cluster in your computer
- **[Cosign](https://www.sigstore.dev/)**, to sign a container and store signatures in an OCI registry generating a keypar (private and public)
- **[Connaisseur](https://sse-secure-systems.github.io/connaisseur/v2.7.0/)**, to integrate container image signature verification into a clusters
- **[Helm](https://helm.sh/)**, a package manager to automate Kubernetes packages deployment
- **[Sysdig](https://sysdig.com/)**, a security, monitoring and compliance solution for Containers.



## Configuration Steps

### 1. Install Minikube & Interact with Clusters on Shell

To install Minikube from your Linux shell:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

Let's start your cluster:


```bash
minikube start
```

Download kubectl and use it:

```bash
minikube kubectl -- get po -A

```

### 2. Installing Cosign

Clone repository:

```bash
$ git clone https://github.com/sigstore/cosign
$ cd cosign
$ go install ./cmd/cosign
$ $(go env GOPATH)/bin/cosign

```
Generate the encrypted pair keys:

```bash
cosign generate-key-pair

```

![Key par](1_install_cosign.png)


### 3. Publish Image in OCI registry

Let's pull busybox from Docker and tag it to then push it in registry with the following commands:

```bash
docker pull busybox

```

```bash
docker tag busybox:latest gesenese1/img_verification:latest

```

```bash
docker push gesenese1/img_verification:latest

```

We should see an output like this:

![Image retag](2.png)

Check in OCI tags if the image is signed:

![Image retag](3.png)


### 3. Signing the Image


```bash
docker tag busybox:latest gesenese1/img_verification:signed

```

```bash
$(go env GOPATH)/bin/cosign sign --key cosign.key gesenese1/img_verification:signed

```
![Image retag](4.1_signing_image.png)



![Image retag](5_signed_image.png)

If we try to verify the signature:

```bash
docker push gesenese1/img_verification:signed

```


![Image retag](7_image_verification.png)


![Image retag](8_dockerhub_image_signed.png)


### 4. Deploy Signed Image to Kubernetes

Time to use automation tool Helm installing a Cosigned Admission Webhook, in order to verify that only signed images can run into the cluster:


```bash
kubectl get pod -n giuseppe

```


```bash
kubectl get all -n giuseppe

```


![Deployment](10_deployment.png)

Now let us enable the webhook in needed namespaces:


```bash
kubectl label --overwrite namespace/giuseppe2 cosigned.sigstore.dev/include=true

```

With that the image fails as is not signed as this proof:

![Deployment](12_unsigned_proof.png)



### 5. Connaisseur as Admission Controller

Let us get Connaisseur:


```bash
kubectl get all -n connaisseur

```

Now we can run images 
