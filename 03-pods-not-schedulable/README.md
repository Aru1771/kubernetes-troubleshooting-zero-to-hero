In Kubernetes, the scheduler is responsible for assigning pods to nodes in the cluster based on various criteria. Sometimes, you might encounter situations where pods are not being scheduled as expected. This can happen due to factors such as node constraints, pod requirements, or cluster configurations.


By using KIND(Kubernets in Docker) we can create multinode cluter in very easy way.

step:1 setup Docker and runit
step:2 install kind from official documentation.
step3: by using create command create a clutser.

Commands:
----------

For creating single node kind cluster: kind create cluster --name CLUSTER_NAME

For creating multi node kind cluster we need to yaml file in the git: kind create cluster --name CLUSTER_NAME --config=FILE_NAME.yaml


if want to see all the K8S cluster our kubectl connected: kubectl config view

if we want to switch to another k8s cluster: kubectl config use-context CLUSTER_NAME



1. Node Selector
==================

if you have node selector in .yaml file and you try to create a deployment after creating a deployment the pods in pending state.

so when i describe the one of the pending Pod i seen an warning message like [Failedscheduling] [default scheduler] [o/4 nodes are available] [1 node had untolerated taint/4]

means we have 4 nodes these error telling none of the node is available for scheduling the Pod.

Because this Node selectore is telling schedule this Pod only in the mentioned label node.

why this labels are used in pods ?

for example the app which is running on the Pod need ARM processer at that with help of this [node selector] option we can schedule a Pod  on the specific node which have that selecter lable.


we can see the Pod selector lable in Deployment.yaml file or we can edit the Pod bu using the command: kubectl edit pod Pod_id

if no Node have that labele. simply we will edit the node by using the Command: [kubectl edit node NODE_NAME]   then add the lable under Labls: [node-name: Lable]

Node Selector is a simple way to constrain pods to nodes with specific labels. It allows you to specify a set of key-value pairs that must match the node's labels for a pod to be scheduled on that node.
Usage: Include a nodeSelector field in the pod's YAML definition to specify the required labels.

```
spec:
    containers:
    - name: my-app
    image: my-image
    nodeSelector:
    disktype: ssd
```

2. Node Affinity:
------------------
What is Node Affinity ?

before to know that first learn the diff B/W the Node selector and Node Affinity ?

with node selecter we are forcing the scheduler to schuede a pod on the only mentioned lable node if the labled node not available stay in pending.

But when comming to the Node Affinity we have two opetions.
1. preferred --> by using this it will tell scheduler try to schedule a pod on mentioned label node. if that labeled node is not available in the cluster you can schedule a pod on any of the node.

2. required --- (it will work alomost like a Node selecter)




Node Affinity is a more expressive way to specify rules about the placement of pods relative to nodes' labels. It allows you to specify rules that apply only if certain conditions are met.
Usage: Define nodeAffinity rules in the pod's YAML definition, specifying required and preferred node selectors.

```
spec:
    containers:
    - name: my-app
    image: my-image
    affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
            operator: In
            values:
            - ssd
```

3. Taints:
----------
By using this Taints node will go to unscheduled state means no pods will schedule on that Tainted Node.

in which case The Taint will use ?
At the time of cluster node upgradation we will use this Taint.
For Eg we have 3 node Cluser so if you want to update node by node because of the Prod cluster.this is one way to upgrade the K8S worker nodes.

if you tainted any of the node means the scheduler will understand this node is in unscheduled state.

in this Taints we have 3 diff options 

1. No Schedule(for cluster upgrades we will use this option)
2. No execute(if you use this all the pods in the node will stop instently)
3. preffered noschedule (by using the we are teeling the scheduler in the worest situation only schedule a Pod in this Node) this will use for performence issue Nodes.


Taints are applied to nodes to repel certain pods. They allow nodes to refuse pods unless the pods have a matching toleration.
Usage: Use kubectl taint command to apply taints to nodes. Include tolerations field in the pod's YAML definition to tolerate specific taints.

```
kubectl taint nodes node1 disktype=ssd:NoSchedule
```

```
spec:
    containers:
    - name: my-app
    image: my-image
    tolerations:
    - key: disktype
      operator: Equal
      value: ssd
      effect: NoSchedule
```

4. Tolerations:
   -------------

Tolerations means exception.

in which case we have to use this ?

in our organization we have few Pods to run at any cost because of they are Prod related pods.in that case if we set the Tolance to that Pods it will run even in the Tainted Nodes.

at the time of tainted the node we will Provide Key=value:Noseheduler.

these key=values if we use in the Pod Tolerations block.

if all the nodes are tainted but this pod will schedule in the matched key=value tainted nodes



Tolerations are applied to pods and allow them to schedule onto nodes with matching taints. They override the effect of taints.

Usage: Include tolerations field in the pod's YAML definition to specify which taints the pod tolerates.

```
spec:
  containers:
  - name: my-app
    image: my-image
  tolerations:
  - key: disktype
    operator: Equal
    value: ssd
    effect: NoSchedule
```
