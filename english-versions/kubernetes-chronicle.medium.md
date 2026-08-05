# A Chronicle of Kubernetes: From Borg to the Cloud-Native Operating System

## The key features and turning points that shaped Kubernetes over more than a decade—told as a story even non-experts can follow

<!-- Medium: upload kubernetes-chronicle-cover.png as the Featured image. Prefer pasting from html/kubernetes-chronicle.medium.html for formatting. -->

> If a data center is a harbor and containers are standardized shipping boxes, then `Kubernetes` is the harbor’s scheduling system: which berth a ship gets, how cargo is loaded and unloaded, what happens when something breaks, and how capacity expands when traffic spikes. It does not manufacture the boxes. It decides whether those boxes can run at scale.

---

## Opening: Why Ordinary People Keep Bumping Into `Kubernetes`

You may never have written a line of `YAML`, yet you are probably already using `Kubernetes`.

Open an app on your phone, scroll through short videos, or ask ChatGPT a question—chances are high that the services behind them run on a `Kubernetes` cluster. It has become something like the default operating system of the cloud age: it does not write business logic, but it places thousands of services onto machines, keeps them alive, and scales them with traffic.

The name comes from Greek and means “helmsman” or “pilot.” The community often shortens it to **K8s** (the eight letters between K and s). The logo is a heptagon—that is not an accident, and we will come back to it.

This chronicle will not copy every release changelog. I want to make three things clear:

1. What problem `Kubernetes` actually solves
2. Which milestones and capabilities truly changed the industry along the way
3. Why it, rather than Swarm, Mesos, or someone else, became the default answer

Even if you have never touched `kubectl`, you should still come away with a working picture.

---

## Prequel: Google’s Internal Borg (from about 2003)

The story starts at Google.

Long before public cloud and Docker made containers mainstream, Google was already using **containers** to carve a physical machine into many isolated rooms, packing Search, Gmail, YouTube, and other services together at very high utilization. The system that managed those “rooms” was called **Borg**.

Borg was not a product for outsiders. It was Google’s internal cluster butler, answering very plain questions:

- Which machine should this task land on?
- If a machine dies, how do we move the work automatically?
- When traffic spikes, how do we open more replicas?
- When jobs of different priority fight for resources, who yields?

Google later built **Omega**, exploring more flexible scheduling and a shared-state model. The Borg / Omega paper in ACM Queue, *Borg, Omega, and Kubernetes*, became nearly required reading in container orchestration.

For non-specialists, one sentence is enough:

> `Kubernetes` was not invented from a blank sheet. It is Google retelling more than a decade of lessons about running containers at scale—this time in the open.

It inherited many of Borg’s instincts. The clearest example is treating a group of tightly cooperating containers as one scheduling unit (an `alloc` in Borg, a `Pod` in `Kubernetes`). At the same time, it deliberately improved on Borg’s limits: Borg mainly grouped work with a relatively rigid `Job`, while `Kubernetes` organizes objects with flexible `Label`s and selectors, and puts more weight on a declarative “desired state”—you describe what you want, and the system works to make it so.

On the control plane, it also differs from later Omega: instead of opening the shared store directly to trusted components, it exposes only a REST API with validation and policy.

---

## 2013–2014: Docker Lights the Fuse, and Scheduling Becomes Everyone’s Problem

In 2013, Docker made containers usable for almost anyone. Before that, containers felt like ops black magic. After Docker, developers could package an app on a laptop and move it to a server nearly unchanged.

Once the shipping boxes were standardized, the next question appeared immediately:

**A few containers on one machine are easy. Who manages thousands of containers across hundreds or thousands of machines?**

So “orchestration” became the new battlefield: automated deploy, scaling, service discovery, failure recovery. Whoever made this into a general platform had a shot at becoming cloud infrastructure’s standard.

In the summer of 2013, Google’s Joe Beda, Brendan Burns, and Craig McLuckie pitched leadership on an idea: build an open-source container management system that brought Borg / Omega experience to the outside world. The internal codename was **Project Seven of Nine**—a nod to *Star Trek*’s Seven of Nine. That is also why the `Kubernetes` logo has seven sides.

