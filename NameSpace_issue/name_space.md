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






 




