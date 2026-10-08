Your first troubleshooting commands
-------------------------------------

1. kubectl cluster-info
2. kubectl get nodes
3. kubectl describe node <node-name>
4. kubectl get pods -A
5. kubectl describe pod <pod-name>
6. kubectl logs <pod-name>
7. kubectl logs <pod-name> --previous
8. kubectl get svc -A
9. kubectl get events -A --sort-by=.lastTimestamp


Read the below things and at last check the challenges

"I will troubleshoot from the traffic path. First I verify cluster and node health, then Pod health, Service configuration and endpoints. After that I check the Ingress/Gateway routing configuration, ALB listener and target group. If the Kubernetes and ALB layers look healthy, I investigate AWS VPC networking such as security groups, NACLs, routes and VPC Flow Logs. Finally, I validate the application-level response."


1 . if you seen Node was in Not Ready state ?

        kubectl describe node node-3

        Then investigate:
   
        Node conditions
        Events
        kubelet
        container runtime
        network
        disk
        memory

if it is kubelet related 

    systemctl status kubelet

for logs:

     journalctl -u kubelet

if it is conteiner runtime related:

    systemctl status containerd

    journalctl -u containerd

    
This is production troubleshooting.



2. kube-proxy
   
* kube-proxy participates in implementing Kubernetes Service networking on nodes.

3. CNI
   
* CNI is responsible for Kubernetes Pod networking.

  The CNI gives Pods network connectivity/IP addresses.
  On EKS, you'll commonly encounter the AWS VPC CNI.
  If CNI has problems, you may see:
  
      Pod can't get IP
      Pod-to-Pod communication failure
      DNS problems
      Service connectivity problems
  
  We'll go deep into this on the networking day.


4. Now let's understand the complete request flow


  This is one of the most important things for you to learn.
  
  Imagine your application is:https://myapp.example.com

  A simplified production architecture:

                    USER
                    |
                    v
                  DNS
                    |
                    v
              Load Balancer
                    |
                    v
                 Ingress
                    |
                    v
                Service
                    |
             +------+------+
             |             |
             v             v
           Pod 1         Pod 2
             |             |
             v             v
         Container     Container
             |             |
             v             v
          Application   Application

Very important distinction
---------------------------

# Suppose your application is returning: HTTP 500

The Kubernetes control plane might be completely healthy.

Kubernetes cluster healthy ≠ application healthy.


5. Pod in CrashLoopBackOff:

  Now your troubleshooting path is:

      Pod
     |
     +-- describe
     |
     +-- logs
     |
     +-- previous logs
     |
     +-- events
     |
     +-- container exit code
     |
     +-- resources
     |
     +-- probes
     |
     +-- configuration
     |
     +-- dependencies


 Commands:
 
     kubectl describe pod payment-7d8f9d8f-x1
    
     kubectl logs payment-7d8f9d8f-x1
    
     kubectl logs payment-7d8f9d8f-x1 --previous

 This is data-plane/workload troubleshooting, not an etcd problem by default.



6. Your production troubleshooting hierarchy

   "The application is down."
   Don't immediately start changing things.

                    APPLICATION DOWN
                       |
                       v
              Can I reach the cluster?
                       |
             +---------+---------+
             |                   |
            NO                  YES
             |                   |
        API/control          Check nodes
          plane                   |
                              Check Pods
                                  |
                              Check Service
                                  |
                              Check Ingress
                                  |
                              Check network
                                  |
                              Check storage
                                  |
                              Check application
                                  |
                              Check dependencies

This is the beginning of your production troubleshooting mindset.

Challenges:
-----------

Production incident — your first exercise

"Our application is down in production."

Pods and Nodes are running file.

1. Service check

Instead, check whether the Service has healthy endpoints.

    kubectl get svc -n production
    kubectl get endpoints -n production
    kubectl get endpointslices -n production

if you find there is no endpoints and endslices. describe the service

    kubectl describe svc payment-service -n production

2. check Ingress / Gateway_API

You would check:  

    kubectl get ingress -n production
    kubectl describe ingress <name> -n production

For Istio:

    kubectl get gateway -n production
    kubectl get virtualservice -n production

4. Load Balancer — correct

   For AWS/EKS, investigate:
 
         ALB
       |
       +-- Listener
       |
       +-- Listener rule
       |
       +-- Target group
       |
       +-- Target health
       |
       +-- Security Group
       |
       +-- Network path

User still getting 502 Error.

we could see service has endpoints how what our next set

Where is the 502 being generated?

Better troubleshooting order

    Client
      ↓
    Route 53
      ↓
    ALB / Load Balancer       ← check target health first
      ↓
    Ingress / Gateway
      ↓
    Service
      ↓
    Pod

