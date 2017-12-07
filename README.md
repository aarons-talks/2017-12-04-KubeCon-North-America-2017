# kube-best-practices

This is a list of best practices for building an app to run on Kubernetes, using cloud-native technologies.

# Observability is golden

Kubernetes schedules your pods, and is more effective at doing so when it knows:

- What your pods need
	- So tell it this information by adding resource limits
- What is going on with your pods
	- So make sure you have good liveness and readiness checks


At the same time, _you_ need to know what is going on with your pods, as they change over time. So make sure your [logging](https://www.fluentd.org/), [tracing](http://opentracing.io/), [monitoring](https://prometheus.io/) is in place, so you can observe what's going on.

# When all else fails, crash

Crash-only architectures mean that your software should just crash instead of trying to recover more gracefully.

[Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language)) popularized this concept because it introduced a [supervisor](http://erlang.org/doc/man/supervisor.html) principle. The supervisor watches over your code (called a process) and restarts it if it crashes. This architecture encourages you to simply crash if something goes wrong, because you know you'll be restarted and can try again.

If you run in Kubernetes, Kubernetes itself is the "supervisor." Instead of building retry loops or other recovery logic into your app, simply crash and let Kubernetes restart your pod.

# Unordered is better than ordered

Whether you like it or not, your app is a distributed system. You can no longer think of it as a single process (even if you're just running one pod, because the pod can move, crash, etc...). One implication of this fact, among many others, is that it's _hard_ to rely on a specific ordering of events.

[Much](https://amturing.acm.org/p558-lamport.pdf) [research](https://scholar.google.com/scholar?q=ordering+in+distributed+systems&hl=en&as_sdt=0&as_vis=1&oi=scholart&sa=X&ved=0ahUKEwjW0aXfjffXAhUE8IMKHVOnBgcQgQMIJzAA) has been done on the topic of ordering in distributed systems, so if your app needs to rely on ordered events, you need to understand at least the basic ideas (causal ordering, wall clock ordering, etc...), decide what kind of ordering your app needs, implement it, and then test to make sure it adheres to the definition.

Alternatively, you can avoid ordering altogether and/or use the Kubernetes primitives to take care of it for you. Examples of these primitives:

- [Distributed locks](https://github.com/pulcy/kube-lock)
- [Leader election](http://blog.kubernetes.io/2016/01/simple-leader-election-with-Kubernetes.html)
- [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

# Loose coupling is better than tight coupling

As we know, Kubernetes is "always on" and watching over your app. If something changes, it adapts to the change and modifies your app's deployment accordingly to account for the change and bring it back to the state you want it to be in (this is called reconciliation).

Your app needs to tolerate that dynamism.

This fact means a wide variety of things:

- Do not rely on specific pods to be running
- Talk to pods via `Service`s or a service mesh
- Crash if you cannot get a resource or talk to a pod (i.e. crash only software)
- If you need to look for a Kubernetes resource via `kubectl`, the API, or something else, query it by labels, not the name

# But tight coupling isn't _always_ wrong

The "atom" of deployment is a `Pod` in Kubernetes, and pods can have more than one container. This design allows for tight coupling between containers - either they all will be running or none will be running.

So, if you have components of your app that are tightly coupled (e.g. legacy systems, "sidecars", etc...), run them all in containers in the same pod.

We see this topology in many cases, including:

- [Envoy](https://www.envoyproxy.io/) as a sidecar to your app - your app talks to `localhost` to access the service mesh
- The [Fluentd logging driver](https://docs.docker.com/engine/admin/logging/fluentd/) runs as a sidecar, and your app is configured to send logs to `localhost`
- A metrics system you may have runs as a sidecar, and your app sends metrics to `localhost`




