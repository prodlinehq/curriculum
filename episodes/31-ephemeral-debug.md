# Episode 31 — kubectl debug and Ephemeral Containers

| | |
| :--- | :--- |
| **YouTube title** | kubectl debug and Ephemeral Containers |
| **Film order** | 31 of 33 · Phase 4 · Week 31 |
| **Duration** | 10 minutes |
| **Lab repo** | `supercheck-io/supercheck` |
| **Supercheck files** | `kubectl debug; gVisor limitations` |
| **Next** | [32-circuit-breakers.md](32-circuit-breakers.md) |

## Overview

App is not distroless (has node). Jobs may be harder. kubectl debug still the staff skill. Do not kubectl exec as the only story.

Film only Supercheck names: `supercheck-app`, `supercheck-worker-eu`, `supercheck-execution`, Traefik, BullMQ, PlanetScale/Postgres, Redis Sentinel. No toy `demo-api`.


## Episode 31: kubectl debug and Ephemeral Containers

### 1. The Production Problem
*"Our production pod runs `gcr.io/distroless/static`. There is no `bash`, no `sh`, no `ls`, no `curl`. The container is running but deadlocked on an internal socket. Running `kubectl exec -it <pod> -- /bin/sh` fails with: `OCI runtime exec failed: exec: '/bin/sh': stat /bin/sh: no such file or directory`."*

### 2. Deep Technical Breakdown
Distroless images are essential for security, but they make traditional debugging impossible:
1. **Linux Namespaces Under the Hood:** A pod is a collection of Linux namespaces (`netns`, `PIDns`, `mntns`) shared among containers.
2. **Ephemeral Containers (`kubectl debug`):** Kubernetes allows injecting an **ephemeral container** into an existing, live pod sandbox. The debug container runs a tool-rich image (e.g. `nicolaka/netshoot`) while **sharing the network namespace and process namespace** (`shareProcessNamespace: true`) of the target distroless container.
3. **Live Process Forensics:** Inside the ephemeral container, you can run `strace -p 1` to trace system calls, `lsof -i` to inspect open file descriptors and sockets, and `gdb` to inspect core dumps without restarting or modifying the target pod.

### 3. Architecture: Ephemeral Container Injection

```mermaid
flowchart TD
    subgraph PodSandbox["Target Pod: supercheck-app (Shared Linux Namespaces)"]
        APP["Container 1: api<br/>Base: Distroless (No Shell, UID 65532)<br/>PID 1 in Target Container"]
        DEBUG["Container 2: debugger<br/>Base: netshoot (bash, strace, curl, tcpdump)<br/>Injected Ephemeral Container"]
    end

    DEBUG -.->|Shares Network Namespace (netns)| APP
    DEBUG -.->|Shares Process Namespace (PIDns via target=api)| APP
    
    DEBUG ==>|strace -p 1: Trace syscalls live| APP
    DEBUG ==>|lsof -i :3000: Inspect open sockets| APP
```

### 4. Ephemeral Container Capability Matrix

| Debug Operation | Standard Container (`kubectl exec`) | Distroless Container (No Debugger) | Distroless Container (`kubectl debug`) |
| :--- | :---: | :---: | :---: |
| **Shell Access (`/bin/sh`)** | Yes | **Fails (No shell)** | **Yes (Via debug container)** |
| **Inspect Running Processes** | Yes | Fails | **Yes (`ps aux` via shared PIDns)** |
| **Trace System Calls (`strace`)** | Often lacks `CAP_SYS_PTRACE` | Fails | **Yes (`strace -p <PID>`)** |
| **Live Packet Sniffing (`tcpdump`)** | Usually lacks tools | Fails | **Yes (`tcpdump -i eth0`)** |
| **Production Image Immutability** | Compromised if tools installed | Preserved | **100% Preserved & Compliant** |

### 5. CLI Execution Lab
```bash
# 1. Attempt standard exec (Fails on distroless)
kubectl exec -it supercheck-app-7b8f94cb-x4z9q -n supercheck -- /bin/sh
# Output: OCI runtime exec failed: exec: "/bin/sh": stat /bin/sh: no such file or directory

# 2. Inject ephemeral container with shared PID namespace targeting the api container
kubectl debug -it supercheck-app-7b8f94cb-x4z9q -n supercheck \
  --image=nicolaka/netshoot:latest \
  --target=api

# 3. Inside the ephemeral debug shell:
# Inspect target process
ps aux

# Trace system calls of the Go application binary live
strace -p 1 -f -e trace=network,file

# Inspect open network sockets
lsof -i :3000
```