On June 6, 2014, `Kubernetes` was open-sourced and the first commit landed. The industry usually treats that day as its birthday.

It was imperfect then—even rough. But the direction was right: developer experience first, a declarative API, extensibility, and open source from day one.

---

## 2015: 1.0 Ships, and the Helm Goes to the CNCF

A little more than a year later, on July 21, 2015, `Kubernetes` **1.0** was released. Google also announced that it was donating the project to the newly formed **CNCF** (Cloud Native Computing Foundation), hosted by the Linux Foundation.

The symbolism mattered.

If `Kubernetes` had remained “Google’s open-source project,” competitors would have hesitated. Once it sat under a neutral foundation, Red Hat, IBM, Intel, VMware—and later AWS and Microsoft—were far more willing to bet on it. The CNCF’s goal was never only to grow `Kubernetes`; it was to advance a whole “cloud-native” stack: container packaging, dynamic scheduling, and microservice-oriented design.

About a month later, on August 26, 2015, Google’s managed offering **Google Container Engine** (later GKE) also reached general availability. Cloud vendors began selling “we will run the `Kubernetes` control plane for you” as a product.

By 1.0, the core story was already in place:

- **`Pod`** — The smallest scheduling unit: a small room that can hold one or more tightly cooperating containers
- **`Node`** — A worker machine (physical or virtual)
- **`Service`** — A stable access point in front of a changing set of `Pod`s
- **`Label` / `Selector`** — Tag objects, then filter by those tags—like sorting codes on parcels
- **Declarative API** — You say “keep 3 replicas alive,” and the system makes it so, instead of you `SSH`ing in by hand

Many capabilities people later take for granted—`Deployment`, `DaemonSet`, `StatefulSet`, `Ingress`, mature `RBAC`—grew after 1.0. Version 1.0 was more like a usable minimum loop: proof that orchestration could be a platform.

---

## 2015–2017: The Orchestration Wars, and Why `Kubernetes` Won

For a while, the market had at least three main contenders:

- **Docker Swarm**: tightest Docker integration, easiest to start
- **Apache Mesos + Marathon**: older, strong at very large and heterogeneous workloads
- **`Kubernetes`**: steep learning curve, but a complete and extensible model

Nomad and others were in the mix too. The press liked to call it the “orchestration war.”

`Kubernetes` won not because any single switch was magical, but because several forces stacked:

1. **Google’s production credibility + CNCF’s neutral governance**, which lowered the political risk of betting on it
2. **A well-designed API and extension points**, on which later `CRD`s, `Operator`s, and interfaces were built
3. **Ecosystem reinforcement**: monitoring, networking, storage, CI/CD, and security tools prioritized `Kubernetes`
4. **Cloud vendors lining up**: once AWS, Azure, and GCP all offered managed `Kubernetes`, enterprise choice was nearly locked in

The climax came at DockerCon Europe in October 2017: Docker announced native `Kubernetes` support alongside Swarm. Coverage was nearly unanimous—the orchestration war had a winner.

Swarm did not vanish overnight, and Mesos still lives in some niches. But the industry already knew what the default answer was.

---

## 2016–2018: From “It Can Run” to “It Can Run for Real”

If 1.0 was learning to walk, the next few years were building muscle. Several capabilities that changed daily use largely arrived in this period.

### `Deployment`: Releases Stop Feeling Like Defusing a Bomb

Early on, people mostly poked at `Pod`s or `ReplicationController`s. Later, **`Deployment`** became the standard posture for stateless apps: rolling updates, rollbacks, declared replica counts.

Think of it this way:

> Stop deciding by hand which container to kill first and which to start next. Tell the system what the new version looks like, and ask it to replace smoothly.

`DaemonSet` (one copy per node—good for logging/monitoring agents), `StatefulSet` (stateful apps with stable identity and storage), `ReplicaSet`, and other core workload APIs all graduated to stable together in `Kubernetes` **1.9** (early 2018) as `apps/v1`.

That meant common application shapes finally had first-class, officially stable APIs.

### `RBAC`: Who Can Do What to the Cluster

