# argocd-helm-demo #

### ArgoCD demo using https://github.com/spenfraz/gke-rmq-prometheus-grafana-perftest ###

### NOTE: replace all instances of ```spenfraz``` with your github email/username.
- ```argo-start.yaml(argo-cd ConfigMap)```
- ```ssh-keygen public/private key pair creation step```

#### NOTE: ```argo-start.yaml``` is from: https://github.com/argoproj/argo-cd/blob/master/manifests/install.yaml
	- edits: (argocd-cm ConfigMap, line 2392)
				data:
                  resource.customizations: |
                    admissionregistration.k8s.io/MutatingWebhookConfiguration:
                      ignoreDifferences: |
                        jsonPointers:
                        - /webhooks/0/failurePolicy
                    admissionregistration.k8s.io/ValidatingWebhookConfiguration:
                      ignoreDifferences: |
                        jsonPointers:
                        - /webhooks/0/failurePolicy
                    apiextensions.k8s.io/CustomResourceDefinition:
                      ignoreDifferences: |
                        jsonPointers:
                        - /metadata/annotations

                repositories: |
                  - url: git@github.com:spenfraz/gke-rmq-prometheus-grafana-perftest.git
                    sshPrivateKeySecret:
                      name: rmq-repo-secret
                      key: sshPrivateKey

#### NOTE: For simplicity use ```Google Cloud Shell```.
#### NOTE: (for port forward of argocd UI) use local ```gcloud``` install.
- ```gcloud container clusters get-credentials example-cluster --region us-central1```
- ```kubectl port-forward -n argocd svc/argocd-server 8080:443```

#### NOTE: Create a public/private key pair and copy public key to new github deploy key for target repo.
- ```ssh-keygen -t rsa -b 4096 -C "spenfraz@gmail.com"```
- https://docs.github.com/en/free-pro-team@latest/developers/overview/managing-deploy-keys#deploy-keys

1. Spin up GKE Public Cluster via Gruntwork
	- https://github.com/gruntwork-io/terraform-google-gke

2. Create Secret (for private repo access from argocd)
	- ```kubectl create ns argocd```
	- ```kubectl -n argocd create secret generic rmq-repo-secret --from-file=sshPrivateKey=/home/<user>/.ssh/id_rsa --from-file=sshPublicKey=/home/<user>/.ssh/id_rsa.pub```

3. Install ArgoCD
	- ```kubectl -n argocd apply -f argo-start.yaml```

4. Create Application(s)
    - ```kubectl create ns monitoring```
	- ```kubectl apply -f argo-app.yaml```

5. Port Forward the Argocd UI
	- ```kubectl port-forward -n argocd svc/argocd-server 8080:443```
		- username:
			- ```admin```
		- password:
			- ```kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2```
            
6. Make Commit/Pull Request to Master Branch changing ```values.yaml```...
    - from ```ArgoCD UI``` select application and select "Refresh".

7. Teardown Applications(s)
    - NOTE: deleting the Application CRDs does not delete workloads.
    - from ```ArgoCD UI``` select and delete each application (```kube-prometheus-stack``` & ```rmq-chart```)
    
8. Teardown ArgoCD
    - ```kubectl -n argocd delete -f argo-app.yaml```
    - ```kubectl -n argocd delete -f argo-start.yaml```
    - ```kubectl delete ns argocd monitoring```
    
9. Teardown GKE Public Cluster ```terraform-google-gke/```
    - ```terraform destroy```