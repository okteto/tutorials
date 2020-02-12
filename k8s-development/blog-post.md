# State of the Art of In-Cluster Development on Kubernetes

There is an increasing need for in-cluster development.

**Problem**: current dev tools are designed to support a development model where both the relevant files (e.g., source code) and the required tools (e.g., compilers) reside on the developer's workstation. Call this is a *local development paradigm*.

Why this blog post? Growing ecosystem for in-clutser development, great tools, hard to stay tuned!

Analysis covers:
1. Are development environments replicable?
1. How fast is to move my latest changes to the cluster (code, dependencies, k8s manifests).
1. Debuggers.
1. Development decoupled from deployment. This allows developing something deployed by an external tool.
1. Client-side only.

Each group covers analysis on the previous properties and a demo of developing a Hello World application in Go.

## Group 1: Automatic Deployment on code changes

**Skaffold**, draft, tilt, garden, ...
(the demo covers the tool in bold)

Pros: replicable, all automatic, decouple, client-side only

Cons: recreating pod is slow. Hard to integrate debuggers

## Group 2: Network abstraction

**Telepresence**, kubefwd, ...
(the demo covers the tool in bold)

Pros: code is local, everything is fast. Debuggers, decouple.

Cons: not replicable, not client-side only. Bad latency from-to the cluster if the cluster is remote.

## Group 3: Code sync

*Workstation-resident file base*

**Okteto**, ksync, dev spaces, dev pod, ...
(the demo covers the tool in bold)

Pros: Replicable, fast, debuggers, decouple, client-side only

Const: Base images changes. Build in cluster consumes resources.

## Group 4: Online IDE running in k8s

*Kubernetes-resident file base*

**Coder**, Eclipse Che, ...
(the demo covers the tool in bold)

Pros: Replicable, fast, debuggers, decouple, client-side only

Const: Unable to work without internet. Latency editing files.