Once a cluster hits production, permissions cannot mean “everyone can `kubectl` as they please.”

**`RBAC`** (Role-Based Access Control) became generally available in **1.8** (2017). It answers whether an account can read a `Secret`, delete a `Namespace`, or change a node.

For non-security people, one sentence is enough:

> `RBAC` turned `Kubernetes` from “a farm of servers sharing the root password” into “a system with a gate.”

### Networking and Policy: Beyond `Service`, Learn to Say “No”

By default, `Kubernetes` is quite “friendly”—`Pod`s can often talk to each other freely. That is great for demos and dangerous in production.

**`NetworkPolicy`** lets you write rules about who may talk to whom. Together with `CNI` plugins (Calico, Cilium, Flannel, and others), cluster networking moved from “it connects” to “it is controllable.” `NetworkPolicy` under `networking.k8s.io/v1` became stable in **1.7** (2017); **1.8** added more useful `egress` policies. The old `extensions/v1beta1` version stopped being served in 1.16.

### Package Management and Application Delivery

An API alone is not enough; people still have to install complex apps. Tools like **Helm** spread quickly in this period, packaging sets of `YAML` into installable, upgradable Charts. `Kubernetes` turns resources into reality; Helm delivers the whole application.

---

## 2016–2019: The Real Killer Feature—Extensibility

If the orchestration war was about “who can schedule containers,” `Kubernetes` won the second half with another sentence:

> **Turn the platform into something that can grow new platforms.**

### `CRI` / `CNI` / `CSI`: Do Not Bind to Any One Implementation

`Kubernetes` realized early that it should not be both referee and every athlete.

- **`CRI`** (Container Runtime Interface): how containers are created and deleted belongs to the runtime. Introduced as Alpha with **1.5** at the end of 2016, it first paved the “pluggable runtime” path; over the following years, `containerd`, `CRI-O`, and others matured their `CRI` implementations.
- **`CNI`** (Container Network Interface): how container networking is wired belongs to network plugins.
- **`CSI`** (Container Storage Interface): how persistent storage is attached belongs to storage plugins. `CSI` reached GA in **1.13** (early 2019).

The strategic value of these three interfaces outweighs almost any single feature:

> Networking vendors, storage vendors, and runtime projects can join the race without changing the `Kubernetes` kernel. A larger ecosystem stabilizes the standard; a stable standard grows the ecosystem.

### `CRD`: Teach `Kubernetes` New Nouns

A **`CustomResourceDefinition` (`CRD`)** lets you register new API types in the cluster—`PostgresCluster`, `Certificate`, `Prometheus`, and so on. After that, `kubectl get` can manage them just like native `Pod`s.

`CRD`s grew out of `ThirdPartyResources`; after a redesign they entered beta in **1.7** and reached GA in **1.16** (2019) as `apiextensions.k8s.io/v1`.

For newcomers, this is almost the key to modern `Kubernetes`:

> Many “advanced capabilities” you see in a cluster are not magic suddenly appearing in the kernel. Someone used a `CRD` to teach the cluster a new object, then wrote a controller to implement it.

### `Operator`: Turn Operations Knowledge Into Software

In November 2016, CoreOS introduced the **`Operator`** pattern: encode expert know-how about deploying, backing up, failing over, and upgrading a complex system into a controller that watches custom resources. `CRD`s were not yet stable then; early implementations leaned more on mechanisms such as `ThirdPartyResource`. Once `CRD`s matured, `Operator`s almost all moved onto that path.

Early open-source examples included the `etcd Operator` and `Prometheus Operator`. The pattern then exploded—databases, message queues, certificates, and machine-learning platforms could all “live” in the cluster as `Operator`s.

In one metaphor:

> A `Deployment` keeps “three identical web replicas” alive; an `Operator` keeps “this stateful system alive the way an expert would.”

---

## 2018: The Big Three Clouds Align, and `Kubernetes` Becomes Cloud’s Common Language

- **GKE**: generally available since 2015—first out of the gate
- **Amazon EKS**: generally available in June 2018
- **Azure AKS**: also generally available in June 2018

