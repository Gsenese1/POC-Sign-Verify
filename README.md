# POC-Signed-Images-Verified-Sysdig-Integration
This repository will show how any OCI container image is signed and published, along with the block or warn emission and integrating to Sysdig for event UI alerts.
I have used Minikube, Docker, Cosign, Connaisseur and linux mint distribution to sign and verify OCI images, pushing alert data to Sysdig.
This POC starts with the creation of minikube to use Kubernetes clusters from my computer and demostrate the flow.
Then genereting the keys par with Cosign to sign containers.
For our example, we will retag an alpine image and push it to our registry as a first step. We will use the tag “signed” to identify the image we are going to sign.
