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
kubectl create -f quarantine.yaml
```

The dns polciy opens the access from k8s hosts as well. This policy is not needed to showcase microsegmentation, and hosts usually will be added as Endpoints (HEP). So, this part can be skipped if you are replicating the steps on your environment:

```
kubectl create -f k8snodes-gns.yaml
kubectl create -f coredns.yaml
```

Before implementing our first PCI policy, let's deploy an application we can test from:

```
kubectl create -f apps.yaml
```

Exec in any of the pods deployed in the app1 namespace, and check connectivity to port 7070 of the cartservice

```
kubectl get pod -o wide | grep cartservice
```
```
APP1_POD=$(kubectl get pod -n app2 --no-headers -o name | head -1) && echo $APP1_POD
```
```
kubectl exec -ti $APP1_POD -n app2 -- sh
```

Once in the pod, try a netcat to the IP ADDRESS of the cartservice on port 7070, you should have connectivity:

```
nc -zv <IP_ADDRESS> 7070
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

#### Tip: (use the command 'kubectl label pod <pod_name> pci=true')

Just with the rule above, the connectivity between other pods will fail, you can test with the same pod in the app1 namespace as before.

Once we have isolated the PCI environment, we can create microsegmentation rules between services:

```
kubectl create -f hipsterpolicies.yaml
```

Finally, in order to implement a Zero-Trust approach, we will implement a default deny for everything else in that namespace:

```
apiVersion: crd.projectcalico.org/v1
kind: NetworkPolicy
metadata:
  name: default.hipsetrshop-deny
  namespace: default
spec:
  order: 1800
  selector: all()
  tier: default
  types:
  - Ingress
  - Egress
```

## Access Controls

### DNS policy

If you access a pod in the app2 namespace, for exanple, it should have unrestricted access to internet.

Now, let's implement some egress access control in form of a DNS policy:

```
apiVersion: projectcalico.org/v3
kind: NetworkSet
metadata:
  name: app2-trusted-domains
  namespace: app2
  labels:
    external-ep: app2-trusted-domains
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
  
You can now retest the connectivity. Only github.com should be accessible.
  
### Global Threat Feeds

You can prevent anonymization attacks, or access using Global Threat Feeds. With them, you can create Global Network Sets that can be used in Network Policies by means of native K8s constructs as labels and selectors. Below there is an example of the Feodo service for Botnet IPs:

https://feodotracker.abuse.ch/

```
apiVersion: projectcalico.org/v3
kind: GlobalThreatFeed
metadata:
  name: global.threatfeed.ipfeodo
spec:
  pull:
    http:
      url: https://feodotracker.abuse.ch/downloads/ipblocklist.txt
  globalNetworkSet:
    labels:
      feed: feodo
```

## Compliance Reports

Compliance report examples:

https://docs.tigera.io/compliance/overview#configure-and-schedule-reports

## Wireguard Encryption

https://docs.tigera.io/v3.10/compliance/encrypt-cluster-pod-traffic

## Intrusion Detection

### Anomaly Detection

To configure anomaly detection:

https://docs.tigera.io/threat/anomaly-detection/customizing

### Honeypods

To deploy Honepods:

https://docs.tigera.io/threat/honeypod/honeypods#deploy-honeypods-in-clusters

### Deep Packet Inspection

```
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: sample-dpi
  namespace: default
spec:
  selector: app == "frontend"
```


  