By then, AWS, Azure, and GCP all offered managed `Kubernetes`. For enterprises, that meant:

> Learn one model and you can speak on multiple clouds. Differences fall more on surrounding services, networking, and identity—not on whether you must learn a third orchestrator.

`Kubernetes` went from an open-source project to a **portability layer** for cloud computing—not eliminating lock-in entirely, but sharply reducing lock-in at the “how does the app run” layer.

A bit earlier, in November 2017, the CNCF launched **Certified Kubernetes Conformance**: any distribution calling itself `Kubernetes` had to pass a test suite guaranteeing core API behavior. That helped avoid severe Unix- or Android-style fragmentation, and made it easier in 2018 for enterprises to treat the major clouds’ managed offerings as different accents of the same language.

---

## 2020–2022: Letting Go, and Coming of Age

Any system that lives a decade must learn to throw away old tickets.

### Saying Goodbye to `dockershim`

For a long time, many people thought “`Kubernetes` = running containers with Docker.” In reality, `Kubernetes` depends on a **container runtime interface**; Docker Engine was only the most popular early implementation.

To stay compatible with Docker, `kubelet` once shipped a built-in shim called **`dockershim`**. It was formally deprecated in **1.20** and removed in **1.24** (April 2022).

The community recommended moving to `CRI`-compatible runtimes such as `containerd` and `CRI-O`. End users could still build images with Docker; the change was mainly about which runtime actually runs containers on cluster nodes.

It was a psychological coming of age:

> `Kubernetes` told the world clearly that the container ecosystem’s standards are `OCI` / `CRI`—not one company’s toolchain.

### Reworking the Security Model: From `PSP` to Pod Security Admission

**`PodSecurityPolicy` (`PSP`)** once limited `Pod` privileges as a built-in mechanism, but it was hard to use and hard to evolve. It was deprecated in **1.21** and removed in **1.25** (2022).

It was replaced by the simpler **Pod Security Admission**: label namespaces and apply one of three standards—`Privileged` / `Baseline` / `Restricted`. For finer needs, hand off to external policy engines such as `OPA` / Gatekeeper or `Kyverno`.

The direction is clear: safer by default, less magic in the mechanism.

### The Control Plane Grows “Adult” Features Too

These years also brought capabilities that rarely make marketing slides but matter enormously at scale: API Priority and Fairness (keeping an admin path open under overload), finer scheduling, topology awareness, and more. They do not sell posters, but they decide whether you can sleep when the cluster has thousands of nodes.

---

## 2023–2026: Ingress Evolves, and AI Pushes New Cargo Into the Old Harbor

### `Gateway API`: The Next Generation of Traffic Ingress

For years, exposing HTTP/HTTPS services mostly relied on **`Ingress`**. It is simple, but extensions fragmented—annotations everywhere, behavior varying by implementation.

**`Gateway API`** released **v1.0** in October 2023, graduating core resources such as `Gateway`, `GatewayClass`, and `HTTPRoute` to stable. It separates “how infrastructure provides an entry point” from “how applications declare routes,” and better fits multi-team, multi-implementation coexistence.

`Ingress` will not vanish overnight, but new projects increasingly default to `Gateway API`. It feels like the earlier move from `ReplicationController` to `Deployment`: the old still works; the new is the direction.

### AI / GPU: An Old Captain Meets New Cargo

Large-model training and inference pushed GPUs, high-speed networking, and huge job scheduling back to center stage. `Kubernetes` was not invented for LLMs, but it is already the only cluster substrate general enough for most companies—so Device Plugins, topology-aware scheduling, batch jobs, and queue systems keep stacking on top.

Around November 2023, public reports showed Google using GKE and related capabilities to schedule a distributed training job on the order of 50,000 TPU v5e chips. That suggests:

> It may not be the final form of AI infrastructure, but right now it is the most ready layer organizations can actually digest.

At the same time, security, observability, and platform engineering (Internal Developer Platforms) keep hiding `Kubernetes` behind self-service—developers may never touch the cluster directly, while platform teams almost always build on it.

---

## A Timeline for Quick Reference

