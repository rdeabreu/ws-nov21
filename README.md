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

## Access Controls

In order to test egress controls with dns policies we will deploy some multitool apps:

```
kubectl create -f apps.yaml
```

If you access a pod in the app2 namespace, for exanple, it should have unrestricted access to internet.

Now, let's implement some egress access control in form of a DNS policy:

```
apiVersion: projectcalico.org/v3
kind: NetworkSet
metadata:
  name: app1-trusted-domains
  namespace: app1
  labels:
    external-ep: app1-trusted-domains
spec:
  allowedEgressDomains:
    - 'github.com'

---

apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: default.egress-from-app2
  namespace: app2
spec:
  egress:
  - action: Allow
    destination:
      namespaceSelector: all()
      ports:
      - 53
      selector: k8s-app == "kube-dns"
    protocol: UDP
    source: {}
  - action: Allow
    destination:
      selector: external-ep == "app2-trusted-domains"
    source: {}
  order: 400
  selector: app == "app2"
  tier: default
  types:
  - Egress
  
  ```
  
  
  
  
  
