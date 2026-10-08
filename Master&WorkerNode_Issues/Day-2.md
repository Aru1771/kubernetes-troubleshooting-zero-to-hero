Production Troubleshooting on K8S Controle Plane 🔥
--------------------------------------------------

Imagine you run:

    kubectl get pods

and get:

    The connection to the server ... was refused

Don't immediately say:

    "Pod problem."

Because you cannot even communicate with Kubernetes API.

Your investigation starts with:

    kubectl cluster-info

Then:

    kubectl get nodes

If kubectl cannot communicate with the API server, you investigate the control-plane/API layer.

🎯 Day 2 Practice — Scenario 1
-------------------------------

Interviewer correction:

If you get:

    The connection to the server 10.0.1.100:6443 was refused

the first component to investigate is kube-apiserver, because:

    kubectl
       ↓
    kube-apiserver :6443
       ↓
    etcd

The error specifically says the connection to port 6443 was refused.

So first ask:

    Is kube-apiserver running and listening on 6443?

Step 1 — Check API server
-------------------------

On a self-managed control plane:

    crictl ps | grep kube-apiserver

or:

    ss -lntp | grep 6443

You can also check the kube-apiserver logs:

    crictl logs <kube-apiserver-container-id>

Step 2 — Then check etcd
-------------------------

If kube-apiserver is unhealthy, then investigate etcd:

    etcdctl endpoint health

You might see:

    https://127.0.0.1:2379 is healthy

The production thinking 🧠

Don't jump directly to etcd just because it's a control-plane component.

Use the error message to identify the layer.

    kubectl
       ↓
    6443
       ↓
    API Server       ← FIRST CHECK
       ↓
    etcd             ← SECOND CHECK
or an unhealthy/error response.

For EKS, you investigate things such as:

    kubectl cluster-info
    aws eks describe-cluster ...


🎯 Next CHECK kube-api server and port
--------------------------------------

#Suppose you check the API server and it's file but still faing the issue to connect the cluster:


Suppose you check the API server and find:

    kube-apiserver: Running
    port 6443: Listening

But:

    kubectl get nodes

still fails.


    What would you investigate next?

If:
  
    kube-apiserver → Running
    port 6443      → Listening

but kubectl still fails, then we should check the client side.

🎯 Next CHECK kube/config file
--------------------------------

1. Check current context

        kubectl config current-context

Then:

        kubectl config get-contexts

You want to verify that you're connected to the correct cluster.

2. Check kubeconfig

kubectl config view

    Look for:
    clusters:
    users:
    contexts:
    current-context:

Especially check the API server address:

    server: https://10.0.1.100:6443

3. Simple test

        kubectl cluster-info

If the context/server is wrong, you may be trying to connect to the wrong cluster.

🧠 Your troubleshooting logic now

    kubectl failing
          ↓
    Is API Server running?
          ↓ YES
    Is 6443 listening?
          ↓ YES
    Is kubeconfig correct?
          ↓
    Is current context correct?
          ↓
    Can client reach API server?
          ↓
    Then check TLS/authentication/network

we have to check the tls certificates but in this case we can avoid that because of it is not related to authentication

🎯 Next check point ETCD
--------------------------

Your investigation

First:
  
    etcdctl endpoint health

This checks whether the etcd endpoint is healthy.

Then:

    crictl ps -a | grep etcd

I recommend -a here because if etcd has exited, crictl ps may not show it.

Then check logs:
  
    crictl logs <etcd-container-id>

Your troubleshooting path is now:

    kubectl
       ↓
    kubeconfig/context       ✅
       ↓
    API server address       ✅
       ↓
    Network connectivity     ✅
       ↓
    kube-apiserver           ✅
       ↓
    TLS/certificate          ✅
       ↓
    etcd health              ← NOW CHECK
       ↓
    etcd process
       ↓
    etcd logs
       ↓
    disk / memory / certificates / corruption

🎯 Production thinking

Don't restart etcd immediately.

First find why etcd is unhealthy.

For example, logs might show:

    disk full

Then you investigate:

    df -h
    df -i

Or:

    connection timeout

Then investigate etcd performance/network.

Or:

    certificate expired

Then investigate the etcd certificates.

Next incident

You run:

    etcdctl endpoint health

and get:

    https://127.0.0.1:2379 is unhealthy:
      connection refused

Then:

crictl ps -a | grep etcd

shows:
  
    etcd   Exited

And the etcd logs show:

    etcdserver: database space exceeded 

For this error message etcdserver: database space exceeded :
------------------------------------------------------------

Production flow
First:

    df -h

This tells us which filesystem is full.
For example:

    Filesystem      Size  Used Avail Use%
    /dev/nvme0n1p1   50G   49G  1G   98%

Then identify the large directories:

    du -sh /* 2>/dev/null

Then drill down:

    du -sh /var/* 2>/dev/null

and eventually identify the large directory/file.

But for etcd, remember one thing

The error:

    etcdserver: database space exceeded

can mean etcd's internal backend quota has been reached. It is not necessarily the same thing as the Linux filesystem being 100% full.

So after identifying the issue, we should also check the etcd alarm/status and database size.

The important production distinction is:

    Linux disk full
            ≠
    etcd database quota exceeded

They can be related, but they are not exactly the same problem.

And don't immediately delete files from the etcd data directory. ⚠️


First:

    ETCDCTL_API=3 etcdctl alarm list

We are looking for something like:

    NOSPACE

Then:

    ETCDCTL_API=3 etcdctl endpoint status --write-out=table

This helps us see the etcd database size and member status.

Very important concept

Imagine your laptop has:
    
    100 GB disk
    55 GB free

But an application has a configured limit:
    
    Application database quota = 2 GB
    Current database = 2 GB

The laptop still has plenty of free space, but that application has reached its own limit.

Same idea with etcd.
    
    "First I will verify whether etcd has reached its backend quota.
    I will check the etcd NOSPACE alarm and endpoint status. 
    If the quota is reached, I will investigate old revisions and perform compaction and defragmentation according to our production procedure, 
    then verify etcd health and Kubernetes API access."

