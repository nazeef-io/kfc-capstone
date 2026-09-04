# KFC Static Website — End-to-End DevOps Pipeline

Deploys a static website using: **Docker → Jenkins → Docker Hub → Kubernetes → Minikube Tunnel → Ngrok**

<img width="1920" height="1200" alt="Screenshot From 2026-09-04 14-28-28" src="https://github.com/user-attachments/assets/d26ebbdd-63d5-46a4-90bf-debc50b42de0" />

## Structure
```
kfc-capstone/
├── website/           # Static site (HTML/CSS)
│   ├── index.html
│   └── css/style.css
├── Dockerfile          # Builds nginx-based image serving the site
├── Jenkinsfile         # CI/CD pipeline: build → verify → push to Docker Hub
├── k8s/
│   ├── deployment.yaml # K8s Deployment (2 replicas)
│   └── service.yaml    # K8s LoadBalancer Service on port 80
└── README.md
```

**Docker Hub image:** `nazeefkhan228/kfc-capstone-dockerhub-repo:latest`

## 1. Local Docker test
```bash
docker build -t nazeefkhan228/kfc-capstone-dockerhub-repo:latest .
docker run -d -p 8080:80 --name kfc-test nazeefkhan228/kfc-capstone-dockerhub-repo:latest
curl http://localhost:8080
docker stop kfc-test && docker rm kfc-test
```

## 2. Jenkins setup
1. Push this repo to GitHub (use a Personal Access Token as the password, not your GitHub account password — GitHub no longer accepts account passwords over HTTPS).
2. In Jenkins: **Manage Jenkins → Credentials → Add Credentials**
   - Kind: Username with password
   - ID: `dockerhub-creds`
   - Username/Password: your Docker Hub username + access token
3. Create a new **Pipeline** job → point it to this repo → it will auto-detect the `Jenkinsfile`.
4. Run the build. Stages: Checkout → Build → Verify → Push to Docker Hub.

## 3. Deploy to Kubernetes (Minikube)
```bash
minikube status          # confirm it's Running before applying anything
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods
kubectl get deployment kfc-deployment
kubectl get service kfc-service
```

## 4. Expose via Minikube tunnel

Run this in its own terminal/session and **leave it running** — closing it (or losing SSH connection to that session) kills the route:

```bash
minikube tunnel
```

Recommended for remote VMs (survives SSH disconnects) — run it in `tmux` instead:
```bash
tmux new -s tunnel
minikube tunnel
# detach without killing it: Ctrl+B then D
# reattach later: tmux attach -t tunnel
```

Then check the Service again:
```bash
kubectl get service kfc-service
```

> **Important:** `EXTERNAL-IP` will be an IP like `10.101.183.17` — **not** `127.0.0.1`/`localhost`. `minikube tunnel` routes to the Service's actual assigned IP, it does not bind to localhost. Always copy the real `EXTERNAL-IP` value before testing.

Test using that real IP:
```bash
curl http://<EXTERNAL-IP>:80
```

## 5. Expose externally via Ngrok
```bash
ngrok config add-authtoken <YOUR_TOKEN>
ngrok http <EXTERNAL-IP>:80
```

Ngrok will output something like:
```
Forwarding   https://xxxx.ngrok-free.app -> http://<EXTERNAL-IP>:80
```

Open that `https://...ngrok-free.app` URL in a browser — this is your live, publicly accessible site.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name/tag, or image not public yet | `docker pull <image>` locally to confirm it exists and is public; run `kubectl describe pod <name>` and check Events |
| `dial tcp ...:8443: connect: no route to host` on `kubectl apply` | Minikube stopped or IP changed | `minikube status` → `minikube start --driver=docker` if stopped |
| `EXTERNAL-IP` stuck on `<pending>` | `minikube tunnel` not running | Start it in a separate terminal/tmux session and leave it running |
| `502 Bad Gateway` from ngrok | Ngrok pointed at `localhost:80` instead of the Service's real `EXTERNAL-IP` | `kubectl get service kfc-service`, use that IP in `ngrok http <IP>:80` |
| `curl http://localhost:80` → Connection refused | Same as above — nothing binds to localhost with `minikube tunnel` | Always test against the real `EXTERNAL-IP`, not localhost |

## Validation checklist
- [ ] `docker build` succeeds locally
- [ ] `docker run` + `curl` returns HTML
- [ ] Jenkins pipeline runs all 4 stages green
- [ ] Image visible on Docker Hub (`nazeefkhan228/kfc-capstone-dockerhub-repo:latest`)
- [ ] `kubectl get pods` shows 2/2 Running
- [ ] `kubectl get service kfc-service` shows LoadBalancer with a real EXTERNAL-IP (after `minikube tunnel`)
- [ ] `curl http://<EXTERNAL-IP>:80` returns the site's HTML
- [ ] Ngrok forwarding to `<EXTERNAL-IP>:80` shows no errors
- [ ] Browser screenshot of the site loading via the ngrok public URL
