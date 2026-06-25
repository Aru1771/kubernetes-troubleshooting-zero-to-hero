Kubernets Name-space resouce limits issue: 
------------------------------------------

i have a cluster with "100 Cpu'S" and "100 GB Ram". in that cluster i have created 5 name spaces for my 5 micro-services and i deployed them in the respective name spaces.

after some time in one of the namesapce one micro service pod have memory-leak and consumming all memoory. 
it causes in another name space another micro-service pods will down due to insufficient resources.

this is real time challenge 1:"Resource shareing" Part-1
-----------------------------------------------------------

Solution: "Use resource Quota"


Resource Quota: applying resource limits to namesapce

we can set both CPU and Ram

Q) who we can know this micro service how much ram and cpu will consume ?

A) for that we have to reachout develpoers and they will perfoem Performence bench marking and get the  one ideal number.
   based on that we will set Resource Quota to Namespace.

As of now we have set the resource Quota to Name space

Blast radius was come from cluster level to Name space end by impementing resource Quota.

but still we have the issue.

challenge 1:"Resource shareing" Part-2:
----------------------------------------

Solution: Use Resource Limits


As of now we have set the resource Quota to Name space out the side Name space there is no issue now.
Now the issue is with in the name space. still the micro service have a memory leak and consumming more Memory.

we have to add this resource limits at the Pod level.

Blast radius was come from name space level to pod level by impementing resource limits in pod.

so pod was gone to creashloop backoff state and when i describe it i have seen OOM kill 

Challenge 2: OOM Killed issues with Pod:
-----------------------------------------

Even after giveing the bench mark resource to the pod still it killed by OOM event.

i went to that Java Pod and get the thread dump and heap dump.

for getting thread dump command: kill -3
for getting heap dump command: jstack

and share these dumps to development teams.

so they will understand which part of this micro service leaking the memory.


Challenge 3: Upgrades
-----------------------


For EKS or other k8s clutser upgrades we have create one manual with a detail steps.

1. taking a backupo
2. reading a release notes of the new version add the points to manual
3. control plane-->ETCD, API, Scheduler
4. data plane--> we are using nodes as a data plane so we taint a node and drain it and make the node unschedulable and upgrade the -->KUBECTL and other objects in that node. post upgrade we will remove all taints from that node.















 




