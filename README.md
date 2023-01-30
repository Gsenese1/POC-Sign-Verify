# Proof of Concept on Container Image Signature Verification

## Project Description

This READMI file explains the main steps to run a proof of concept on container image signature and verification, it contains code snippets and links to guide you through it directly from a personal computer utilizing open-source tools.

The project demonstrates how any OCI container image can be signed and published in Kubernetes clusters blocking unsigned images in namespaces or sending warnings. Integration with Sysdig events UI will complete the process.


## Download, Installations and Configurations

Find below the list of sofware tools needed:

- **[Linux](https://www.linux.org/pages/download/)** distribution shell, I have used Mint 21.1 version.
- **[Minikube](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-intro/)**, to create a Kubernetes cluster in your computer.
- **[Cosign](https://www.sigstore.dev/)**, to sign a container and store signatures in an OCI registry generating a keypar (private and public).
- **[Connaisseur](https://sse-secure-systems.github.io/connaisseur/v2.7.0/)**, to integrate container image signature verification into a clusters.
- **[Helm](https://helm.sh/)**, a package manager to automate Kubernetes packages deployment.
- **[Sysdig](https://sysdig.com/)**, a security, monitoring and compliance solution for Containers.
- **[Busybox](https://busybox.net/about.html)**, to combine tiny versions of UNIX utilities into a single small executable.
- **[Docker Hub](https://www.docker.com/products/docker-hub/)**, to use a large container images repository.

<p align="left"> <a href="https://www.docker.com/" target="_blank" rel="noreferrer"> <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/docker/docker-original-wordmark.svg" alt="docker" width="40" height="40"/> </a> <a href="https://kubernetes.io" target="_blank" rel="noreferrer"> <img src="https://www.vectorlogo.zone/logos/kubernetes/kubernetes-icon.svg" alt="kubernetes" width="40" height="40"/> </a> <a href="https://www.linux.org/" target="_blank" rel="noreferrer"> <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/linux/linux-original.svg" alt="linux" width="40" height="40"/>  </a> </p>

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

Download and install Kubectl (the CLI which communicates with the cluster) updating the apt package index and installing packages needed to use repository:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl


```

If you use Debian 9 or earlier you would also need to install apt-transport-https:

```bash
sudo apt-get install -y apt-transport-https
```

Download the Google Cloud public signing key:

```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

```

Add the Kubernetes apt repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg]
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update now apt package index with the new repository and install kubectl:

```bash
sudo apt-get updatesudo apt-get install -y kubectl
minikube kubectl -- get po -A
```




### 2. Installing Cosign

Cosign aims to make signatures invisible infrastructure, clone repository and install it:

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


### 3. Publish Image in OCI registry

Let us pull Busybox (a  common utility troubleshooting tool) from Docker and tag it to push it in to my Docker registry:

```bash
docker pull busybox
docker tag busybox:latest gesenese1/img_verification:latest
docker push gesenese1/img_verification:latest
```

After that verify in OCI tags if the image is signed.




### 4. Signing the Image


We are going to sign the image using Cosign, although we can’t sign it without proper permissions and it is very important to have an image pushed into the registry before using Cosign. The command to sign is:

```bash
docker tag busybox:latest gesenese1/img_verification:signed

```

```bash
$(go env GOPATH)/bin/cosign sign --key cosign.key gesenese1/img_verification:signed

```


Now that the image is signed let us verify the signature using the public key cosign.pub previously generated:

```bash
cosign verify --key cosign.pub gsenese1/img_verification:signed | jq .

```


### 5. Deploy Signed Image to Kubernetes

We can deploy Cosign using the automation tool Helm chart to verify that only signed images can run into the cluster.

In this example I deploy Cosign admission webhook inside giuseppe namespace:

```bash

Kubectl create namespace giuseppe

```

Now we are going to create a Kubernetes generic secret using the cosign.pub, cosign.key and the password used to generate the signature:

```bash

kubectl create secret generic mysecret -n cosign-system \
--from-file=cosign.pub=./cosign.pub \
--from-file=cosign.key=./cosign.key \
--from-literal=cosign.password=$MY_PASSWORD

```

We add the registry and install Helm inside our cluster:

```bash
helm repo add sigstore https://sigstore.github.io/helm-charts helm repo update

helm install cosigned -n cosign-system sigstore/cosigned --devel –set cosign.secretKeyRef.name=mysecret
```

Let’s verify the installation result:

```bash
kubectl get all -n giuseppe
```

We enable the webhook in namespaces to run only images signed with Cosign using our private key:

```bash
kubectl label --overwrite namespace/giuseppe2 cosigned.sigstore.dev/include
```

### 6. Connaisseur as Admission Controller

Let us install Connaisseur:

```bash
git clone https://github.com/sse-secure-systems/connaisseur.git
cd connaisseur
helm install connaisseur helm --atomic --create-namespace --namespace connaisseur
kubectl get all -n connaisser

```

With Connaisseur, we can decide which namespaces are going to be analyzed and changed between alerting or directly blocking the deployments.
By default, Connaisseur is configured to verify the signature for the official Docker hub images and the project ones. We configure it also to verify what is signed with our generated keys.

Now we can run images signed with our previously generated Cosign keys, modifying the connaisseur deployment.
With the default configuration we are able to run official images from the Docker hub but the ones signed by our internet authority are being rejected. Here the check is applied globally so there is no need to annotate the namespace. We run them on the default namespace to showcase (trust root “default” not configured for validator “default”).

We are now going to add our own public key into Connaisseur to deploy only our signed images. In order to do this we go through the Value.yaml file and add our own Cosign key, we need to change the type to Cosign and remove the host entry.

We modified type field from Cosign, deploy the chart with new values and then it is possible to run signed images as below:
```bash
giuseppe@giuseppe-K53SV:~/cosign/connaisseur$ kubectl run giuseppe_img_signed --image=gsenese1/img_verification:signed
pod/giuseppe_img_signed created
```

To prove that it wouldn’t work with an unsigned image you can deploy one coming from Docker hub though not signed with our authority (that will result with an error):

```bash
giuseppe@giuseppe-K53SV:~/cosign/connaisseur$ kubectl run unsigned_img --image=docker.io/hello-world
```


## Integration with Sysdig

After creating a Sysdig account we can alert third-party applications, and this is done via Connaisseur notification template system, which allows integration API modifying the value.yaml file replacing Sysdig Secure token. In every request there will be an event generated automatically.

We are here going to modify Connaisseur Values.yaml file to trigger events on Sysidig.


## Conclusion

Using Cosign allows us to easily deploy a system where no external services are needed so we can set our first level of trust. Cosign, along with Connaisseur, ensures that images running in our Kubernetes clusters have been verified with Sysdig’s automated alerts.
Regarding additional automation tips on the full flow, consider that components deployed for signature and control have been already created as two Helm chart and it is not advisable to create a further wrapper.
A longer and more accurate analysis could bring to adding further open-source tools to the flow creating a pipeline and additional Helm charts.



<h3 align="right">Giuseppe Senese - Cloud Architect</h3>
<a href="https://linkedin.com/in/linkedin.com/in/giusen" target="blank"><img align="right" src="https://raw.githubusercontent.com/rahuldkjain/github-profile-readme-generator/master/src/images/icons/Social/linked-in-alt.svg" alt="linkedin.com/in/giusen" height="30" width="40" /></a>
