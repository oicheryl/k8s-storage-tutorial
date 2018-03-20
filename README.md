# Kubernetes storage tutorial

This tutorial runs through
1. Installing Helm
2. Installing Rocket.Chat with MongoDB, backed by PVs
3. Deleting the pod running MongoDB
4. Waiting for Kubernetes to restart MongoDB and showing the data was persisted.

## Prerequisites

`kubectl` installed and configured to access a two node Kubernetes 1.8 cluster. Run `kubectl get nodes` to check.

## Tested

This has been tested on MacOS High Sierra 10.13.2, kubectl 1.8, Google Kubernetes Engine.

## Install Rocket.Chat 

Install helm with the correct RBAC settings:

```bash
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
curl -sSL https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | sh
helm init --service-account tiller
```

Create the config file for Rocket.Chat helm chart. Note that there are more options and we are only modifying sections related to persistent storage.

```bash
cat << EOF > rocketchat_options.yaml
mongodb:
  persistence:
    enabled: true
    storageClass: standard
    accessMode: ReadWriteOnce
    size: 8Gi
persistence:
  enabled: true
  storageClass: standard
  accessMode: ReadWriteOnce
  size: 8Gi
EOF
```

Create Rocket.Chat instance named chat1 from helm stable repo.

`helm install --name chat1 -f rocketchat_options.yaml stable/rocketchat`

List instances of installed helm packages.

`helm ls`

List kubernetes services. Services are the frontend to pods. You will note that the chat1 app has 2 services, one for the chat frontend and another for the backend mongodb database.

`kubectl get svc`

Get list of deployments.

`kubectl get deployments`

Expose the Rocket.Chat deployment to external traffic.

`kubectl expose deployment chat1-rocketchat --type=LoadBalancer --name=chat1-service`

Get list of pods. You should see 2 pods for rocketchat, one that runs the mongodb backend database and another for the rocketchat app.

`kubectl get pods -o wide`

Get a list of StorageClasses. By default, in GKE, you are given a standard storage class.

`kubectl get storageclass`

Get a list of PersistentVolumeClaims (PVCs). You will notice that helm automatically created persistent volume claims for Rocket.Chat to consume. This was set via the Rocket.Chat options file you created earlier.

`kubectl get pvc`

Get a list of PersistentVolumes (PVs). These PVs were created by the standard StorageClass once the PVCs were created.

`kubectl get pv`

## Test chat and persistence

Get the external ip address and port so you can login to your newly launched Rocket.Chat app.

`kubectl get services`

Open a browser and connect to EXTERNAL-IP:PORT.

Once in the chat room, type a few lines, then delete the pod that is running mongodb. A new pod will be scheduled in its place and come up with a minute or so (the container image needs to be fetched which slows the process down).

`kubectl delete pod $(kubectl get pods | grep mongodb | awk '{print $1}')`

The text will go grey.

Check to see pod placements (mongodb should be up and running as a new pod and probably on a different node). The text will go back to black.

Go back into Rocket.Chat app and start chatting again.

## Clean up

When done testing / exploring, delete the chat1 app via helm.

`helm delete chat1 --purge`

Remove the LoadBalancer.

`kubectl delete service/chat1-service`

Check to make sure the persistent volumes get cleaned up. You may need to check a few times. It takes about a minute for everything to clean up.

`kubectl get pv`

Remove role binding, secret and Helm.

`kubectl delete clusterrolebinding/tiller`

`kubectl delete serviceaccount tiller --namespace kube-system`

`sudo kubectl get secrets --namespace kube-system`

`sudo kubectl delete secret --namespace=kube-system tiller-token-td7lb`

`sudo rm /usr/local/bin/helm`