### 6. 10-Minute Video Script Outline
* **1. The Hook (0:00–0:45):** Try to exec into a locked-up production container. Show `OCI runtime exec failed: stat /bin/sh: no such file or directory`. *"Your container has no shell, no tools, and no curl. How do you inspect a live deadlock without restarting it? With ephemeral containers."*
* **2. The Stakes & Blast Radius (0:45–1:45):** The compromise between security and observability. Why leaving curl and bash in production images creates attack vectors, and why restarting a deadlocked pod destroys the evidence.
* **3. Architecture & Mental Model (1:45–3:30):** How Linux namespaces enable ephemeral containers: sharing `netns` and `PIDns` between two independent OCI images in the same pod sandbox.
* **4. Hands-On Implementation (3:30–8:30):**
  - Step 1: Run `kubectl debug` with `--target=api` and `--image=nicolaka/netshoot`.
  - Step 2: Show the target process appearing in `ps aux`.
  - Step 3: Attach `strace -p 1` to watch live incoming HTTP socket reads.
* **5. Verification & Guardrails (8:30–10:00):** Verify that once the debug session ends, the original distroless container remains completely untouched and undisturbed.
* **6. Outro & Call to Action (10:00–10:30):** Rule of thumb: *"Never add debug binaries to production images; inject ephemeral containers instead."* Next: Episode 28 — Cascading Failures & Circuit Breakers.

---

---

## Creator prep (from original kit)

### Episode 31 Preparation: kubectl debug and Ephemeral Containers

#### 1. High-Yield Learning Links (Watch & Read Before Filming)
* **YouTube**: *That DevOps Guy (Viktor Farcic)* — "How to Debug Kubernetes Pods with kubectl debug" ([YouTube Search: Viktor Farcic kubectl debug](https://www.youtube.com/results?search_query=Viktor+Farcic+kubectl+debug))
* **YouTube**: *KubeCon + CloudNativeCon* — "Debugging Distroless Containers in Production" ([YouTube Search: KubeCon Debugging Distroless Containers](https://www.youtube.com/results?search_query=KubeCon+Debugging+Distroless+Containers))
* **Official Docs**: [Debugging Pods with Ephemeral Containers](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)

#### 2. Intuitive Mental Model
* **The Paramedic with the Heavy Diagnostic Backpack**:
  * In modern production, your containers are built using **Distroless** or Scratch images. There is no `bash`, no `sh`, no `curl`, no `netstat`, and no `apt-get` for maximum security.
  * When an incident strikes, typing `kubectl exec -it <pod> -- /bin/sh` fails immediately.
  * **`kubectl debug`** is like air-dropping a fully equipped paramedic into the exact same room: it injects an ephemeral container (`nicolaka/netshoot`) that shares the exact same Linux network namespace and PID namespace as the locked-down container, allowing you to run `tcpdump`, `curl`, and `strace` without modifying or restarting the application!

#### 3. Pre-Flight (Supercheck app is not distroless — Jobs + gVisor are the hard case)

The Next.js image is `node:22-slim` (shell exists). `kubectl exec` often works on `supercheck-app`. Production pain is **gVisor Jobs** in `supercheck-execution` and workers cleaning Playwright dirs on preStop.

```bash
POD=$(kubectl get pod -n supercheck -l app.kubernetes.io/component=app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n supercheck "$POD" -c app -- node -e 'console.log("ok")'
kubectl debug -it "$POD" -n supercheck --image=nicolaka/netshoot --target=app -- bash

# Jobs: runtimeClassName gvisor — debug/attach may be restricted; film that constraint
kubectl get pod -n supercheck-execution -o jsonpath='{range .items[*]}{.metadata.name} {.spec.runtimeClassName}{"\n"}{end}' | head
```

#### 4. YouTube Retention & Growth Kit
* **Clickable Titles**:
  1. *How to Debug Distroless Containers (No Shell? No Problem!)*
  2. *Mastering `kubectl debug`: The Modern Kubernetes Troubleshooting Superpower*
  3. *Debug Live Kubernetes Pods Without Restarting Them*
* **Thumbnail Concept**: Split terminal: Left: `kubectl exec -it ... ERROR: /bin/sh not found`. Right: `kubectl debug ... Connected via netshoot!` with Wireshark and socket monitors running. Bold text: **"NO SHELL? NO PROBLEM"**
* **30-Second Hook**: *"You're facing a high-severity production incident. You type `kubectl exec -it my-pod -- /bin/sh` to inspect network connections, only to get slapped with: `OCI runtime exec failed: /bin/sh not found`. Your company adopted secure distroless containers, leaving you with no shell, no curl, and no package manager. How do you troubleshoot? In this video, we master `kubectl debug` to inject modern diagnostic toolkits into live running pods without breaking security or restarting the process."*
* **Beginner Gotcha**: Remember that ephemeral containers cannot be removed once added to a pod spec! They remain in the pod until the pod terminates. Also, to inspect processes with `ps aux` or attach with `strace`, you must specify `--target=<container-name>` to share the target container's PID namespace.

---
