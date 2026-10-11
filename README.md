# K8s Fundamentals Labs

Hands-on Kubernetes networking labs for a 3-node [kind](https://kind.sigs.k8s.io/) cluster on a Mac.

This page follows one packet through the cluster, from inside a pod out to an Ingress. Each section has a diagram, the hops in order, and the commands to watch it happen.

## Three rules everything builds on

1. Every pod gets its own IP address.
2. Any pod can reach any other pod by that IP, with no NAT in between.
3. Everything else (Services, NodePort, load balancers, Ingress) is a way to find or reach a pod IP.

## Reading the diagrams

| Mark | Meaning |
|---|---|
| Orange line | The packet being followed |
| Blue line | A second path or the reply |
| Dashed grey line | Possible but not taken |
| Grey box | Node |
| Blue box | Pod |

The addresses are the same in every diagram so you can follow them from one to the next. Yours will differ; `kubectl get pods -o wide` and `kubectl get nodes -o wide` show the real ones.

| Thing | Example address |
|---|---|
| worker1 node | `172.18.0.3`, pod CIDR `10.244.1.0/24` |
| worker2 node | `172.18.0.4`, pod CIDR `10.244.2.0/24` |
| pod A (client) | `10.244.1.5` on worker1 |
| pod B / api-2 | `10.244.2.7` on worker2 |
| Service `api` | `10.96.0.50:80` → pod port `8080` |
| CoreDNS Service | `10.96.0.10` |
| NodePort | `30080` |
| Load balancer IP | `172.18.255.200` |

## Contents

Start with [Set up the lab](#set-up-the-lab). Each later section adds one small piece under its own "Set up" heading, in order. Finish with [Clean up](#clean-up).

1. [Inside one pod](#1-inside-one-pod)
2. [Pod to pod: same node and across nodes](#2-pod-to-pod-same-node-and-across-nodes)
3. [Routed versus overlay](#3-routed-versus-overlay)
4. [A Service IP is a rewrite rule](#4-a-service-ip-is-a-rewrite-rule)
5. [How a name becomes a Service IP](#5-how-a-name-becomes-a-service-ip)
6. [NodePort: opening a door on every node](#6-nodeport-opening-a-door-on-every-node)
7. [Load balancer: one stable IP in front of the nodes](#7-load-balancer-one-stable-ip-in-front-of-the-nodes)
8. [Ingress: routing by hostname and path](#8-ingress-routing-by-hostname-and-path)
9. [The whole trip, layer by layer](#9-the-whole-trip-layer-by-layer)

---

## Set up the lab

Do this once per machine. It starts from nothing: install the tools, clone this repo, then build the cluster. The files are in [setup/](setup/).

### Step 1: install the tools (once per machine)

You need five things: Git, Docker, kind, kubectl and Helm. Run **only the block for your operating system** and skip anything you already have.

**macOS** (uses Homebrew). There is no `usermod` step on a Mac; Docker Desktop needs no group setup.

```sh
brew install --cask docker
brew install git kind kubectl helm
```

**Linux only.** The `usermod` line lets you run `docker` without `sudo`; it does not exist on macOS.

```sh
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER      # then log out and back in
brew install git kind kubectl helm # Homebrew on Linux, or use your package manager
```

**Windows.** Install WSL2 and Docker Desktop, then do everything else inside the WSL (Ubuntu) terminal.

```sh
winget install Docker.DockerDesktop
# then, inside WSL, install git, kind, kubectl and helm (last line of the Linux block)
```

The commands on this page are written for a bash or zsh shell. On Windows, use the WSL terminal, not PowerShell.

### Step 2: clone the repo and go into it

```sh
git clone https://github.com/sandeshlamsal/K8sFundamentals_Labs.git
cd K8sFundamentals_Labs
ls setup/                      # should list kind-3node.yaml and 01- to 04- files
```

Stay in this folder. Every later command that mentions `setup/…` is relative to it, so if you open a new terminal, `cd K8sFundamentals_Labs` again first.

### Step 3: start Docker and check the tools

```sh
# start Docker: pick the line for your machine
open -a Docker                 # macOS: wait for the whale icon to settle
sudo systemctl start docker    # Linux
                               # Windows: start Docker Desktop from the Start menu

docker info | head -5          # must not say "Cannot connect"
kind version && kubectl version --client && helm version --short
```

### Step 4: create three nodes

Each kind node is a Docker container that runs a full Kubernetes node inside it. [setup/kind-3node.yaml](setup/kind-3node.yaml) makes one control plane and two workers, and forwards `localhost:8080` on the Mac into the cluster for the Ingress section.

<details open>
<summary><code>setup/kind-3node.yaml</code>, every line explained</summary>

```yaml
# kind cluster definition: 1 control-plane node + 2 worker nodes.
# Used by:  kind create cluster --config setup/kind-3node.yaml

# kind: tells the "kind" tool what this document describes. "Cluster" is the only
# top-level type a kind config file can hold.
kind: Cluster

# apiVersion: the version of kind's config format. v1alpha4 is the current one;
# kind refuses the file if this line is missing or wrong.
apiVersion: kind.x-k8s.io/v1alpha4

# networking: cluster-wide network settings. Anything not listed keeps kind's default.
# Defaults we rely on but do not set (they explain the addresses in the diagrams):
#   podSubnet: 10.244.0.0/16      -> every pod IP comes from here, one /24 slice per node
#   serviceSubnet: 10.96.0.0/16   -> every Service (ClusterIP) comes from here
#   disableDefaultCNI: false      -> kind installs its own CNI (kindnet), so nodes go Ready
networking:
  # kubeProxyMode: how kube-proxy turns a Service into something the node kernel enforces.
  #   iptables -> one chain of DNAT rules per Service and per pod behind it
  #   nftables -> the same idea in the newer nftables engine, inspected with "nft"
  #   ipvs     -> the kernel's IPVS load balancer; being phased out upstream
  #   none     -> no kube-proxy at all (for a CNI such as Cilium that replaces it)
  # Why iptables here, written out even though it is kind's default:
  #   1. Section 4 reads the Service rules with "iptables-save". In any other mode
  #      that command shows nothing for Services and the lab step fails.
  #   2. Its chains read like the diagram: KUBE-SERVICES (match the Service IP)
  #      -> KUBE-SVC-... (pick one pod) -> KUBE-SEP-... (rewrite the destination).
  #   3. Stating it keeps the lab working if a later kind release changes the default.
  kubeProxyMode: iptables

# nodes: one list entry per node. Each entry becomes one Docker container, named
# <cluster>-<role>: kind-control-plane, kind-worker, kind-worker2.
nodes:

# role: control-plane -> this node runs the cluster's brain: API server, etcd,
# scheduler, controller-manager. A cluster needs at least one. It carries a
# NoSchedule taint, so the lab pods land on the workers, not here.
- role: control-plane

  # extraPortMappings: publish a port of this node container on the Mac, exactly like
  # "docker run -p". Needed because Docker Desktop runs inside a VM: the Mac cannot
  # reach the nodes' 172.18.x.x addresses, only ports published this way.
  # It can only be set when the cluster is created, which is why it is here already.
  extraPortMappings:

    # containerPort: the port inside the node container to expose. 30081 is the
    # NodePort we give the ingress controller in section 8. A NodePort is open on
    # every node, so publishing it from this one node is enough.
  - containerPort: 30081

    # hostPort: the port on the Mac. http://localhost:8080 therefore reaches
    # NodePort 30081. 8080 avoids the privileged, often-taken port 80.
    hostPort: 8080

    # protocol: TCP, because HTTP runs over TCP. TCP is the default; it is written
    # out so nobody has to guess.
    protocol: TCP

# role: worker -> runs only kubelet, kube-proxy, the CNI and your pods.
# Two workers are the minimum that lets us compare "same node" with "across nodes".
# The first becomes kind-worker (worker1 in the diagrams).
- role: worker

# The second becomes kind-worker2 (worker2 in the diagrams).
- role: worker
```

</details>

```sh
pwd                            # must end in /K8sFundamentals_Labs
kind create cluster --config setup/kind-3node.yaml
kubectl get nodes -o wide      # three nodes, all Ready, with their 172.18.x.x addresses
# which pod address range each node owns
kubectl get nodes -o 'custom-columns=NODE:.metadata.name,IP:.status.addresses[0].address,PODCIDR:.spec.podCIDR'
docker ps --format '{{.Names}}'   # the same three nodes, seen as containers
```

| In the diagrams | Real node name | How to get a shell on it |
|---|---|---|
| worker1 | `kind-worker` | `docker exec -it kind-worker bash` |
| worker2 | `kind-worker2` | `docker exec -it kind-worker2 bash` |
| (not drawn) | `kind-control-plane` | `docker exec -it kind-control-plane bash` |

kind installs its own CNI (kindnet) during this step, which is why the nodes reach `Ready`.

### Step 5: start the test pods

[setup/01-pods.yaml](setup/01-pods.yaml) creates four pods, pinned to nodes so they match the diagrams: `web`, `pod-a` and `pod-c` on `kind-worker`, and `pod-b` on `kind-worker2`. They run netshoot, an image with `curl`, `ping`, `ip`, `nslookup` and `tcpdump` built in.

<details open>
<summary><code>setup/01-pods.yaml</code>, every line explained</summary>

```yaml
# Four test pods for sections 1 to 3.
# Used by:  kubectl apply -f setup/01-pods.yaml

# ---------------------------------------------------------------------------
# Pod "web": two containers in ONE pod, to show they share localhost (section 1).
# ---------------------------------------------------------------------------

# apiVersion: which API group and version defines this object. Pods belong to the
# original "core" group, which is written as plain v1.
apiVersion: v1

# kind: the type of object. A bare Pod (no Deployment around it) is not recreated
# if it dies. That is fine here and keeps its IP stable while you study it.
kind: Pod

metadata:
  # name: how you address the pod in kubectl, e.g. "kubectl exec web ...".
  name: web

  # labels: free-form key/value tags. On their own they do nothing; the Service
  # "web" in 04-ingress.yaml finds this pod by looking for app=web.
  labels:
    app: web

spec:
  # nodeSelector: only schedule this pod on a node carrying this label.
  # kubernetes.io/hostname is a label every node gets automatically.
  # We pin the pod so the lab matches the diagrams: web lives on worker1.
  nodeSelector:
    kubernetes.io/hostname: kind-worker

  # containers: every container listed here shares the pod's single network
  # namespace: one IP address, one eth0, one localhost.
  containers:

    # name: needed to pick a container, e.g. "kubectl exec web -c nginx".
  - name: nginx

    # image: nginx is a ready-made web server on port 80. The tag is pinned so the
    # lab behaves the same every time; "alpine" is the small variant.
    image: nginx:1.27-alpine

    # ports/containerPort: documentation only. It does NOT open or publish anything;
    # nginx listens on 80 whether this line exists or not.
    ports:
    - containerPort: 80

    # Second container in the same pod. It will curl localhost:80 and reach nginx,
    # proving both containers share one network namespace.
  - name: log-sidecar

    # image: netshoot is a troubleshooting image with curl, ping, ip, nslookup, dig
    # and tcpdump preinstalled. The nginx image has none of these.
    image: nicolaka/netshoot:v0.13

    # command: replaces the image's default command (an interactive shell, which
    # exits at once with no terminal attached). "sleep infinity" keeps the container
    # alive doing nothing, so you can "kubectl exec" into it whenever you like.
    command: ["sleep", "infinity"]

# "---" separates one Kubernetes object from the next inside a single file.
---
# ---------------------------------------------------------------------------
# Pod "pod-a": the client in every diagram. Lives on worker1.
# ---------------------------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
spec:
  # Pinned to worker1 (kind-worker), the same node as pod-c.
  nodeSelector:
    kubernetes.io/hostname: kind-worker
  containers:
  - name: netshoot
    # Network tools, kept alive with sleep; see the notes on pod "web" above.
    image: nicolaka/netshoot:v0.13
    command: ["sleep", "infinity"]
---
# ---------------------------------------------------------------------------
# Pod "pod-c": the SAME-node target. pod-a -> pod-c never leaves worker1.
# ---------------------------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: pod-c
spec:
  # Same node as pod-a on purpose.
  nodeSelector:
    kubernetes.io/hostname: kind-worker
  containers:
  - name: netshoot
    image: nicolaka/netshoot:v0.13
    command: ["sleep", "infinity"]
---
# ---------------------------------------------------------------------------
# Pod "pod-b": the CROSS-node target. pod-a -> pod-b must travel worker1 -> worker2.
# ---------------------------------------------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
spec:
  # The other worker on purpose, so the packet has to use the route the CNI installed.
  nodeSelector:
    kubernetes.io/hostname: kind-worker2
  containers:
  - name: netshoot
    image: nicolaka/netshoot:v0.13
    command: ["sleep", "infinity"]
```

</details>

```sh
kubectl apply -f setup/01-pods.yaml
kubectl wait --for=condition=Ready pod --all --timeout=300s   # the first image pull takes a few minutes
kubectl get pods -o wide       # note each pod's IP and NODE column
```

You now have everything sections 1 to 3 need.

---

## 1. Inside one pod

A pod is a private network namespace with one interface, `eth0`. All containers in the pod share it, so they talk over `localhost`. The other end of `eth0` is a virtual cable (a veth pair) that surfaces on the node.

![A pod's containers share one network namespace; its eth0 is one end of a veth pair whose other end sits in the node](docs/diagrams/01-inside-a-pod.svg)

Anything leaving the pod crosses the veth pair and lands in the node's routing table, which decides where it goes next.

```sh
# containers in one pod share localhost
kubectl exec web -c log-sidecar -- curl -s localhost:80
# the pod's only interface
kubectl exec web -c log-sidecar -- ip addr show eth0
# the node's route to that pod (one line, naming a veth)
docker exec kind-worker ip route | grep "$(kubectl get pod web -o jsonpath='{.status.podIP}')"
```

## 2. Pod to pod: same node and across nodes

Pod A pings pod C on the same node, then pod B on another node. Both use the same logic: leave through the veth, let the node's routing table choose. The only difference is whether the next hop is a local veth or another node.

![Pod A reaches pod C through worker1's routing table, and pod B by being routed from worker1 to worker2 with pod IPs unchanged](docs/diagrams/02-pod-to-pod.svg)

The route `10.244.2.0/24 via 172.18.0.4` is what the CNI installs. Without a CNI that line is missing and cross-node traffic has nowhere to go.

1. Pod A sends to `10.244.2.7`. Its only route is "everything goes out `eth0`".
2. The packet appears on worker1. The routing table matches `10.244.2.0/24 via 172.18.0.4`.
3. worker1 forwards it out its own `eth0` to worker2.
4. worker2 matches `10.244.2.7 dev vethB` and pushes it down that veth into pod B.
5. Pod B sees the source as `10.244.1.5`, pod A's real IP. The reply takes the same path back.

```sh
kubectl get pods -o wide                      # pod IPs and which node each is on
docker exec kind-worker ip route              # the routes in the diagram
# watch packets arrive on worker2 while you ping from pod A
kubectl debug node/kind-worker2 -it --profile=sysadmin --image=nicolaka/netshoot:v0.13 -- tcpdump -ni any icmp
# in a second terminal: same node first, then across nodes
kubectl exec pod-a -- ping -c 3 "$(kubectl get pod pod-c -o jsonpath='{.status.podIP}')"
kubectl exec pod-a -- ping -c 3 "$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')"
```

## 3. Routed versus overlay

CNIs differ mainly in how they carry a pod packet between nodes. A routed CNI sends it as it is. An overlay CNI wraps it inside a node-to-node packet, which works even when the network between nodes knows nothing about pod IPs.

![A routed packet carries pod IPs directly; an overlay packet adds an outer node-to-node IP header and a VXLAN header](docs/diagrams/03-routed-vs-overlay.svg)

The orange headers are the cost of an overlay: about 50 extra bytes per packet, in exchange for not needing pod routes on the underlying network.

## 4. A Service IP is a rewrite rule

Pod IPs change every time a pod restarts, so clients use a stable Service IP instead. No interface anywhere owns that IP. kube-proxy writes rules on every node that rewrite the destination to one real pod before the packet is routed.

![Pod A sends to the Service IP; iptables rules on its own node rewrite the destination to one of three backing pod IPs](docs/diagrams/04-service-clusterip.svg)

The rewrite happens on the client's own node. After it, the packet is an ordinary pod-to-pod packet and follows diagram 2.

1. The Service's selector matches three pods; Kubernetes records their IPs in an EndpointSlice.
2. kube-proxy on every node turns that list into rules: one per Service, one per endpoint.
3. Pod A's packet to `10.96.0.50:80` hits those rules and leaves with destination `10.244.2.7:8080`.
4. Scale the Deployment and the EndpointSlice changes, so the rules change within a second or two.

**Set up: three api pods and a Service** ([setup/02-api.yaml](setup/02-api.yaml))

<details open>
<summary><code>setup/02-api.yaml</code>, every line explained</summary>

```yaml
# The "api" application: three identical pods plus one Service in front of them.
# Used by:  kubectl apply -f setup/02-api.yaml   (sections 4 to 8)

# apiVersion: Deployments live in the "apps" API group, version v1.
apiVersion: apps/v1

# kind: a Deployment keeps a set number of identical pods running and replaces any
# that die. Replacements get NEW IP addresses, which is exactly why clients need a
# Service instead of talking to pod IPs.
kind: Deployment

metadata:
  # name: the Deployment's name. Its pods are named api-<hash>-<random>.
  name: api

spec:
  # replicas: how many pods to keep running. Three gives the Service something to
  # choose between and usually puts pods on both workers.
  replicas: 3

  # selector: how the Deployment recognises its own pods. It must match the labels
  # in "template" below, otherwise the API server rejects the Deployment.
  selector:
    matchLabels:
      app: api

  # template: the blueprint every pod is stamped from.
  template:
    metadata:
      # labels: put on each pod. Two things look for app=api:
      # the Deployment's selector above and the Service's selector below.
      labels:
        app: api
    spec:
      containers:
      - name: whoami

        # image: a tiny web server that answers any request with its own hostname
        # and the address the request came from (RemoteAddr). That makes it visible
        # which pod answered and whether the client IP survived the trip.
        image: traefik/whoami:v1.10

        # args: options handed to the program. whoami listens on 80 by default;
        # we move it to 8080 so the Service visibly translates port 80 -> 8080.
        args: ["--port=8080"]

        # containerPort: documentation of the listening port. Informational only.
        ports:
        - containerPort: 8080
---
# apiVersion: Services belong to the core group, written as v1.
apiVersion: v1

# kind: a Service is a stable virtual IP and DNS name in front of changing pods.
kind: Service

metadata:
  # name: becomes the DNS name. Pods in the same namespace reach it as "api",
  # in full: api.default.svc.cluster.local
  name: api

spec:
  # type: not set, so it is ClusterIP: reachable only from inside the cluster.
  # Section 6 patches it to NodePort and section 7 to LoadBalancer.

  # selector: send traffic to every Ready pod labelled app=api. Kubernetes keeps the
  # matching pod IPs in an EndpointSlice, and kube-proxy turns that into rules.
  selector:
    app: api

  ports:
    # port: the port clients dial on the Service IP (http://api:80).
  - port: 80

    # targetPort: the port on the pod the traffic is rewritten to. It must equal the
    # port the container really listens on (the --port=8080 above).
    targetPort: 8080
```

</details>

```sh
kubectl apply -f setup/02-api.yaml
kubectl rollout status deploy/api
kubectl get svc api            # CLUSTER-IP is the Service IP in the diagram
kubectl get pods -l app=api -o wide
```

**Watch it**

```sh
kubectl get endpointslices -l kubernetes.io/service-name=api
docker exec kind-worker iptables-save -t nat | grep -E 'KUBE-(SVC|SEP)' | grep api
# hit the Service 6 times and see different pods answer
kubectl exec pod-a -- sh -c 'for i in 1 2 3 4 5 6; do curl -s api | grep Hostname; done'
# scale and watch the endpoint list follow
kubectl scale deploy api --replicas=5 && kubectl get endpointslices -l kubernetes.io/service-name=api
kubectl scale deploy api --replicas=3
```

## 5. How a name becomes a Service IP

Nobody types `10.96.0.50`. Every pod is started with a `resolv.conf` pointing at CoreDNS, which watches the API server and answers with the Service's IP. DNS only hands back an address; the traffic itself never passes through CoreDNS.

![Pod A asks CoreDNS for the api Service name, gets back the ClusterIP, then connects to that IP itself](docs/diagrams/05-dns.svg)

Typing `curl api` works because the first `search` entry is appended, giving `api.default.svc.cluster.local`.

`ndots:5` means a name with fewer than five dots is tried against each search entry first. That is why one lookup of an outside name produces four queries:

```text
curl example.com
  example.com.default.svc.cluster.local   → NXDOMAIN
  example.com.svc.cluster.local           → NXDOMAIN
  example.com.cluster.local               → NXDOMAIN
  example.com.                            → answer
```

```sh
kubectl exec pod-a -- cat /etc/resolv.conf
kubectl exec pod-a -- nslookup api
# watch the four queries go by
kubectl debug node/kind-worker -it --profile=sysadmin --image=nicolaka/netshoot:v0.13 -- tcpdump -ni any port 53
```

## 6. NodePort: opening a door on every node

A NodePort Service opens the same port on every node, whether or not that node runs a matching pod. A node without one forwards the packet to a node that has, and rewrites the source on the way so the reply comes back through it.

![A client hits port 30080 on worker1, which forwards to the pod on worker2 and rewrites the source; with externalTrafficPolicy Local the client goes straight to worker2](docs/diagrams/06-nodeport.svg)

The default policy (`Cluster`) works from any node but hides the client's IP. `Local` keeps the IP but only nodes that run a pod answer.

**Set up: turn the Service into a NodePort**

```sh
kubectl patch svc api -p '{"spec":{"type":"NodePort","ports":[{"port":80,"nodePort":30080}]}}'
kubectl get svc api            # PORT(S) now shows 80:30080/TCP
# the real node addresses
W1=$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' kind-worker)
W2=$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' kind-worker2)
echo $W1 $W2
```

**Watch it**

The Mac cannot reach node IPs directly, because Docker runs in a VM. A throwaway container on the same Docker network plays the outside client.

```sh
# default policy: RemoteAddr is a node-side address, not the client's
docker run --rm --network kind nicolaka/netshoot:v0.13 curl -s ${W1}:30080 | grep -E 'Hostname|RemoteAddr'

# Local policy: RemoteAddr becomes the client container's own 172.18.x.x address
kubectl patch svc api -p '{"spec":{"externalTrafficPolicy":"Local"}}'
docker run --rm --network kind nicolaka/netshoot:v0.13 curl -s ${W1}:30080 | grep -E 'Hostname|RemoteAddr'

# Local policy with one pod: the node without the pod stops answering
kubectl scale deploy api --replicas=1 && kubectl get pods -l app=api -o wide
docker run --rm --network kind nicolaka/netshoot:v0.13 curl -s -m 3 ${W1}:30080 | grep Hostname
docker run --rm --network kind nicolaka/netshoot:v0.13 curl -s -m 3 ${W2}:30080 | grep Hostname

# put it back
kubectl scale deploy api --replicas=3
kubectl patch svc api -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

## 7. Load balancer: one stable IP in front of the nodes

A `LoadBalancer` Service asks something outside Kubernetes for an address. In a cloud that is the provider's load balancer. On a Mac it is MetalLB: one node claims the IP by answering ARP for it, and from there the packet follows the same rewrite rules as before.

![The client asks by ARP who owns the load balancer IP; the MetalLB speaker on worker1 answers, and iptables rewrites the packet to a pod](docs/diagrams/07-load-balancer.svg)

Without MetalLB nothing answers step 1, which is why `EXTERNAL-IP` stays `<pending>` on a bare kind cluster. The pod can be on any node; worker1 is only the entry point.

**Set up: install MetalLB and give it addresses** ([setup/03-metallb-pool.yaml](setup/03-metallb-pool.yaml))

<details open>
<summary><code>setup/03-metallb-pool.yaml</code>, every line explained</summary>

```yaml
# Tells MetalLB which addresses it may hand out and how to announce them.
# Apply AFTER installing MetalLB, because these object types only exist once
# MetalLB's own definitions (CRDs) are installed.
# Used by:  kubectl apply -f setup/03-metallb-pool.yaml   (section 7)

# apiVersion: a custom API group that MetalLB adds to the cluster.
apiVersion: metallb.io/v1beta1

# kind: an IPAddressPool is a range of addresses MetalLB may give to Services of
# type LoadBalancer. With no pool, such Services stay at EXTERNAL-IP <pending>.
kind: IPAddressPool

metadata:
  # name: any name; the L2Advertisement below could refer to it.
  name: kind-pool

  # namespace: MetalLB only reads its configuration from its own namespace.
  # Put this anywhere else and it is silently ignored.
  namespace: metallb-system

spec:
  # addresses: the range to hand out. Two rules decide the choice:
  #   1. It must be INSIDE the kind Docker network (normally 172.18.0.0/16), because
  #      layer-2 mode works by ARP and ARP only reaches machines on the same network.
  #   2. It must not collide with addresses Docker gives to containers. Docker
  #      assigns from the bottom (172.18.0.2, .3, ...), so we take the far top.
  # The range differs per machine (172.18, 172.19, 172.20 ...). The command in
  # section 7 swaps 172.18 for your machine's prefix before applying this file.
  # Check your network with:
  #   docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1

# kind: an L2Advertisement switches on layer-2 mode: one node answers ARP requests
# for each assigned address, so traffic for that address arrives at that node.
# Without this object MetalLB assigns the IP but nobody answers for it.
kind: L2Advertisement

metadata:
  name: kind-l2
  # Same rule as above: must be in MetalLB's namespace.
  namespace: metallb-system

# No "spec" on purpose: an empty L2Advertisement announces every pool.
```

</details>

```sh
# before MetalLB: ask for a load balancer and watch it stay pending
kubectl patch svc api -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc api            # EXTERNAL-IP <pending>

# install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
kubectl wait -n metallb-system --for=condition=Ready pod -l app=metallb --timeout=180s

# the pool must sit inside the kind Docker network, whose range differs per machine
docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
# take the first two numbers of the IPv4 range (for example 172.18 or 172.20) ...
PREFIX=$(docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}' | tr ' ' '\n' | grep -E '^[0-9]+\.' | cut -d. -f1-2)
echo $PREFIX
# ... and apply the pool file with 172.18 swapped for your prefix
sed "s/172\.18\./${PREFIX}./g" setup/03-metallb-pool.yaml | kubectl apply -f -
kubectl get ipaddresspool -n metallb-system
```

**Watch it**

```sh
kubectl get svc api                           # EXTERNAL-IP is now filled in, ending in .255.200
LB=$(kubectl get svc api -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $LB
# which node answers for that address (the MAC in the reply belongs to one node)
docker run --rm --network kind nicolaka/netshoot:v0.13 arping -c 2 $LB
docker run --rm --network kind nicolaka/netshoot:v0.13 curl -s $LB | grep -E 'Hostname|RemoteAddr'
```

## 8. Ingress: routing by hostname and path

Everything so far works on IPs and ports. An ingress controller is an ordinary proxy pod that reads the HTTP request and chooses a backend from the `Host` header and path, so many apps can share one load balancer IP.

![A request for shop.local/api reaches the ingress controller pod, which matches host and path and forwards to an api pod](docs/diagrams/08-ingress.svg)

An `Ingress` or `HTTPRoute` object is only the rule table in the middle. Without a controller pod running, the object does nothing.

**Set up: install a controller and add the rules** ([setup/04-ingress.yaml](setup/04-ingress.yaml))

<details open>
<summary><code>setup/04-ingress.yaml</code>, every line explained</summary>

```yaml
# A Service for the web pod, plus the Ingress rules that route by host and path.
# Apply AFTER installing Traefik.
# Used by:  kubectl apply -f setup/04-ingress.yaml   (section 8)

# An Ingress can only point at Services, never at pods directly, so the web pod
# from 01-pods.yaml needs a Service first.
apiVersion: v1
kind: Service

metadata:
  # name: the Ingress below refers to this Service by this name.
  name: web

spec:
  # selector: matches the label app=web on the pod "web".
  selector:
    app: web

  ports:
    # port: the Service port the Ingress will name.
  - port: 80
    # targetPort: nginx inside the pod listens on 80.
    targetPort: 80
---
# apiVersion: Ingress lives in the networking.k8s.io group, version v1.
apiVersion: networking.k8s.io/v1

# kind: an Ingress is only a table of routing rules. It moves no traffic itself;
# a controller pod (Traefik here) reads it and configures its proxy to match.
kind: Ingress

metadata:
  name: shop

spec:
  # ingressClassName: which controller should act on these rules. The Traefik Helm
  # chart creates a class called "traefik". Leave this out, or name a class nobody
  # serves, and the rules are ignored without any error.
  ingressClassName: traefik

  rules:
    # host: these rules apply only when the request's Host header is shop.local.
    # Any other Host gets a 404 from the controller. This is what lets many sites
    # share one IP address.
  - host: shop.local

    # http: rules for HTTP traffic (the only kind an Ingress understands).
    http:
      paths:

        # path: match on the URL path.
      - path: /api

        # pathType: Prefix matches /api and everything under it (/api/users).
        # The alternatives are Exact (only /api) and ImplementationSpecific.
        pathType: Prefix

        # backend: where matching requests go. "number" is the Service's port (80),
        # not the pod's targetPort (8080); the Service does that translation.
        backend:
          service:
            name: api
            port:
              number: 80

        # The catch-all. The longest matching prefix wins regardless of the order
        # written here, so /api/... goes to "api" and everything else to "web".
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

</details>

```sh
# the controller: Traefik, exposed on NodePort 30081 (which kind maps to localhost:8080)
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm install traefik traefik/traefik -n traefik --create-namespace \
  --set service.type=NodePort --set ports.web.nodePort=30081
kubectl -n traefik rollout status deploy/traefik

# a Service for the web pod, plus the Ingress rules
kubectl apply -f setup/04-ingress.yaml
kubectl get ingress shop
```

**Watch it**

These run straight from the Mac terminal, because `localhost:8080` is forwarded into the cluster.

```sh
# same IP, different Host and path, different backend
curl -H 'Host: shop.local' http://localhost:8080/
curl -H 'Host: shop.local' http://localhost:8080/api
curl -H 'Host: other.local' http://localhost:8080/     # 404: no rule matches
```

## 9. The whole trip, layer by layer

When something breaks, walk this table top to bottom and find the first row whose check fails.

| Layer | Question it answers | Who does it | Check |
|---|---|---|---|
| DNS | What IP is this name? | CoreDNS | `nslookup api` |
| Load balancer | Which node takes traffic for this IP? | MetalLB or cloud | `kubectl get svc` |
| Ingress | Which Service gets this host and path? | Controller pod | `kubectl describe ingress` |
| Service | Which pod backs this Service IP? | kube-proxy rules | `kubectl get endpointslices` |
| Pod network | How do I reach that pod's node? | CNI routes | `ip route` on the node |
| Pod | Is the app listening on that port? | Your container | `kubectl exec … curl localhost` |

## Clean up

```sh
# remove the helper pods that "kubectl debug node/…" leaves behind
kubectl get pods -o name | grep node-debugger | xargs kubectl delete
# delete the whole cluster; "kind create cluster" brings a fresh one back in about a minute
kind delete cluster
```

---

The diagrams live in [docs/diagrams/](docs/diagrams/) as standalone SVG files. The same content as a single web page is in [docs/packet-paths.html](docs/packet-paths.html).
