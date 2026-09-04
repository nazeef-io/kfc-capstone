# KFC Static Website — End-to-End DevOps Pipeline

Deploys a static website using: **Docker → Jenkins → Kubernetes → Ngrok**

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

## 1. Local Docker test
```bash
docker build -t hanif040/kfc-static:latest .
docker run -d -p 8080:80 --name kfc-test hanif040/kfc-static:latest
curl http://localhost:8080
docker stop kfc-test && docker rm kfc-test
```

## 2. Jenkins setup
1. Push this repo to GitHub/GitLab.
2. In Jenkins: **Manage Jenkins → Credentials → Add Credentials**
   - Kind: Username with password
   - ID: `dockerhub-creds`
   - Username/Password: your Docker Hub username + access token
3. Create a new **Pipeline** job → point it to this repo → it will auto-detect the `Jenkinsfile`.
4. Run the build. Stages: Checkout → Build → Verify → Push to Docker Hub.

## 3. Deploy to Kubernetes (Minikube)
```bash
minikube start --driver=docker
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml

kubectl get pods
kubectl get deployment kfc-deployment
kubectl get service kfc-service
```

## 4. Expose via Minikube tunnel (separate terminal, keep running)
```bash
minikube tunnel
```

## 5. Expose externally via Ngrok
```bash
ngrok config add-authtoken <YOUR_TOKEN>
ngrok http 80
```
Open the `https://xxxx.ngrok-free.app` URL from the ngrok output in a browser.

## Validation checklist
- [ ] `docker build` succeeds locally
- [ ] `docker run` + `curl` returns HTML
- [ ] Jenkins pipeline runs all 4 stages green
- [ ] Image visible on Docker Hub (`hanif040/kfc-static:latest`)
- [ ] `kubectl get pods` shows 2/2 Running
- [ ] `kubectl get service kfc-service` shows LoadBalancer with EXTERNAL-IP (after `minikube tunnel`)
- [ ] Ngrok URL loads the website publicly
