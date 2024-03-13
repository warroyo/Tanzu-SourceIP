# Getting source IP using TKG, AVI, TSM and TMC

This repo is an example of how to use the above metioned stack to retrieve a source IP in an application. This will be surfaced and an `X-Forward-For` header which is commonly used by applications. 

The core products/features used here are:
* envoy filters and istio(TSM)
* proxy-protocol
* AVI load balancer
* AKO
* TKGs
* Flux Helm controller(TMC Provided)
* OPA Gatekeeper mutations(TMC provided)


## Pre-reqs

* TKGs deployed
* TKGs workload cluster deployed
* Flux helm controller installed in the cluster
* AVI installed and working with TKGs
* TSM subscription
* Cluster managed or attached by/to TMC


## Add a new AVI profile

We need to create a new AVI app profile. This is so we can enable the [proxy-protocol settings](https://avinetworks.com/docs/latest/proxy-protocol-support/) to be used by our virtual service.

1. Navigate to Template > Profiles.
2. Create a new profile named `L4-app-proxy-protocol`.
3. For Type, select L4.
4. Click Enable PROXY Protocol.
5. Select version 1.
6. Save the profile



## Install AKO

When using TKGs there is already load balancing provided by AVI/AKO. However for the functionality that we are using we need to override the supervisor provided AKO and use an in cluster AKO. The short explanation of this is that in TKGs the AKO pods runs on the supervisor cluster and layer 4 service type `LoadBalancer` is handled by this AKO instance. The challenge is that this install of AKO is not able to handle certain features like AKO CRDs. For this reaosn we need to deploy AKO in cluster using helm, by doing this we can offload the servicing of load balancers to the in cluster AKO and get access to all features. Until AKO 1.12 you will need to make sure that any loadbalancers created use the `spec.loadBalancerClass: ako.vmware.com/avi-lb` this will ensure there are no duplciate LBs created. 

For this install we are using a vsphere 8U2 environment with NSX-T and AVI. This same setup can be done if you are not using NSX-T but the values file for helm will be different. 

1. Update the values file in [helmrelease.yml](/helm-ako/helmrelease.yml)
   1. `nsxtT1LR` - this should be the T1 associated with your supervsior namespace where the cluster is provsioned. This is automatically created by TKGs.
   2. `vipNetworkList[0].networkName` - this is the ingress network. It is automatically created by TKGs.
   3. `vipNetworkList[0].cidr` - cidr for the above network. These can be found in the avi network list in the console.
   4. `ControllerSettings.serviceEngineGroupName` - auto created service engine group name in the NSX-T cloud. 
   5. `AKOSettings.clusterName` - name of the cluster.
   6. `ControllerSettings.cloudName` - name of the NSX-T cloud.
   7. `ControllerSettings.controllerHost` -  the ip of fqdn of the AVI controller.
   8. `avicredentials` -  your avi credentials.
2. In the context of the workload cluster run the following

```bash
kubectl apply -f helm-ako/helmrelease.yml
```
3. validate the ako pod is running and there are no errors.



## Install TSM

In this case we are using TSM to provide Istio into our clusters. Istio also has fucntionality that allows us to inspect the proxy-protocol TCP headers and inject them into http headers. If this is not in place the app needs to know how to do this. Luckily this can be done with an `envoyFilter` in Istio.

### Create an L4rule

This is a CRD that is provided by AKO. This is needed to be able to tell our istio ingress gateway to use the new profile we created that has proxy-protocol enabled. 

```bash
kubectl apply -f helm-ako/l4rule.yml
```

### Setup the mutations

We need to use some mutations to add some settings to the istio install provided by TSM. We are using the Gatekeeper mutation policy provdied through TMC to do this. You can find the policies [here](/mutate/istio-mutations.yml).

Here is what the mutations are doing:

* `add-lb-class-istio` - this adds the above mentioned `loadBalancerClass` setting to the istio ingress gateway. this allows our in cluster AKO to manage the LB. 
* `add-l4rule` -  this adds the annotation to the istio ingress gateway loadbalancer so that it uses our new app profile. It does this by refering the L4rule CRD created previously.
* `add-gateway-config` -  this add an istio gateway config to ingress gateway to trust multiple upstream proxies, AKA AVI.

```bash
kubectl apply -f mutate/istio-mutations.yml
```

### Onboard cluster into TSM

You can either use TMC or TSM directly. The docs can be found [here](https://docs.vmware.com/en/VMware-Tanzu-for-Kubernetes-Operations/2.3/tko-reference-architecture/GUID-deployment-guides-tko-saas-services.html#tsm) for TMC and [here](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/getting-started-guide/GUID-DE9746FD-8369-4B1E-922C-67CF4FB22D21.html) for TSM. When onboarding set the namespace inclusion to "Cluster admin owned" 

At this point you should see istio components in the cluster. You can validate all of the mutations were sucessfull by looking at the ingress gateway service object and pods. In AVI you should see that the Virtual service is using our custom profile. 

After the cluster has been onboarded create a namespace in the cluster that will be used by TSM. 

```bash
kubectl apply -f httpbin/namespace.yml
```

Finally add this namespace to a global namespace in TSM. docs on creating the GNS can be found [here](https://docs.vmware.com/en/VMware-Tanzu-Service-Mesh/services/using-tanzu-service-mesh-guide/GUID-8D483355-6F58-4AAD-9EAF-3B8E0A87B474.html).

### Create an envoy filter

In order to enable proxy-protocol in istio we need an envoy filter since TSM deploys istio `1.18.5` in `1.20` this can be done with a gateway annotation. You can see what we are doing in the istio docs [here](https://istio.io/v1.18/docs/ops/configuration/traffic-management/network-topologies/#proxy-protocol). 

```bash
kubectl apply -f envoy/proxy-proto-filter.yml
```


## Deploy a sample app

Deploying the httpbin app will allow you to see the headers passed through. You will want to update the FQDN in the `httpbin/istio.yml` to match something for your enviroment and DNS domains.

```bash
kubectl apply -f httpbin/deploy.yml
kubectl apply -f httpbin/istio.yml
```

### Test source IP

```bash
curl <your-vs-fqdn>/get?show_env=true
```

The output should have something like this, and the IP should be the client IP instead of the service engine IP.

```bash
"X-Forwarded-For": "10.16.119.109",
```