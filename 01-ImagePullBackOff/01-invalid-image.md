# Invalid Image

`ImagePullBackoff` can be caused if the pod runs with an invalid image or typo in the image name or even a non-existent image.

this "Error" will occurs at the time of deploying.

Imagepull backoff: we can divide this error into two parts.

Now we can see Imagepull
-------------------------

Imagepull ---> related to pulling container image to cluster. this issue occurs due to two reasons 
1. invalid image Name -->
2. no-existing image 
3. wrong image name
4. private repo image --> if you try to pull with out including secret in yaml file. for that we have to "ImagePullSecrets: -name: 'name of the secret' "

6. to create a secret please refer the doc in github.

7. to identify the root cause we have to use 'describe' command or 'events' commad [for commands refer kubectl quick reference chat]

Now we can see the backoff
---------------------------
1. k8s will not throw the "imagepull backoff" error directly  before that it will throw "ErroImage Pull" Error and wait for some time and retry to pull the image because some times due to the network issue we could see this error in this way k8s for with nearly 5 min incresing the time like first 10sec after 2sec in that way.



Watch the demo on the youtube channel for practical understanding.

