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

Docker Hub image I deployed (for example `ravi123/hello-app:v1`):

```
<paste here>
```

## 2. Pods across nodes

Output of `kubectl get pods -o wide` after you applied `hello-app.yaml`:

```
<paste here>
```

## 3. The Service and its endpoints

Output of `kubectl get service hello-app` and `kubectl get endpointslices -l kubernetes.io/service-name=hello-app`:

```
<paste here>
```

## 4. Browser

Add `screenshot.png`: the page at `http://localhost:8080` with at least two Pods in the "Answers so far" table.

## 5. The four traps

For each trap, write what you saw and why it happened, in two or three lines of your own words.

**Trap 1 — the local image.**

**Trap 2 — the selector typo.**

**Trap 3 — the wrong targetPort.**

**Trap 4 — the EXTERNAL-IP address.**

## 6. Your YAML vs the generated YAML

Name two fields in the `--dry-run=client -o yaml` output that were not in your file. Why does your file still work without them?



## Pods and nodes (kubectl get pods -o wide)
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE
hello-app-66c64958d7-dclvs   1/1     Running   0          10m   10.244.2.3   desktop-worker2
hello-app-66c64958d7-gd8nq   1/1     Running   0          10m   10.244.2.2   desktop-worker2
hello-app-66c64958d7-jdkjc   1/1     Running   0          10m   10.244.1.3   desktop-worker
hello-app-66c64958d7-lvplt   1/1     Running   0          10m   10.244.1.2   desktop-worker

The Pods are spread over the two worker nodes. None run on desktop-control-plane because it is reserved for cluster components.

## Service (kubectl get service hello-app)
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-app   LoadBalancer   10.96.223.87   172.18.0.7    8080:32318/TCP   10m

## EndpointSlice (kubectl get endpointslices -l kubernetes.io/service-name=hello-app)
NAME              ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
hello-app-2l2dn   IPv4          80      10.244.2.2,10.244.1.2,10.244.2.3 + 1 more...   10m

## Trap 1 - local image
- Pods stayed in ErrImagePull / ImagePullBackOff.
- Cause: the cluster nodes are separate machines with their own image storage and cannot see images in my laptop's Docker. hello-app:v1 without a username means the official Docker Hub image hello-app, which does not exist.
- Fix: tagged the image himavarshinivenkatesh/hello-app:v1 and pushed it to Docker Hub.

## Trap 2 - selector typo
- Changed the Service selector to app: helo-app.
- The Service and Pods still existed, but the EndpointSlice was empty and the page showed an error.
- Cause: no Pod has the label app: helo-app, so the Service had nowhere to send traffic.
- Fixed the typo and the page recovered.

\Documents\ms-cloud-devops-lab2> kubectl get endpointslices -l kubernetes.io/service-name=hello-app
NAME              ADDRESSTYPE   PORTS     ENDPOINTS   AGE
hello-app-2l2dn   IPv4          <unset>   <unset>     14m

## Trap 3 - wrong targetPort
- Changed targetPort from 80 to 8080.
- The EndpointSlice listed the Pods, but with port 8080, and the page showed an error.
- Cause: the Service found the Pods but sent traffic to port 8080, where nothing listens. nginx listens on 80.
- Changed it back to 80 and the page recovered.
- An empty EndpointSlice means the selector is wrong. Pods listed on the wrong port means the targetPort is wrong.

\Documents\ms-cloud-devops-lab2> (Get-Content hello-app.yaml -Raw) -replace 'targetPort: 80','targetPort: 8080' | Set-Content hello-app.yaml
\Documents\ms-cloud-devops-lab2> kubectl apply -f hello-app.yaml
deployment.apps/hello-app unchanged
service/hello-app configured
\Documents\ms-cloud-devops-lab2> kubectl get endpointslices -l kubernetes.io/service-name=hello-app
NAME              ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
hello-app-2l2dn   IPv4          8080    10.244.2.2,10.244.1.2,10.244.1.3 + 1 more...   16m
\Documents\ms-cloud-devops-lab2> (Get-Content hello-app.yaml -Raw) -replace 'targetPort: 8080','targetPort: 80' | Set-Content hello-app.yaml
\Documents\ms-cloud-devops-lab2> kubectl apply -f hello-app.yaml
deployment.apps/hello-app unchanged
service/hello-app configured


## Trap 4 - EXTERNAL-IP
- EXTERNAL-IP showed 172.18.0.7, which does not load in the browser.
- It is inside Docker Desktop's private network. I used http://localhost:8080 because Docker Desktop publishes LoadBalancer ports on localhost.

## Dry-run comparison
- The generated YAML has the same core fields as my hello-app.yaml.
- Extra fields: creationTimestamp: null and strategy: {} (both empty or default values).

## Screenshot
- screenshot.png shows 4 different Pod names answering at http://localhost:8080.
