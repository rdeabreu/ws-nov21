# PCI compliance workshop

On preparation for the workshop, deploy a testing application (we will use Google demo app HipsterShop):

https://github.com/GoogleCloudPlatform/microservices-demo

## Microsgementation

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
kubectl create -f basepolicies.yaml
```

Now we will create a policy which provides external access to our PCI environment, and provides microsegmentation between compliant services:

```  
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

  
  
  