Since the Service already has endpoints, I would check the Ingress/ALB layer next.

For example:

    kubectl get ingress -n production
    kubectl describe ingress <ingress-name> -n production
If using Istio:

    kubectl get gateway -n production
    kubectl get virtualservice -n production

Then on AWS, check:

    ALB
     ├── Listener
     ├── Listener rules
     ├── Target group
     └── Target health

Why target health is important
Suppose:

    ALB
     |
     +---- Target 1 → unhealthy
     +---- Target 2 → unhealthy
     +---- Target 3 → unhealthy

Your Pods can still show:

    1/1 Running
    1/1 Running
    1/1 Running

but the ALB can return:

    502 Bad Gateway

because the ALB cannot successfully communicate with the backend.

Where VPC Flow Logs fit

Your idea becomes very useful after we suspect a network-level problem.

For example:

    ALB
     ↓
    Target
     ↓
    ??? connection failing ???

Then we investigate:

    Security Group
    NACL
    Route Table
    VPC Flow Logs

VPC Flow Logs can help determine whether traffic is being accepted or rejected at the network interface level.

But remember: Flow Logs don't tell you the complete application-level reason for a 502

For example, an application could return HTTP 500 while the network connection itself is completely allowed.

So:

    VPC Flow Logs = network evidence
    ALB logs      = load-balancer evidence
    Ingress logs  = Kubernetes ingress evidence
    Pod logs      = application evidence

That's the production mindset I want you to develop.

# After checking at ALB end still we are seeing the issue


Test the Pod directly
Get the Pod IP:

    kubectl get pods -n production -o wide

Then test from inside the cluster:

    kubectl exec -it <debug-pod> -n production -- curl http://10.244.x.x:8080

If your application has a health endpoint:

    kubectl exec -it <debug-pod> -n production -- \
    curl http://10.244.x.x:8080/actuator/health

For a Spring Boot application, you might get:

{
  "status": "UP"
}


If this fails:
    
    Service
       ↓
    Endpoint
       ↓
    Pod
       ↓
    Application  ← problem likely here


Check application logs

    kubectl logs <pod-name> -n production

If the container restarted:

    kubectl logs <pod-name> -n production --previous

Look for things like:

    Connection refused
    Database connection failed
    OutOfMemoryError
    Timeout
    Exception
    Connection pool exhausted

Check application dependencies

For your Spring Boot example:

    Application
       |
       +---- RDS
       |
       +---- Redis
       |
       +---- External API

The Pod can be healthy while the application is unable to reach its database.

For example:

    Pod         → Running
    Service     → Healthy
    ALB         → Target Healthy
    
    BUT
    
    Spring Boot → DB connection timeout

The user may still experience:

    502 / 500 / timeout

⭐ Important distinction
You said:

"check if there is any issue with the application end"

That's correct, but in an interview don't simply say:

"I'll check the application."

Say something like:

"Since the ALB targets are healthy and the Kubernetes Service has valid endpoints, 
I'll test the application directly from inside the cluster, check the application's health endpoint and logs,
and then verify its downstream dependencies such as the database or external APIs."

That sounds much more like a production DevOps engineer.

# after checking the helth point of the application inside the cluster and it's looks good.

One more challenge for you
Suppose you execute:

    kubectl exec -it debug-pod -n production -- \
    curl http://10.244.1.15:8080/actuator/health

and get:

    HTTP/1.1 200 OK

Application is healthy.

    But users still get 502 from the ALB.

Now we have:

    Pod             ✅
    Application     ✅
    Service         ✅
    Endpoints       ✅
    ALB targets     ✅
    User            ❌

Where would you investigate next, and why?


Now we investigate the traffic path.

Your networking troubleshooting order

Don't immediately start with VPC Flow Logs. Go layer by layer:

    Internet
       |
       v
    Route 53
       |
       v
    ALB
       |
       v
    Listener / Rule
       |
       v
    Ingress / Gateway
       |
       v
    Service
       |
       v
    Pod


"Since the application, Service, endpoints, and ALB are healthy, I'll isolate the network path by checking the ALB-to-backend connectivity, Security Groups, NACLs, route tables, and finally VPC Flow Logs to determine whether traffic is being rejected."


# still facing the issue and thers is no issue with the network

* Then Check Ingress / Gateway configuration

         kubectl logs -n ingress-nginx <ingress-controller-pod>
 
If using Istio Gateway + VirtualService:

        kubectl get gateway -n production
        kubectl get virtualservice -n production
        
        kubectl describe gateway <gateway-name> -n production
        kubectl describe virtualservice <vs-name> -n production


Your troubleshooting flow is becoming:

        User → Route53 → ALB → Ingress/Gateway → Service → Endpoint → Pod → Application
