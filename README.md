# Day 9: Dockerfile Deep Dive

Node.js API in a Docker image, configured at runtime.
Repo: https://github.com/YOUR-USERNAME/YOUR-REPO

## Run

```bash
docker build --progress=plain -t demo-api:v1 --build-arg APP_VERSION=1.0 .
docker run -d --name api -p 3000:3000 demo-api:v1
curl http://localhost:3000/          # Hello from Docker | env=dev
curl http://localhost:3000/health    # ok
```

## Dockerfile explained

- `FROM node:20-alpine`: small base image with Node.js 20
- `WORKDIR /app`: working folder inside the image
- `ARG APP_VERSION=1.0`: build-time value only
- `ENV PORT=3000 APP_ENV=dev ...`: default runtime settings
- `COPY package*.json ./`: copy dependency files first (for caching)
- `RUN npm install --omit=dev`: install production dependencies
- `COPY . .`: copy the source code
- `EXPOSE 3000`: documents the port, does not publish it
- `USER node`: run as non-root
- `CMD ["node","server.js"]`: default start command (exec form)

`.dockerignore` keeps `.git`, `.env`, `node_modules` and logs out of the build.

## Runtime config

```bash
docker rm -f api
docker run -d --name api -p 3000:3000 -e GREETING="Hi Winter Arc" -e APP_ENV=prod demo-api:v1
curl http://localhost:3000/          # Hi Winter Arc | env=prod
```

## Cache experiment

- v1: first build, everything runs
- v2: edited only `server.js`, `npm install` was **CACHED**
- v3: edited `package.json`, `npm install` **ran again**

Why: `package*.json` is copied and installed before the source code, so a
source-only change does not invalidate the dependency layer.

## CMD vs ENTRYPOINT

```bash
docker build -f Dockerfile.greeter -t greeter:local .
docker run --rm greeter:local          # Hello World
docker run --rm greeter:local DevOps   # Hello DevOps
```

`CMD` was overridden. It only gives default arguments, while `ENTRYPOINT` stays fixed.

## Evidence

![health](evidence/health-check.png)
![env](evidence/env-override.png)
![cache](evidence/cache-hit.png)
![entrypoint](evidence/entrypoint-test.png)

## Reflection

- **Reused step after editing only server.js:** the `npm install` layer (CACHED).
- **What changed when package.json changed:** `COPY package*.json` changed, so `npm install` and later steps re-ran.
- **Build time vs runtime:** ARG is set at `docker build`; ENV defaults can be overridden with `-e` at `docker run`, no rebuild.
- **Before production:** use `npm ci` with a lock file, pin the base image digest, scan for vulnerabilities, add a HEALTHCHECK, use a secrets manager.
