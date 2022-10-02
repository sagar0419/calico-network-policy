## Problem statement:
In this tutorial we will create a network policy in Kubernetes using calico CNI. So, only two namespaces will be able to communicate with each other. Calico is an open source networking and network security solution for containers, virtual machines, and native host-based workloads

## Prerequisite:
- The Kubernetes cluster is up and running.
- Calico is installed. If not installed, you can follow this link to find out the installation steps
- Application services are deployed in different namespaces. (For this task we have created an application in default, frontend and backend namespaces.)
Your namespace is labelled. For this demo, we are going to label the namespace with the “network=calico” label.

For this task, we created a manifest for the calico network policy which will create a rule in our Kubernetes cluster, that only frontend and backend namespaces will be able to communicate with each other. 

## Network Policy Manifest 
This manifest will create a Network Policy which will only allow traffic between the namespaces which are mentioned in this yaml file under the namespaceSelector section.

In this example, we have allowed communication between the kube-system, calico-system, backend and frontend namespace.

You can change this manifest according to your need by adding or removing different namespaces of your choice.

You can read more about GlobalNetworkPolicy and its parameter in detail here. Manifest file is added in repo with the name `networkpolicy-with-label.yaml`

```yaml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  namespaceSelector: 'network == "calico"'
  ingress:
  - action: Allow
    source:
      namespaceSelector: 'network == "calico"'
  egress:
  - action: Allow
    destination:
      namespaceSelector: 'network == "calico"'
  - action: Allow
    protocol: UDP
    destination:
      selector: 'k8s-app == "kube-dns"'
      ports:
      - 53
```
` If you don't want to use this label approach, you can simply add the namespace name in the manifest file infront of namespaceSelector. Manifest file for this is also added in the repo with the name networkpolicy-with-namespace.yaml. `  

## Testing Connectivity
For testing you can deploy your own application or you can use the below-mentioned manifest and deploy it in different namespaces. Manifest file os also added in repo name of the file is `deployment.yaml`.
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  ports:
    - targetPort: 80
      port: 80
  selector:
    app: test-one

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-one
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-one
  template:
    metadata:
      labels:
        app: test-one
    spec:
      containers:
      - name: test-one
        image: sagar27/testing:latest
        ports:
        - containerPort: 80
```

We have used 3 namespaces to deploy our application: -
- frontend
- backend
- default

## Testing:-
To test the application SSH into the frontend pod and curl the pods in other namespace . You will only be able to curl the pod running in backend namespace and vice versa.
