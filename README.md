# 🌍 Wanderlust – MERN DevOps Deployment

A containerized **MERN (MongoDB, Express.js, React, Node.js)** travel-blog application deployed on **AWS EC2 using Docker**.

This repository documents the application, Dockerization, container networking, MongoDB initialization, environment configuration, and manual Docker deployment workflow **without relying on Docker Compose**.

> **Repository:** https://github.com/iamajaypokharel/Wanderlust-DevOps-MERN-open-source

---

## 📌 Project Overview

**Wanderlust** is a travel-blog application where users can explore travel posts, categories, and related content.

The application is composed of three main services:

* **Frontend:** React + TypeScript + Vite
* **Backend:** Node.js + Express.js + Mongoose
* **Database:** MongoDB 6.0
* **Reverse Proxy:** Nginx configuration is included for routing `/api` requests to the backend and other traffic to the frontend.
* **Deployment Platform:** AWS EC2
* **Container Platform:** Docker
* **Deployment workflow:** Docker CLI and a custom Docker network; Docker Compose is not required for the main deployment workflow.

---

# 🏗️ Architecture

```text
                         Internet
                            │
                            ▼
                     AWS EC2 Public IP
                            │
              ┌─────────────┴─────────────┐
              │                           │
          Port 5173                   Port 5000
              │                           │
              ▼                           ▼
     ┌────────────────┐         ┌────────────────┐
     │ React / Vite   │         │ Node / Express │
     │ mern-frontend  │────────▶│ mern-backend   │
     └────────────────┘         └───────┬────────┘
                                         │
                               mongodb://mongo:27017
                                         │
                                         ▼
                                ┌─────────────────┐
                                │ MongoDB 6.0     │
                                │ mern-mongo      │
                                │ Database:       │
                                │ wanderlust      │
                                └─────────────────┘
```

All application containers communicate through the Docker network:

```text
mern-network
```

The backend connects to MongoDB using:

```text
mongodb://mongo:27017/wanderlust
```

`mongo` is the Docker network hostname/alias for the MongoDB container.

---

# 📁 Repository Structure

```text
Wanderlust-DevOps-MERN-open-source/
│
├── backend/
│   ├── api/
│   ├── config/
│   ├── controllers/
│   ├── data/
│   │   └── sample_posts.json
│   ├── middlewares/
│   ├── models/
│   ├── public/
│   ├── routes/
│   ├── services/
│   ├── tests/
│   ├── .env.sample
│   ├── Dockerfile
│   ├── app.js
│   ├── server.js
│   └── package.json
│
├── frontend/
│   ├── public/
│   ├── src/
│   ├── Dockerfile
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig.prod.json
│
├── Dockerfile.mongo
├── nginx.conf
├── package.json
├── package-lock.json
├── .gitignore
├── LICENSE
├── Documents/
└── README.md
```

---

# 🧰 Technology Stack

| Layer             | Technology                 |
| ----------------- | -------------------------- |
| Frontend          | React 18, TypeScript, Vite |
| Backend           | Node.js 22, Express.js     |
| Database          | MongoDB 6.0                |
| ODM               | Mongoose 8                 |
| Authentication    | JWT                        |
| HTTP Client       | Axios                      |
| Styling           | Tailwind CSS               |
| Validation        | Zod / React Hook Form      |
| Cache Integration | Redis support in backend   |
| Containerization  | Docker                     |
| Reverse Proxy     | Nginx                      |
| Cloud             | AWS EC2                    |
| Source Control    | Git + GitHub               |

---

# 🐳 Dockerization

The project uses separate Docker images for the frontend, backend, and MongoDB.

## 1. Backend Dockerfile

The backend Dockerfile uses `node:22-slim`, installs application dependencies, exposes port `5000`, and starts the server with `npm start`.

Main flow:

```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY . .
RUN npm i

FROM node:22-slim
WORKDIR /app
COPY --from=builder /app .
EXPOSE 5000
CMD ["npm","start"]
```

The backend entry point is:

```text
node server.js
```

---

# 🎨 Frontend Dockerfile

The frontend uses Node.js 23 Alpine and Vite. The current container starts Vite with host binding enabled so the application can be reached through the EC2 published port.

The important command is:

```dockerfile
CMD ["npm", "run", "dev", "--", "--host"]
```

The frontend listens on:

```text
5173
```

---

# 🍃 MongoDB Dockerfile

MongoDB is built from `mongo:6.0` and imports the sample post data from the repository.

```dockerfile
FROM mongo:6.0

COPY backend/data/sample_posts.json /data/sample_posts.json

CMD ["bash", "-c", "mongod --bind_ip_all --fork --logpath /var/log/mongodb.log && sleep 5 && mongoimport --db wanderlust --collection posts --file /data/sample_posts.json --jsonArray && tail -f /var/log/mongodb.log"]
```

## What it does

1. Starts from MongoDB 6.0.
2. Copies `sample_posts.json` into the image.
3. Starts MongoDB with `--bind_ip_all`.
4. Imports the JSON data.
5. Uses the `wanderlust` database and `posts` collection.
6. Keeps the container running.

