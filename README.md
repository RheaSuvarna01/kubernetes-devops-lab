# Lab 2 — hello-app on Kubernetes

Your copy of this repository is your submission. The lab sheet is on the course site; this file is where you record what you did.

What's here:

| File | What it is |
|---|---|
| `src/`, `index.html` | a small React page that shows which Pod answered |
| `nginx.conf` | serves the page, and answers `/whoami` with the Pod name |
| `Dockerfile` | builds the page, then serves it with nginx on port 80 |

You don't change any of these. You add `hello-app.yaml` and fill in the sections below.

## Before you push, check

- [ ] `hello-app.yaml` is in the root of this repository
- [ ] `screenshot.png` is in the root of this repository
- [ ] every section below is filled in

---

## 1. My image
DockerHub image I deployed: `suvarnarhea/hello-app:v1`

```
suvarnarhea/hello-app:v1
```

## 2. Pods across nodes

Output of `kubectl get pods -o wide` after you applied `hello-app.yaml`:

```
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE              NOMINATED NODE   READINESS GATES
hello-app-7bb55f9645-5xshw   1/1     Running   0          20m   10.244.2.7   desktop-worker2   <none>           <none>
hello-app-7bb55f9645-b2pnb   1/1     Running   0          20m   10.244.1.6   desktop-worker    <none>           <none>
hello-app-7bb55f9645-mjbvj   1/1     Running   0          20m   10.244.1.7   desktop-worker    <none>           <none>
hello-app-7bb55f9645-plxkb   1/1     Running   0          17m   10.244.2.8   desktop-worker2   <none>           <none>
```

## 3. The Service and its endpoints

Output of `kubectl get service hello-app` and `kubectl get endpointslices -l kubernetes.io/service-name=hello-app`:

```
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-app   LoadBalancer   10.96.221.96   172.18.0.7    8080:30388/TCP   20m

NAME              ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
hello-app-2dv7n   IPv4          80      10.244.1.6,10.244.2.7,10.244.1.7 + 1 more...   20m
```

## 4. Browser

Add `screenshot.png`: the page at `http://localhost:8080` with at least two Pods in the "Answers so far" table. : `kubernetes-devops-lab/screenshot.png`

## 5. The four traps

For each trap, write what you saw and why it happened, in two or three lines of your own words.

**Trap 1 — the local image.**
The Pods stayed at 0/1 with ErrImagePull, then ImagePullBackOff, and `kubectl describe pod` showed "pull access denied" for `hello-app:v1`. The cluster's nodes have their own image storage and can't see images in my laptop's Docker, and a name with no username means the official Docker Hub image, which doesn't exist. Pushing the image as `suvarnarhea/hello-app:v1` fixed it.

**Trap 2 — the selector typo.**
After changing the Service selector to `app: helo-app`, the Service and the Pods still existed, but the EndpointSlice had no endpoints and the page showed an error. No Pod has the label `app: helo-app`, so the Service had nowhere to send traffic. Fixing the spelling brought the endpoints and the page back.

**Trap 3 — the wrong targetPort.**
With `targetPort: 8080`, the EndpointSlice still listed all the Pods, but on port 8080, and the page showed an error. The Service found the Pods but sent traffic to port 8080, where nothing listens, because nginx listens on port 80. Setting it back to 80 fixed it.

**Trap 4 — the EXTERNAL-IP address.**
`kubectl get service hello-app` showed an EXTERNAL-IP like 172.21.0.8, but `http://172.21.0.8:8080` never loaded. That address is inside Docker Desktop's private network, which my laptop can't reach. Docker Desktop also publishes LoadBalancer ports on localhost, so `http://localhost:8080` worked.

## 6. Your YAML vs the generated YAML

`strategy: {}` and `status: {}`. My file works without them because they're optional. When Kubernetes creates the object it fills in defaults, so the strategy becomes a rolling update. `status` is written by the cluster to report the object's current state, so it never belongs in a file you apply.
