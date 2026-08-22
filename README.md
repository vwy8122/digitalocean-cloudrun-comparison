# DigitalOcean vs Cloud Run: Which Should You Pick for Containers, Serverless, and APIs? Pricing, Cold Starts, and Migration Compared (With a $200 Credit Walkthrough)

If you've ever typed "digitalocean vs cloud run" into a search bar at 2 a.m. while staring at a half-finished Dockerfile, you already know the feeling. Both platforms promise to take a container, run it, scale it, and bill you something reasonable. Both have slick landing pages. Both have a free tier. And yet, picking the wrong one can mean a migration story that ends with you throwing your hands in the air at midnight.

I've spent time reading through Reddit threads, dev.to migration write-ups, official pricing pages, and a handful of independent cost breakdowns. This article pulls all of that together so you don't have to bounce between twelve tabs. The short version: the right answer depends on whether your workload sleeps, whether you need scale-to-zero, and how much you hate surprises on your monthly bill. The longer version follows.

## Why This Comparison Keeps Coming Up

Google Cloud Run and DigitalOcean App Platform sit in roughly the same spot on the cloud stack diagram — both are managed container runtimes that take a Docker image and give you a URL back. That similarity is exactly why developers keep asking which one to use.

The differences show up the moment you try to do anything real with them. Cloud Run leans into Google's enormous cloud ecosystem: IAM roles, load balancers, Cloud Build, Artifact Registry, the whole GCP machinery. DigitalOcean App Platform leans into "it just worked" — connect a GitHub repo, pick a container size, get a domain, done.

A developer named Charlie Brinicombe wrote a migration post on dev.to that captures the contrast in one sentence: "The experience using App Platform was like night and day compared to Cloud Run." His team at Trophy.so tried Cloud Run first, hit a wall trying to attach a custom domain without downtime, and switched. He's not the only one — the r/node thread from May 2026 has the same flavor, with commenters calling DO App Platform "predictable" and "closest to self hosting in pricing without self hosting."

So when does each one win? Let's get into the specifics.

## The Core Question: What Are You Actually Running?

Before pricing or features matter, you need to answer one question honestly: **does your workload sleep?**

If the answer is yes — a personal blog, an internal tool, a side project that gets a few requests a day — Cloud Run's scale-to-zero is genuinely hard to beat. One developer documented running his entire site on Cloud Run for under $10 a year because the containers spin down to nothing when there's no traffic. Try that on App Platform and you'll pay $5/month minimum even if nobody visits.

If the answer is no — an API that needs to respond in under 200ms, a production app with steady traffic, a service your customers depend on — the math flips. The same developer who runs his blog for pennies on Cloud Run calculated that an always-on 1 vCPU / 0.5 GB container on Cloud Run costs roughly $49.40/month, while the equivalent on DigitalOcean App Platform is $5/month. That's a 9x difference at the small end.

This is the single most important fork in the "digitalocean vs cloud run" decision tree. Get this one right and the rest of the comparison falls into place.

## Pricing: Where the Numbers Actually Land

Both platforms advertise a free tier and per-second billing, but the units they bill in are different enough that side-by-side comparison takes some translation. Here's what the official pricing pages show as of the most recent crawl.

**Google Cloud Run** bills in vCPU-seconds and GiB-seconds under two CPU allocation models. The cheaper "CPU only allocated during request" tier charges roughly $0.0000240 per vCPU-second and $0.0000025 per GiB-second. The "CPU always allocated" tier — which is what you need if you want predictable latency and no cold starts — runs about $0.0000180 per vCPU-second but bills for the entire wall-clock time the instance is up. The free tier gives you 180,000 vCPU-seconds and 360,000 GiB-seconds per month, which sounds generous until you realize an always-on 1 vCPU instance burns through that in about 50 hours.

**DigitalOcean App Platform** bills in flat monthly container sizes. You pick a container, you pay that price, you get a fixed amount of vCPU, RAM, and transfer. No per-second math, no surprise spikes from a viral tweet. The free tier covers three static sites with 1 GiB transfer each. Paid containers start at $5/month for 1 vCPU / 512 MiB and scale up in clear steps.

The independent breakdown by Hamy.xyz put the always-on comparison in stark terms:

