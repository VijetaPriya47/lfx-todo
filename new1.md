No, you don't need your own cloud. Everest is running entirely on your local machine inside a **kind cluster** (Kubernetes IN Docker). Here's what's happening:

- **kind** spins up a full Kubernetes cluster inside Docker containers on your laptop. The node you saw — `everest-control-plane` — is just a Docker container.
- The **Everest server** (`everest-server-764df6b8fb-jddc2`) and **controller** are pods running inside that local Kubernetes cluster.
- The `kubectl port-forward svc/everest -n everest-system 8080:8080` command tunnels the Everest API from inside the cluster to `localhost:8080` on your machine — that's why `curl http://localhost:8080/v1/...` works.
- The CLI binary (`/tmp/everestctl`) talks to `http://localhost:8080`, which is that tunnel.

So the entire stack — Kubernetes, Everest API, database operators — is running on your laptop's CPU and RAM, not in any cloud. This is standard local development for Kubernetes-based projects. The only things that require internet access are pulling container images and Helm charts, which happen once during setup.

The database operators (PSMDB for MongoDB etc.) also run as pods inside the same local cluster. When you create an instance, the operator pod reconciles it and starts more pods for the actual database — all still on your machine.
