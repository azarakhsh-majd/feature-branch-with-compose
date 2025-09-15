# feature-branch-with-compose
1- Git Branching Strategy:

1-1 master → stable production branch

1-2 dev → shared development environment

1-3 feature/* → per-developer branches (e.g. feature/amin, feature/azar)
If we want for python:
# create a new feature branch for Azar
git checkout -b feature/azar
# commit and push changes
git add api/main.py
git commit -m "feat(api): add /hello endpoint for azar"
git push -u origin feature/azar
*** Note : Rule: Nobody commits directly to master
Flow is: feature/* → merge into dev → after review → merge into master

2- API with FastAPI:

 ✅create Directory: api/main.py
 from fastapi import FastAPI
 import os
 app = FastAPI()
 VERSION = "1.0.0"
 BRANCH = os.getenv("TAG", "unknown")
 @app.get("/version")
 def get_version():
     return {
         "app": "myapp-api",
         "version": VERSION,
         "branch": BRANCH
     }
 @app.get("/hello")
 def hello():
     return {"message": f"Hello from API branch: {BRANCH}"}

 ✅ Create: api/requirements.txt
   fastapi==0.115.0
 uvicorn[standard]==0.30.6
 
 ✅ Create api/Dockerfile:
 FROM python:3.11-slim
 WORKDIR /app
 COPY requirements.txt .
 RUN pip install --no-cache-dir -r requirements.txt
 COPY . .
 ENV PORT=8000
 EXPOSE 8000
 CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
3-Create Simple UI (HTML & API)

 ✅ create : ui/index.html:
  <!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>MyApp UI</title>
</head>
<body>
  <h1>Welcome to MyApp UI</h1>
  <div id="info">Loading...</div>
  <script src="/env-config.js"></script>
  <script>
    const apiUrl = window.__API_URL__ || "http://localhost:8000";
    const uiVersion = window.__UI_VERSION__ || "unknown";
    async function loadVersion() {
      const res = await fetch(apiUrl + "/version");
      const data = await res.json();
      document.getElementById("info").innerText =
        `UI v${uiVersion} | API v${data.version} (${data.branch})`;
    }
    loadVersion();
  </script>
</body>
</html>
 ✅ Create ui/env-config.js.template:
 window.__API_URL__ = "${API_URL}";
window.__UI_VERSION__ = "${UI_VERSION}";
 ✅ Create ui/Dockerfile:
 FROM nginx:stable-alpine

COPY index.html /usr/share/nginx/html/index.html
COPY env-config.js.template /usr/share/nginx/html/env-config.js.template

CMD ["/bin/sh", "-c", "envsubst '\$API_URL \$UI_VERSION' < /usr/share/nginx/html/env-config.js.template > /usr/share/nginx/html/env-config.js && nginx -g 'daemon off;'"]

4- Base docker-compose:
docker-compose.yml:
version: "3.9"

services:
  api:
    build: ./api
    image: myapp-api:${TAG:-latest}
    container_name: myapp-api-${TAG:-latest}
    environment:
      - TAG=${TAG:-latest}
    ports:
      - "${API_PORT:-8000}:8000"
  ui:
    build: ./ui
    image: myapp-ui:${TAG:-latest}
    container_name: myapp-ui-${TAG:-latest}
    environment:
      - API_URL=http://localhost:${API_PORT:-8000}
      - UI_VERSION=${TAG:-latest}
    ports:
      - "${UI_PORT:-3000}:80"
      
    5- RUN Master ,Dev , Feature:
    TAG=dev API_PORT=8001 UI_PORT=3001 docker-compose up -d --build
    TAG=master API_PORT=8002 UI_PORT=3002 docker-compose up -d --build
    TAG=feature-azar API_PORT=8003 UI_PORT=3003 docker-compose up -d --build

    6- Test in Browser:
    Dev
      UI → http://localhost:3001
      API → http://localhost:8001/version
    Master
      UI → http://localhost:3002
      API → http://localhost:8002/version
    Feature azar
      UI → http://localhost:3003
      API → http://localhost:8003/version
      