The `--bind_ip_all` option is important for container-to-container access because the backend connects from a different container.

---

# 🔗 Docker Network

Create the dedicated network:

```bash
docker network create mern-network
```

Verify:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect mern-network
```

The application containers should be connected to this network.

---

# 🍃 Run MongoDB

Build the custom MongoDB image:

```bash
docker build -t wanderlust-devops-mongodb:latest -f Dockerfile.mongo .
```

Create persistent storage:

```bash
docker volume create mongo-data
```

Run MongoDB:

```bash
docker run -d \
  --name mern-mongo \
  --network mern-network \
  --network-alias mongo \
  -v mongo-data:/data/db \
  wanderlust-devops-mongodb:latest
```

### Why `--network-alias mongo`?

The backend uses:

```text
mongodb://mongo:27017/wanderlust
```

so Docker DNS must resolve `mongo` to the MongoDB container.

Remember the distinction:

```text
wanderlust-devops-mongodb:latest  -> image
mern-mongo                        -> container name
mongo                             -> network alias
```

---

# 🚀 Run Backend

Build:

```bash
docker build -t mern-backend:latest ./backend
```

Run:

```bash
docker run -d \
  --name mern-backend \
  --network mern-network \
  -p 5000:5000 \
  -e PORT=5000 \
  -e MONGODB_URI="mongodb://mongo:27017/wanderlust" \
  mern-backend:latest
```

Verify:

```bash
docker ps
```

Logs:

```bash
docker logs mern-backend
```

---

# 💻 Run Frontend

Build:

```bash
docker build -t mern-frontend:latest ./frontend
```

Run:

```bash
docker run -d \
  --name mern-frontend \
  --network mern-network \
  -p 5173:5173 \
  mern-frontend:latest
```

Verify:

```bash
docker ps
```

Expected published port:

```text
0.0.0.0:5173->5173/tcp
```

---

# ☁️ AWS EC2 Deployment

## EC2 requirements

Use an Ubuntu-based EC2 instance with Docker installed.

Clone the repository:

```bash
git clone https://github.com/iamajaypokharel/Wanderlust-DevOps-MERN-open-source.git
cd Wanderlust-DevOps-MERN-open-source
```

Verify Docker:

```bash
docker --version
```

---

# 🔐 AWS Security Group

For a learning/test deployment, the following ports may be opened as needed:

| Port | Purpose          |
| ---: | ---------------- |
|   22 | SSH              |
|   80 | HTTP / Nginx     |
| 5173 | Frontend testing |
| 5000 | Backend testing  |

MongoDB should not be exposed publicly when it is only needed by the backend.

For a production deployment, prefer exposing only HTTP/HTTPS and keep application/database ports private.

---

# 🧪 Testing

## Frontend

Open:

```text
http://YOUR_EC2_PUBLIC_IP:5173
```

## Backend

```bash
curl http://localhost:5000
```

Posts API:

```bash
curl http://localhost:5000/api/posts
```

## MongoDB

Check imported posts:

```bash
docker exec -it mern-mongo mongosh --eval 'db.getSiblingDB("wanderlust").posts.countDocuments()'
```

For the included sample data, this should return:

```text
10
```

Inspect data:

```bash
docker exec -it mern-mongo mongosh --eval 'db.getSiblingDB("wanderlust").posts.find().limit(2).pretty()'
```

---

# 🩺 Troubleshooting

## `getaddrinfo ENOTFOUND mongo`

The backend cannot resolve the MongoDB hostname.

Check:

```bash
docker network inspect mern-network
```

Make sure MongoDB was started with:

```bash
--network-alias mongo
```

## `ECONNREFUSED <mongo-ip>:27017`

MongoDB is reachable by name but is refusing connections. Ensure the MongoDB process listens on the container network interface by using:

```bash
mongod --bind_ip_all
```

Then rebuild/recreate the MongoDB container.

## Backend not reachable on port 5000

Check:

```bash
docker ps
```

The port mapping should include:

```text
0.0.0.0:5000->5000/tcp
```

Also verify the EC2 Security Group.

## Frontend not reachable on port 5173

Check:

```bash
docker ps
```

The port mapping should include:

```text
0.0.0.0:5173->5173/tcp
```

Also ensure Vite is started with `--host`.

---

# 📦 Useful Docker Commands

```bash
docker ps
docker ps -a
docker images
docker network ls
docker network inspect mern-network
docker logs mern-backend
docker logs mern-frontend
docker logs mern-mongo
docker restart mern-backend
docker restart mern-frontend
docker restart mern-mongo
docker inspect mern-backend
```

Follow backend logs:

```bash
docker logs -f mern-backend
```

---

# 🔄 Updating and Redeploying

Pull the latest code:

```bash
git pull
```

Rebuild the backend when backend code changes:

```bash
docker build -t mern-backend:latest ./backend
```

Recreate it:

```bash
docker rm -f mern-backend

docker run -d \
  --name mern-backend \
  --network mern-network \
  -p 5000:5000 \
  -e PORT=5000 \
  -e MONGODB_URI="mongodb://mongo:27017/wanderlust" \
  mern-backend:latest