| Configuration | DigitalOcean App Platform | Google Cloud Run (always-allocated, Tier 1) |
| --- | --- | --- |
| 1 vCPU, 0.5 GB RAM | $5/mo | ~$49.40/mo |
| 1 vCPU, 1 GB RAM | $12/mo | ~$52/mo |
| 1 vCPU, 2 GB RAM | $25/mo | ~$57.20/mo |
| 2 vCPU, 4 GB RAM | $50/mo | ~$114.40/mo |

The pattern is consistent: at the small, always-on end, DigitalOcean is roughly 2x to 9x cheaper. As you scale up the gap narrows but doesn't close. The catch, again, is that Cloud Run can scale to zero and App Platform can't — so if your traffic is bursty and intermittent, Cloud Run's per-request model can come out ahead.

If you want to test these numbers against your own workload, you can spin up an account with 👉 [$200 in credit for 60 days](https://bit.ly/DigitaLocean) and run the same load test on both platforms. That credit is enough to keep a $5/month container running for the full trial window with room to spare.

## Cold Starts and Predictability

This is the part of the "digitalocean vs cloud run" conversation that gets the most airtime on Reddit, and for good reason.

Cloud Run's scale-to-zero is a feature and a footgun. When traffic drops, containers spin down. When traffic returns, a new container has to spin up before it can serve the first request. That delay — the cold start — can range from a few hundred milliseconds to several seconds depending on your runtime, image size, and how recently the container ran. For a marketing site, nobody notices. For an API that a mobile app calls on every screen load, your users absolutely notice.

The standard workaround is to set a minimum instance count of 1, which keeps one container warm at all times. This eliminates cold starts but also eliminates the cost savings of scale-to-zero — you're now paying always-on prices, which is exactly the scenario where DigitalOcean wins on cost.

DigitalOcean App Platform doesn't scale to zero at all in the traditional sense. Your container runs, you pay for it, it's there. The trade-off is predictability: no cold starts, no surprise latency spikes, no 3 a.m. pages because a container took 4 seconds to wake up. The r/node commenter who called DO "predictable" was specifically talking about this. For an early-stage API project still in user acquisition, predictability matters more than theoretical cost optimization.

There's a nuance worth flagging: App Platform's autoscaling only works on dedicated instances, not shared ones. Shared instances are cheaper but can be affected by noisy neighbors, so if autoscaling is a hard requirement you're looking at the dedicated tier pricing.

## Developer Experience and Setup Friction

Pricing and cold starts are quantifiable. Developer experience is squishier, but it's where the migration stories actually live.

Charlie Brinicombe's dev.to post is the clearest case study I found. His team needed three things: seamless migration with no downtime, pricing based on resources rather than request count, and automatic deploys from GitHub. Cloud Run checked the boxes on paper but failed in practice. The custom domain setup required either Firebase Hosting, a Google Load Balancer, or a still-in-preview domain mapping feature. The load balancer route demanded he point his domain at the load balancer before Google would provision an SSL certificate — guaranteeing downtime during the switch. He found a ServerFault thread from someone else hitting the exact same wall and decided it wasn't worth it.

DigitalOcean App Platform's custom domain flow, by contrast, was a single screen where he typed the domain and the platform handled the rest. His quote: "I practically heard Hallelujah playing in my head."

That's not a technical spec you'll find on a pricing page, but it's the kind of thing that determines whether a weekend migration actually finishes on the weekend. The Reddit thread echoes it — multiple commenters describe DO's admin panel as "very intuitive" and call out the absence of surprise charges over a year of use.

Cloud Run's friction isn't universal. If you're already deep in GCP — using Artifact Registry, Cloud Build, IAM for your whole org — Cloud Run slots in naturally and the IAM complexity Charlie hit was a non-issue because the permissions were already configured. The pain shows up most for people coming from outside the GCP ecosystem who expect a Vercel-style "connect repo, get URL" experience.

## The Full DigitalOcean Plan Lineup

If the comparison is leaning you toward DigitalOcean, the next question is which product you actually need. DigitalOcean isn't a single product — it's a stack that ranges from raw VMs to managed Kubernetes to serverless functions. Here's the full current lineup pulled from the official pricing pages, with every plan listed so you can see the complete picture.

### Droplets (Cloud VMs)

The foundation. Linux virtual machines billed per second with a 60-second minimum, effective January 2026. Five families cover different workload shapes.

| Family | Smallest Plan | Largest Plan | Use Case |
| --- | --- | --- | --- |
| Basic | 512 MiB / 1 vCPU / $4/mo | 16 GiB / 8 vCPUs / $96/mo | Bursty apps, low-traffic sites |
| CPU-Optimized | 4 GiB / 2 vCPUs / $42/mo | 96 GiB / 48 vCPUs / $1,008/mo | Media streaming, gaming, analytics |
| General Purpose | 8 GiB / 2 vCPUs / $63/mo | 160 GiB / 40 vCPUs / $1,260/mo | Production workloads, balanced needs |
| Memory-Optimized | 16 GiB / 2 vCPUs / $84/mo | 256 GiB / 32 vCPUs / $1,344/mo | Databases, in-memory caches |
| Storage-Optimized | 16 GiB / 2 vCPUs / $131/mo | 256 GiB / 32 vCPUs / $2,096/mo | High I/O, large datasets |

Every Droplet includes free outbound transfer starting at 500 GiB/month, free DNS management, free cloud firewalls, and a free container registry with 1 repository and 500 MiB storage. If you want the raw-VM equivalent of what Cloud Run gives you as a managed container, this is where you'd land — you manage the OS, DigitalOcean handles the hardware.

👉 [Start with a Basic Droplet at $4/month](https://bit.ly/DigitaLocean)

### App Platform (Managed Containers)

The closest direct comparison to Cloud Run. Build and deploy from GitHub, GitLab, or a container registry. Free tier covers 3 static sites with 1 GiB transfer each. Paid tier starts at $5/month and adds horizontal/vertical scaling, autoscaling on dedicated instances, log forwarding, dedicated egress IPs, and dev/prod databases.

| Container Type | vCPU | RAM | Transfer | Price |
| --- | --- | --- | --- | --- |
| Free tier (static) | — | — | 1 GiB | $0/mo |
| Shared (Fixed) | 1 | 512 MiB | 50 GiB | $5/mo |
| Shared (Fixed) | 1 | 1 GiB | 100 GiB | $10/mo |
| Shared | 1 | 1 GiB | 150 GiB | $12/mo |
| Shared | 1 | 2 GiB | 200 GiB | $25/mo |
| Shared | 2 | 4 GiB | 250 GiB | $50/mo |

Additional charges: dedicated egress IP at $25/month per app, additional outbound transfer at $0.02/GiB, development database (512 MiB, PostgreSQL only) at $7/month. This is the product to reach for if you want Cloud Run's "give it a container, get a URL" flow without Cloud Run's IAM and load balancer complexity.

👉 [Deploy your first app on App Platform](https://bit.ly/DigitaLocean)

### Functions (Serverless)

DigitalOcean's actual serverless offering, and the closest functional equivalent to Cloud Run's scale-to-zero model. Billed in GiB-seconds with a 100ms minimum runtime per invocation. Free tier includes 90,000 GiB-seconds per month per account, pooled across all apps. Overages run $0.0000185 per GiB-second.

The free tier math: a function with 256 MiB RAM running for 100ms can be invoked 3.6 million times before you pay anything. A function with 512 MiB RAM running for 100ms gets you 1.8 million free invocations. There's no separate charge for invocations themselves — only the GiB-seconds compute.

This is the product that actually competes with Cloud Run on the scale-to-zero axis. If your workload is event-driven, bursty, or genuinely idle most of the time, Functions is where DigitalOcean matches Cloud Run's model rather than App Platform's always-on model.

👉 [Try DigitalOcean Functions with the 90,000 GiB-sec free tier](https://bit.ly/DigitaLocean)

### Managed Kubernetes (DOKS)

For workloads that have outgrown a single container. Free control plane, with high-availability control plane available for $40/month. Node pricing follows the Droplet scale — a basic 2 vCPU / 4 GiB node runs about $24/month, and you can use any Droplet size as a node. DigitalOcean advertises this against competitors charging $0.10/hr ($73/month) per cluster for the control plane alone.

If you're running multiple services, need rolling deploys, or want the operational patterns Kubernetes enables, DOKS is the path. It's also the natural upgrade from App Platform when your app outgrows a single container.

👉 [Spin up a DOKS cluster starting at $12/month](https://bit.ly/DigitaLocean)

### GPU Droplets

For AI/ML workloads. On-demand pricing starts at $0.76/GPU/hour, with 12-month reserved plans dropping NVIDIA H100 to $3.26/GPU/hour and AMD Instinct MI350X to $4.76/GPU/hour. New pricing takes effect August 1, 2026. This is the layer DigitalOcean has been investing in heavily — the homepage currently leads with AI inference messaging, citing Workato running 1T+ automation tasks at 67% lower cost and Character.ai handling 1B+ queries per day on AMD Instinct GPUs.

This is the one area where the "digitalocean vs cloud run" comparison doesn't really apply — Cloud Run doesn't offer GPU-backed containers in the same way. If your workload is LLM inference or model serving, DigitalOcean's GPU layer is a different conversation entirely.

👉 [Explore GPU Droplets from $0.76/GPU/hour](https://bit.ly/DigitaLocean)

### Managed Databases

PostgreSQL, MySQL, Redis, MongoDB, Kafka, and OpenSearch, all fully managed. Single-node clusters start at $15/month, high-availability clusters start at $30/month, read-only nodes start at $15/month. Backups are included free. Storage runs $0.215/GiB/month in 10 GiB increments.

If you're building an app on App Platform or Functions, you'll likely pair it with one of these. The development database option on App Platform ($7/month, 512 MiB, PostgreSQL only) is the cheap way to test the waters before committing to a full managed cluster.

👉 [Add a managed database starting at $15/month](https://bit.ly/DigitaLocean)

## When to Pick Which

After reading through the migration stories, the Reddit threads, and the pricing breakdowns, the decision rule that emerges is fairly clean.

**Pick Google Cloud Run if:**
- Your workload is genuinely bursty or sleeps for long stretches
- You're already invested in the GCP ecosystem (Artifact Registry, Cloud Build, IAM)
- You need scale-to-zero and can tolerate cold starts
- Your team has the patience for load balancer and IAM configuration

**Pick DigitalOcean App Platform if:**
- You have steady traffic and want predictable monthly costs
- You're coming from Vercel, Heroku, or another PaaS and want simplicity
- Custom domains and SSL need to "just work"
- You're price-sensitive at the small always-on end (the 2x-9x gap is real)

**Pick DigitalOcean Functions if:**
- You want Cloud Run's scale-to-zero economics but prefer DigitalOcean's billing simplicity
- Your workload is event-driven with long idle periods
- You want the 90,000 GiB-second free tier without per-invocation charges

**Pick DigitalOcean Droplets if:**
- You want full OS control and don't need a managed container runtime
- You're comfortable managing your own infrastructure
- You want the absolute lowest cost for a fixed workload

**Pick DOKS if:**
- You're running multiple services that need orchestration
- You've outgrown a single App Platform container
- You want Kubernetes patterns without the control-plane cost of GKE

## The $200 Credit Question

One thing that kept coming up in the research: DigitalOcean's new-user credit. Multiple sources — DevOpsCube, HostDean, SimplyCodes, and DigitalOcean's own community pages — confirm that new accounts currently get $200 in credit valid for 60 days. One community answer notes that as of mid-2026 the standard new-account credit shifted to $5 for 90 days, while referral links (like the one this article uses) still unlock the larger $200 / 60-day offer.

That credit is enough to run a $5/month App Platform container for the full trial with $100 left over, or to load-test a Functions deployment hard enough to see real cold-start behavior, or to spin up a small DOKS cluster and see how the pricing actually feels. The referral link below is the entry point for the $200 offer.

👉 [Claim the $200 credit and test both approaches yourself](https://bit.ly/DigitaLocean)

## A Note on the Bigger Picture

The "digitalocean vs cloud run" question is really two questions in a trench coat. The first is technical: which platform's model fits your workload? The second is operational: which platform's friction can your team actually absorb?

Charlie Brinicombe's migration story ended with a fun fact worth repeating. His team calculated that serving 3 billion monthly requests on DigitalOcean would cost $1-2k/month. The same volume on Vercel would cost about $130,000. The Cloud Run comparison wasn't quite that dramatic, but the direction was the same — predictable, resource-based pricing beat request-based pricing by a wide margin once traffic scaled.

That doesn't make Cloud Run wrong. For the right workload — bursty, idle-heavy, already-GCP-native — it's a clean fit and the scale-to-zero savings are real. For the wrong workload, it's a migration story that ends at midnight with a half-finished SSL cert and a ServerFault tab open.

The honest answer to "digitalocean vs cloud run" is: it depends on your traffic pattern, your existing stack, and your tolerance for configuration friction. The honest follow-up is: the only way to know for sure is to run your actual workload on both. The $200 credit makes that experiment cheap. The pricing pages above tell you what to expect once the experiment ends.
