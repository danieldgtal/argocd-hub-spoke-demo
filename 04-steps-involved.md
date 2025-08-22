# run argocd api-server in insecure mode. 
### kubectl get cm -n argocd | use this command to get the configmap and edit the configmap by adding the data column as shown below.

## change argocd api server service from clusterip to nodeport
### k edit svc argocd-server -n argocd

## port forward the N