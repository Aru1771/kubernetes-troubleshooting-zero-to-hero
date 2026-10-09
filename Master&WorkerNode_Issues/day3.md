Day 3: Node + Kubelet + Container Runtime
------------------------------------------

1. Worker node troubleshooting

        Identify whether a node is Ready, NotReady, or under resource pressure.


2. Kubelet troubleshooting

        Understand why the node agent cannot communicate with the API server or start Pods.

3. Container runtime troubleshooting

        Investigate containerd, image pulls, and container creation failures.

Lesson 1 — Worker node troubleshooting
---------------------------------------

Step 1: Understand the architecture

A Kubernetes worker node has three important components:

    Kubelet: communicates with the API server and manages Pods on its node.
    Container runtime: runs containers; commonly containerd.
    Kube-proxy or equivalent networking implementation: helps provide Service networking, depending on the cluster setup.

Think of it this way:

    - API server = manager
    - Kubelet = worker node's agent
    - Container runtime = engine that runs containers

Step 2: Production scenario

The application team reports:

    "Our application was working earlier. Now the Pods on one worker node are failing."

You run:
    
    kubectl get nodes

Output:

    NAME        STATUS     ROLES    AGE   VERSION
    worker-1    Ready      <none>   20d   v1.32.0
    worker-2    NotReady   <none>   20d   v1.32.0
    worker-3    Ready      <none>   20d   v1.32.0
Only worker-2 is NotReady.

Step 3: What should you check first?

Run:

    kubectl describe node worker-2


Look at two sections:

    Conditions
    Ready              False
    MemoryPressure     False
    DiskPressure       False
    PIDPressure        False

Events

    NodeNotReady

    Kubelet stopped posting node status

These events suggest that the kubelet may not be reporting its status to the API server. They don't yet prove why.

Step 4: Troubleshooting flow

If you have access to the affected self-managed worker node, connect to it and check:

    sudo systemctl status kubelet


If kubelet is failing, inspect its logs:

    sudo journalctl -u kubelet -n 100 --no-pager

Then investigate the actual error before restarting services or changing configuration.

Important for EKS: On EKS managed node groups, you can investigate the worker node this way if you have host access. 
AWS manages the control plane, so you cannot SSH into its control-plane hosts to inspect kubelet or etcd there.


Investigate kubelet logs
----------------------------
You ran:

    sudo journalctl -u kubelet -n 100 --no-pager


The logs show:

    kubelet.service: Main process exited, code=exited, status=1/FAILURE

    Error:
    failed to load kubelet config file
    /etc/kubernetes/kubelet-config.yaml: no such file or directory


What does this error mean?

The kubelet service is failing because it cannot find its configuration file.

The troubleshooting path is:

    Node is NotReady
    
    Check kubelet status
    Check kubelet logs
    Configuration file is missing

What should you check next?

First, inspect the kubelet service configuration to discover which configuration file it expects:

    sudo systemctl cat kubelet


Then inspect the service's startup arguments and configured file paths. For example, check whether the referenced file exists:

    sudo ls -l /etc/kubernetes/kubelet-config.yaml


The path in this scenario is an example; actual paths vary by Kubernetes installation.

Production rule: Don't create an empty configuration file or copy one from another node blindly. 
Investigate whether the file was deleted, whether the service arguments are incorrect, or whether node provisioning failed.

Verify kubelet recovery
------------------------

We discovered that the kubelet configuration file was missing. The team restores the correct configuration using the approved recovery procedure.

Now, what should we do?

Step 1: Check kubelet status

    sudo systemctl status kubelet


Expected result:

    Active: active (running)


This tells us the service is running, but we still need to verify that the node has recovered.


Check the node from the control plane

    kubectl get nodes


Expected result:

    NAME       STATUS
    worker-1   Ready
    worker-2   Ready
    worker-3   Ready


Step 3: Verify the affected Pods

    kubectl get pods -A -o wide


Check whether the affected Pods are healthy and whether any are restarting or remaining Pending.

Remember: A running kubelet does not automatically mean the node is healthy. Confirm the node status and workload health.

"If the worker nodes use an identical kubelet configuration, 
I can restore the missing file from a healthy node after verifying that the file is shared and contains no node-specific settings. 
I would preserve the correct permissions, restart kubelet if required, and verify that the node returns to Ready. Otherwise, 
I would restore it through configuration management or  rebuild the node."

