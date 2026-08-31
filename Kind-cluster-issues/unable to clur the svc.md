* When i try to curl the ns-2 service name with FQSN from the other ns-1 pod i am unable to do.

* Error:

       unable to resolve the host

* Step: 1

* first i will check the endpoints of the service.
* second i check the core dns pods are running or not
* i found those pods are not running.
* when i describe the pod i could seen the below error

Error:

         Warning Unhealthy 95s (x447 over 71m) kubelet spec.containers{coredns}: Readiness probe failed: HTTP probe failed with statuscode: 503

* then i check the logs of the CoreDNS Pod:

  Error:



          kubectl logs coredns-559f6c778d-fnscd -n kube-system --since=10m

          [INFO] plugin/ready: Plugins not ready: "kubernetes"
But the Kubernetes plugin inside CoreDNS has not become ready:

What does this mean?

CoreDNS itself is running:

      CoreDNS process
           ↓
         Running ✅

But the Kubernetes plugin inside CoreDNS has not become ready:

      CoreDNS
         │
         └── kubernetes plugin ❌

And because the Kubernetes plugin isn't ready, the ready plugin reports:

Plugins not ready: "kubernetes"

That's why Kubernetes' readiness probe returns:

      503

and your Pod shows:

      0/1 Running


Why does CoreDNS need the Kubernetes plugin?
------------------------------------------------
Remember what we're trying to do:

      cl-sv.demo-ns.svc.cluster.local
                    ↓
                CoreDNS
                    ↓
              Kubernetes API
                    ↓
              Find Service
                    ↓
               ClusterIP

The kubernetes plugin needs to communicate with the Kubernetes API server so it can learn about:

      Services
      Pods
      Endpoints / EndpointSlices
      Namespaces

If CoreDNS can't properly communicate with the Kubernetes API, it can't build the Kubernetes DNS records.


* Then we have checked :
  1. CoreDNS ClusterRole ✅
  2. ClusterRoleBinding ✅
  3. Corefile also looks correct ✅


* We alos checked the coreDNS service is able to list the services, pods, endpoints or not with the below command:

         kubectl auth can-i list services \
      --as=system:serviceaccount:kube-system:coredns

* it's responce is Yes

* Than i found Kube-proxy one of the node is not running.
* we have resrted it but no use
* then we describe it and check the logs



  Describe:


                    Yes. This output tells us kube-proxy is repeatedly crashing, but it does not yet tell us why.

 Logs: 

                 E0831 12:01:07.724726 1 run.go:72] "command failed" err="failed complete: fsnotify watcher init: too many open files"
Your kube-proxy error is:

        failed complete: fsnotify watcher init: too many open files

This means the Linux host has reached its limit for open file descriptors / filesystem watchers.

It is not a Kubernetes RBAC problem and not an image problem.


What is happening?

Your kind node is essentially a Docker container. Inside that environment, kube-proxy tries to create an fsnotify watcher:

      kube-proxy
         ↓
      fsnotify
         ↓
      Linux file/watch resources
         ↓
      ❌ too many open files
         ↓
      kube-proxy crashes

That's why Kubernetes keeps doing:

      kube-proxy starts
            ↓
      too many open files
            ↓
      crashes
            ↓
      kubelet restarts itFix it in your Docker host

Because you're using kind, first check the limits on the machine where Docker is running.

1. Check current file limit
---------------------------
On your Linux host:

ulimit -n

Also check:

cat /proc/sys/fs/file-max

And:

cat /proc/sys/fs/inotify/max_user_instances
cat /proc/sys/fs/inotify/max_user_watches
2. Increase inotify limits

For a Linux development machine, you can increase them with:

sudo sysctl -w fs.inotify.max_user_instances=8192
sudo sysctl -w fs.inotify.max_user_watches=524288

You can verify:

sysctl fs.inotify.max_user_instances
sysctl fs.inotify.max_user_watches

You should see:

fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
3. Make the change persistent

If this fixes the problem, put these values into:

sudo vi /etc/sysctl.conf

Add:

fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=524288

Then:

sudo sysctl -p
4. Restart the failing kube-proxy

After changing the host limits:

kubectl delete pod kube-proxy-4szqz -n kube-system

Because kube-proxy is normally managed by a DaemonSet, Kubernetes will recreate it automatically.

Check:

kubectl get pods -n kube-system | grep kube-proxy

You want:

kube-proxy-xxxxx   1/1   Running
kube-proxy-yyyyy   1/1   Running
5. Check CoreDNS again

Now:

kubectl get pods -n kube-system | grep coredns

We want:

coredns-xxxxx   1/1   Running
coredns-yyyyy   1/1   Running

Then go back into your application Pod:

getent hosts cl-sv.demo-ns.svc.cluster.local

You should finally get something like:

10.x.x.x    cl-sv.demo-ns.svc.cluster.local

Then:

curl http://cl-sv.demo-ns.svc.cluster.local
🧠 The important lesson from this troubleshooting

We started with:

curl cl-sv.demo-ns.svc.cluster.local
        ↓
Could not resolve host

We then followed the chain:

Pod DNS
   ↓
CoreDNS
   ↓
CoreDNS 0/1
   ↓
Readiness 503
   ↓
kubernetes plugin not ready
   ↓
Investigate cluster
   ↓
kube-proxy 0/1 Error
   ↓
kube-proxy logs
   ↓
"too many open files"

So the actual root cause we found is:

Linux resource limit
        ↓
too many open files/watchers
        ↓
kube-proxy crashes
        ↓
Service networking is affected
        ↓
CoreDNS/DNS resolution problems

Before changing anything, run the four sysctl/ulimit checks above and send me the outputs. We can then confirm whether your host's limits are indeed exhausted and choose the appropriate fix.
            ↓
      BackOff

And this can also explain why you're seeing problems with CoreDNS readiness and Service DNS.

