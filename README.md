# aci-demos
Demos with ACI

Description: Run a facial recognition demo across your AKS cluster and use ACI and the ACI Connector to burst into for on-demand compute.

Contact ria.bhatia@microsoft.com if you need help!

steps to deploy dis demo & talking tips

Part 1
create resource group
```
$ az group create --name myResourceGroup --location westus2
```

deploy aks kub cluster
```
$ az aks create --resource-group myResourceGroup --name myK8sCluster --node-count 1 --generate-ssh-keys

```

get creds
```
$ az aks get-credentials --resource-group=myResourceGroup --name=myK8sCluster
```

make sure you're connected
```
$ kubectl get nodes
```

clone this repo
```
$ git clone https://github.com/prwitt/aci-demos.git
```

Make sure you have helm installed and initialize this
```
$ helm init
```
edit the values.yaml file and replace <your name> with your own hosted domain
```
$ cd aci-demos/charts/fr-demo/  
$ vim values.yaml
```

install kube-lego w/ helm for certs

```
helm install stable/kube-lego --name kube-lego --namespace kube-system --set config.LEGO_EMAIL=<your email>,config.LEGO_URL=https://acme-v01.api.letsencrypt.org/directory
```

install ingress controller w/ helm
 ```
 helm install stable/nginx-ingress --name ingress --namespace kube-system
 ```
 ^set up can be done before you demo
 Make sure to grab the Public IP of the ingress controller and place it into your dns zone registration record set.

Start at the top of the aci-demos directory and deploy the Facial Recognition application that consists of a frontend, a backend, and a set of image recognizers.

```
$ helm install charts/fr-demo --name demo
```
Checkout the UI that's generated in the output and see the pictures start to get processed
Right now you first need to reset the demo at fr-backend.<your domain>
Now checkout the UI which is usually at fr.<your domain>
The rate will be super slow because we have a 1 node AKS cluster running 2 pods.

Deploy the ACI connector :
Run this script from the `create-aci-connector` folder. The script auto creates a service principal at the resource group level and will populate a Helm chart for you.

```
$ cd create-aci-connector
$ python3 generateManifest.py --resource-group <resource group> --location <location> --subscription-id <subscription id>
```

Output

```
('Creating Resource Group ', 'myResourceGroup')
Using defalut location: westus
Creating Service Principal
Retrying role assignment creation: 1/36
Retrying role assignment creation: 2/36
Retrying role assignment creation: 3/36
Run the following command to install the ACI connector:
-----Begin Command----
helm install --name my-release --set env.azureClientId=b75b5741-b091-4e62-a7bf-000000000000,env.azureClientKey=0f7bf673-5372-4a1c-b5d9-000000000000,env.azureTenantId=72f988bf-86f1-41af-91ab-2000000000000,env.azureSubscriptionId=3762d87c-ddb8-425f-b2fc-000000000000,env.aciResourceGroup=myResourceGroup,env.aciRegion=westus ../charts/aci-connector
-----End Command----
```
Install the ACI Connector from the helm chart specified in the output from the command above.

The connector has been deployed and with a `kubectl get nodes` you can see that the ACI Connector is a new node in your cluster. Now scale up the image recognizer to 10 using the following command

```
$ kubectl scale deploy demo-fr-ir-aci --replicas 10
```
Though we are using `kubectl`, the ACI Connector is dispatching pods to Azure Container Instances transparently, via the ACI connector node.
This virtual node has unlimited capacity and a per-second billing model, making it perfect for burst compute scenarios like this one.
If we wait a minute or so for the ACI containers to warm up, we should see image recognizer throughput increase dramatically.

Check out the dashboard to see throughput it dramatically increase...

Here we can see throughput really beginning to pick up, thanks to burst capacity provided by ACI.
 
This is powerful stuff.  Here we can see AKS and ACI combine to provide the best of “serverless” computing – invisible infrastructure and micro-billing – all managed through the open Kubernetes APIs.  This kind of innovation – the marriage of containers and serverless computing -- is important for the industry, and Microsoft is working hard to make it a reality.


Once you've done all the set up you just need these commands during the live demo:
```
$ helm install charts/fr-demo --name demo
$ helm install --name aci-connector --set env.azureClientId=b75b5741-b091-4e62-a7bf-000000000000,env.azureClientKey=0f7bf673-5372-4a1c-b5d9-000000000000,env.azureTenantId=72f988bf-86f1-41af-91ab-2000000000000,env.azureSubscriptionId=3762d87c-ddb8-425f-b2fc-000000000000,env.aciResourceGroup=myResourceGroup,env.aciRegion=westus ../charts/aci-connector
$ kubectl scale deploy demo-fr-ir-aci --replicas 10
```


To clean up:
```
$ helm del purge demo
$ helm del purge aci-connector
```
