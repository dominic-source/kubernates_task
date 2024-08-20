# Issues Faced and Resolutions

## Disk Size Issue

I initially faced an issue with the disk size of my EC2 instance. Minikube required a minimum of 20 GB of free space, and the default EBS volume was insufficient. To resolve this, I expanded the EBS volume to meet the required storage capacity.

## Minikube Installation Problems

During the installation of Minikube, I encountered issues because `kubectl` was not installed on the system beforehand. I mistakenly tried to resolve the issue by deleting the Minikube image and starting again, which exacerbated the problem. After further research, I discovered that running `minikube delete` before `minikube start` resolved the issue. This sequence ensured that Minikube could start correctly without conflicts from previous attempts.

## Actions Runner Controller Installation

I encountered file size issues when trying to install the Actions Runner Controller using `kubectl`. This issue was related to the limitations of `kubectl` for handling large configurations. I resolved this problem by switching to Helm for the installation, which handled the deployment more effectively and did not present the same file size issues.
