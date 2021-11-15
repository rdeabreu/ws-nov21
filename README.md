# PCI compliance workshop

On preparation for the workshop, deploy a testing application (we will use Google demo app HipsterShop):

https://github.com/GoogleCloudPlatform/microservices-demo

## Microsegmentation

First let's start creating some tiers on Calico:

```
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: security
spec:
  order: 125

---

apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: platform
spec:
  order: 150
```

There will be an additional Tier which secures Tigera Components. Now, we can proceed to install some base policies:

```
kubectl create -f quarentine.yaml
```

The dns polciy opens the access from k8s hosts as well. This policy is not needed to showcase microsegmentation, and hosts usually will be added as Endpoints (HEP). So, this part can be skipped if you are replicating the steps on your environment:

```
kubectl create -f k8snodes-gns.yaml
kubectl create -f coredns.yaml
```

Now we will create a policy which provides external access to our PCI environment, and provides microsegmentation between compliant services:

```  
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.pciwhitelist
spec:
  ingress:
  - action: Allow
    destination:
      ports:
      - 8080
      selector: app == "frontend"
    protocol: TCP
    source: {}
  - action: Pass
    destination: {}
    source:
      selector: pci == "true"
  order: 101
  selector: pci == "true"
  tier: security
  types:
  - Ingress
```

We must label all components with pci=true for the policy above to be effective.

Once we have isolated the PCI environment, we can create microsegmentation rules between services:

```
kubectl create -f hipsterpolicies.yaml
```


  
  
  