- **~2003 onward** — Google’s internal Borg manages containerized workloads at scale
- **2013** — Docker makes containers a daily developer tool; `Kubernetes` starts as Project Seven of Nine
- **2014-06-06** — `Kubernetes` is open-sourced
- **2015-07-21** — `Kubernetes` 1.0; CNCF is formed and receives the project
- **2015-08-26** — Google Container Engine (later GKE) reaches GA
- **2016** — `CRI` Alpha (1.5); CoreOS proposes the `Operator`; the ecosystem accelerates
- **2017** — `NetworkPolicy` GA (1.7); `RBAC` GA (1.8); Docker announces `Kubernetes` support; CNCF launches conformance certification
- **2018** — Core workload APIs GA (1.9); EKS / AKS reach GA
- **2019** — `CSI` GA (1.13); `CRD` GA (1.16)
- **2020–2022** — `dockershim` deprecated (1.20) and removed (1.24); `PSP` removed; Pod Security Admission stable (1.25)
- **2023-10** — `Gateway API` v1.0; core resources become stable
- **2024** — Tenth open-source anniversary; continued push into AI, multi-cluster, and platform engineering
- **2025–2026** — `Gateway API` keeps promoting experimental features to stable; K8s remains the default cloud-native substrate

---

## Why It Became the Protagonist of This Chronicle

Looking back over more than a decade, `Kubernetes`’s victory compresses into four words:

1. **Timing**: Docker solved packaging; it filled in scaled operations
2. **Experience**: a decade of Borg scars produced clearer abstractions
3. **Governance**: handing it to the CNCF made even competitors willing to build together
4. **Interfaces**: `CRI`/`CNI`/`CSI` + `CRD`/`Operator` let others keep building on top of it

It was never “simple.” The learning curve is steep, the `YAML` is long and ugly, and production incidents can be dramatic—all fair complaints. But infrastructure rarely wins by being flawless. It wins by being **general enough, extensible enough, and backed by a strong enough ecosystem**.

If you remember only one metaphor, make it this:

> Docker turned software into standard shipping boxes; `Kubernetes` taught the harbor to schedule automatically. The monitoring, service meshes, GitOps, and platform engineering that grew later are mostly docks, cranes, and customs houses added onto that harbor.

Looking ahead, the harbor will not disappear—but it will look less and less like the pier developers face every day. More teams will hide it behind platform engineering and self-service. More workloads—from microservices to batch jobs to AI training and inference—will treat it as the default berth. `Gateway API`, policy as code, multi-cluster, and edge scheduling may keep turning “how to expose, how to govern, how to place across regions” into more standardized shipping lanes.

`Kubernetes` may not be the last chapter forever, but it has already written cloud-native’s common language into infrastructure. What is more likely over the next decade is not building an entirely new harbor from scratch, but raising this one higher and deeper so more people can dock with confidence.

---

## References

- [Borg, Omega, and Kubernetes - ACM Queue](https://queue.acm.org/detail.cfm?id=2898444)
- [How Kubernetes came to be - Google Cloud Blog](https://cloud.google.com/blog/products/containers-kubernetes/from-google-to-the-world-the-kubernetes-origin-story)
- [CNCF founding announcement and Kubernetes donation (2015-07-21)](https://www.cncf.io/announcements/2015/06/21/new-cloud-native-computing-foundation-to-drive-alignment-among-container-technologies/)
- [Introducing CRI in Kubernetes](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)
- [Core Workloads API GA (1.9)](https://kubernetes.io/blog/2018/01/core-workloads-api-ga/)
- [CSI for Kubernetes GA](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/)
- [Kubernetes 1.16: CRD GA](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/)
- [Dockershim removal (1.24)](https://kubernetes.io/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/)
- [Pod Security Admission stable / PSP removed (1.25)](https://kubernetes.io/blog/2022/08/25/pod-security-admission-stable/)
- [Gateway API v1.0: GA Release](https://kubernetes.io/blog/2023/10/31/gateway-api-ga/)
- [KuberTENes: a decade review - The Register](https://www.theregister.com/2024/06/12/kubertenes_decade_anniversary/)
