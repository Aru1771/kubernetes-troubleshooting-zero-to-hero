🚨 𝗔 𝘀𝗺𝗮𝗹𝗹 𝗰𝗼𝗻𝗳𝗶𝗴𝘂𝗿𝗮𝘁𝗶𝗼𝗻 𝗺𝗶𝘀𝘁𝗮𝗸𝗲... 𝗮𝗻𝗱 𝘀𝘂𝗱𝗱𝗲𝗻𝗹𝘆 𝗺𝘆 𝗮𝗽𝗽𝗹𝗶𝗰𝗮𝘁𝗶𝗼𝗻 𝘄𝗮𝘀 𝘂𝗻𝗿𝗲𝗮𝗰𝗵𝗮𝗯𝗹𝗲...
======================================================================================

While testing one of my applications on Kubernetes, everything looked fine at first.
 ✅ The Pods were running.
 ✅ The deployment was successful.
 ❌ But the application wasn't accessible.
The browser kept timing out, and even a simple curl request failed. My first thought was that the application had crashed.

After spending some time troubleshooting, I realized the application wasn't the problem at all—it was the Kubernetes Service. A Service exposes applications running inside Pods, and if it's 𝓶𝓲𝓼𝓬𝓸𝓷𝓯𝓲𝓰𝓾𝓻𝓮𝓭, users simply can't reach the application.

Instead of making random changes, I started checking things one by one.
--------------------------------------------------------------------------
𝟭.𝗩𝗲𝗿𝗶𝗳𝗶𝗲𝗱 𝘁𝗵𝗲 𝗦𝗲𝗿𝘃𝗶𝗰𝗲 𝗰𝗼𝗻𝗳𝗶𝗴𝘂𝗿𝗮𝘁𝗶𝗼𝗻.
𝟮.𝗖𝗼𝗺𝗽𝗮𝗿𝗲𝗱 𝘁𝗵𝗲 𝗦𝗲𝗿𝘃𝗶𝗰𝗲 𝘀𝗲𝗹𝗲𝗰𝘁𝗼𝗿 𝘄𝗶𝘁𝗵 𝘁𝗵𝗲 𝗣𝗼𝗱 𝗹𝗮𝗯𝗲𝗹𝘀.
𝟯.𝗖𝗵𝗲𝗰𝗸𝗲𝗱 𝘄𝗵𝗲𝘁𝗵𝗲𝗿 𝗲𝗻𝗱𝗽𝗼𝗶𝗻𝘁𝘀 𝘄𝗲𝗿𝗲 𝗰𝗿𝗲𝗮𝘁𝗲𝗱.
𝟰.𝗖𝗼𝗻𝗳𝗶𝗿𝗺𝗲𝗱 𝘁𝗵𝗮𝘁 𝘁𝗮𝗿𝗴𝗲𝘁𝗣𝗼𝗿𝘁 𝗺𝗮𝘁𝗰𝗵𝗲𝗱 𝘁𝗵𝗲 𝗮𝗽𝗽𝗹𝗶𝗰𝗮𝘁𝗶𝗼𝗻'𝘀 𝗹𝗶𝘀𝘁𝗲𝗻𝗶𝗻𝗴 𝗽𝗼𝗿𝘁.
𝟱.𝗧𝗲𝘀𝘁𝗲𝗱 𝗰𝗼𝗻𝗻𝗲𝗰𝘁𝗶𝘃𝗶𝘁𝘆 𝗳𝗿𝗼𝗺 𝗶𝗻𝘀𝗶𝗱𝗲 𝘁𝗵𝗲 𝗰𝗹𝘂𝘀𝘁𝗲𝗿.
𝟲.𝗥𝗲𝘃𝗶𝗲𝘄𝗲𝗱 𝗡𝗲𝘁𝘄𝗼𝗿𝗸𝗣𝗼𝗹𝗶𝗰𝗶𝗲𝘀 𝗮𝗻𝗱 𝗳𝗶𝗿𝗲𝘄𝗮𝗹𝗹 𝗿𝘂𝗹𝗲𝘀.

𝓣𝓱𝓮 𝓻𝓸𝓸𝓽 𝓬𝓪𝓾𝓼𝓮❓
👉 The Service selector didn't match the Pod labels, so Kubernetes couldn't create any endpoints for the Service. Once I fixed the labels, the endpoints appeared, and the application became accessible immediately.

𝙏𝙝𝙖𝙩 𝙞𝙨𝙨𝙪𝙚 𝙩𝙖𝙪𝙜𝙝𝙩 𝙢𝙚 𝙖𝙣 𝙞𝙢𝙥𝙤𝙧𝙩𝙖𝙣𝙩 𝙡𝙚𝙨𝙨𝙤𝙣:
--------------------------------------------------
When a Service isn't reachable, don't assume the application is broken.
Production issues like this remind me that 𝕥𝕣𝕠𝕦𝕓𝕝𝕖𝕤𝕙𝕠𝕠𝕥𝕚𝕟𝕘 𝕚𝕤𝕟'𝕥 𝕒𝕓𝕠𝕦𝕥  𝗸𝗻𝗼𝘄𝗶𝗻𝗴 𝗲𝘃𝗲𝗿𝘆 𝗰𝗼𝗺𝗺𝗮𝗻𝗱—𝕚𝕥'𝕤 𝕒𝕓𝕠𝕦𝕥 𝕗𝕠𝕝𝕝𝕠𝕨𝕚𝕟𝕘 𝕒 𝕝𝕠𝕘𝕚𝕔𝕒𝕝 𝕡𝕣𝕠𝕔𝕖𝕤𝕤 𝕒𝕟𝕕 𝕖𝕝𝕚𝕞𝕚𝕟𝕒𝕥𝕚𝕟𝕘 𝕡𝕠𝕤𝕤𝕚𝕓𝕚𝕝𝕚𝕥𝕚𝕖𝕤 𝕠𝕟𝕖 𝕓𝕪 𝕠𝕟𝕖.

Every issue I solve makes me a little more confident as a DevOps Engineer.
Have you ever spent hours debugging an application, only to discover the real problem was a simple Service configuration?