```

Rebuild the frontend when frontend code changes:

```bash
docker build -t mern-frontend:latest ./frontend
```

Recreate it:

```bash
docker rm -f mern-frontend

docker run -d \
  --name mern-frontend \
  --network mern-network \
  -p 5173:5173 \
  mern-frontend:latest
```

MongoDB does not normally need to be rebuilt for application-code changes.

---

# ⚙️ Environment Variables

The backend uses values including:

```env
PORT=5000
MONGODB_URI=mongodb://mongo:27017/wanderlust
CORS_ORIGIN=http://YOUR_EC2_PUBLIC_IP:5173
FRONTEND_URL=http://YOUR_EC2_PUBLIC_IP:5173
NODE_ENV=Development
JWT_SECRET=<generate-a-new-secret>
```

Do not use `127.0.0.1` for `MONGODB_URI` inside the backend container because `127.0.0.1` points to the backend container itself.

---

# 🌐 CORS

When frontend and backend run on different origins, the backend must allow the frontend origin.

For the EC2 test deployment:

```text
http://YOUR_EC2_PUBLIC_IP:5173
```

In production, use a domain and HTTPS instead.

---

# 🌍 Nginx

The repository also contains an Nginx configuration that is designed to route:

```text
/api/*  -> backend:5000
/*      -> frontend:5173
```

Conceptually:

```text
Browser
   │
   ▼
Nginx :80
   │
   ├── /api/*  ──► backend:5000
   │
   └── /*      ──► frontend:5173
```

The supplied `nginx.conf` references Docker upstream names `frontend` and `backend`, which are natural service names in a Compose setup. For a manual Docker CLI deployment, either create matching network aliases or adjust the upstream names to your actual Docker container names.

---

# 🔒 Security

Never commit real credentials, JWT secrets, database passwords, API keys, or cloud credentials to a public repository.

Recommended `.gitignore` entries:

```gitignore
.env
.env.*
!.env.example
```

Use a safe example file such as:

```text
backend/.env.sample
```

For production, store secrets using AWS Systems Manager Parameter Store, AWS Secrets Manager, CI/CD secret variables, or another secure secret-management system.

Also do not expose MongoDB directly to the public Internet.

---

# 🚀 Recommended Production Improvements

1. Put Nginx in front of the application.
2. Expose only ports 80/443 publicly.
3. Serve a production React build rather than the Vite development server.
4. Use HTTPS and a domain name.
5. Push images to Docker Hub or Amazon ECR.
6. Add GitHub Actions CI/CD.
7. Add health checks and monitoring.
8. Use a managed database such as MongoDB Atlas for production workloads.
9. Move secrets out of the repository and image layers.
10. Add resource limits and log rotation.

---

# 🧠 DevOps Concepts Demonstrated

* Git and GitHub
* Docker image creation
* Docker containers
* Docker volumes
* Docker bridge networks
* Container-to-container DNS
* Network aliases
* Environment variables
* MongoDB containerization
* Multi-container deployment
* AWS EC2
* AWS Security Groups
* Nginx reverse proxy concepts
* API testing with `curl`
* Container troubleshooting
* Application rebuild and redeployment
* Basic production hardening

---

# 📊 Deployment Flow

```text
Developer
   │
   ▼
GitHub Repository
   │
   │ git clone / git pull
   ▼
AWS EC2
   │
   ▼
Docker
   │
   ├───────────────────────┐
   │                       │
   ▼                       ▼
mern-frontend          mern-backend
:5173                  :5000
                            │
                            │ Docker DNS: mongo
                            ▼
                       mern-mongo
                         :27017
                            │
                            ▼
                   wanderlust database
                            │
                            ▼
                     posts collection
```

---

# ✅ Deployment Checklist

* [ ] EC2 instance is running
* [ ] Docker is installed
* [ ] Repository cloned
* [ ] `mern-network` created
* [ ] `mongo-data` volume created
* [ ] MongoDB image built
* [ ] MongoDB container running
* [ ] `mongo` network alias configured
* [ ] Backend image built
* [ ] Backend container running
* [ ] Frontend image built
* [ ] Frontend container running
* [ ] AWS Security Group configured
* [ ] `/api/posts` returns data
* [ ] Frontend loads successfully
* [ ] MongoDB sample data verified
* [ ] Secrets are kept outside the public repository

---

# 📝 License

This project is licensed under the **MIT License**. See `LICENSE` for details.

---

# 👨‍💻 Author

**Ajay Pokharel**

GitHub: https://github.com/iamajaypokharel

Repository: https://github.com/iamajaypokharel/Wanderlust-DevOps-MERN-open-source

---

# ⭐ Support

If this project is useful for learning MERN, Docker, AWS EC2, or DevOps practices, consider starring the repository and contributing improvements.

---

## ⚠️ Security Reminder

Before treating this public repository as production-safe, verify that no real credentials or secrets are committed. Generate a fresh JWT secret and keep real environment values outside Git and Docker image layers.





















































