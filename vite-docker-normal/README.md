# 🐳 Docker Setup — Step by Step Guide

## 1️⃣ Create a `Dockerfile`

Create a `Dockerfile` in your project root.

### Example (Vite / Node.js)
```Dockerfile
FROM node:20-alpine

# Set working directory inside container
WORKDIR /app

# Copy dependency files first (better caching)
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy remaining source code
COPY . .

# Expose Vite dev server port
EXPOSE 5173

# Start the app
CMD ["npm", "run", "dev"]
````

### Notes

* Copy `package*.json` first → faster rebuilds (Docker caching)
* Use `npm ci` → clean & reproducible installs
* `CMD` runs when container starts

---

## 2️⃣ Build the Docker Image

```bash
docker build -t app-image-name .
```

### Breakdown

* `docker build` → builds image
* `-t app-image-name` → assigns name (tag)
* `.` → current directory (build context)

---

## 3️⃣ Verify the Image

```bash
docker images
```

Shows all:

* built images
* pulled images

---

## 4️⃣ Pre-requisite (Vite specific)

Ensure Vite runs with `--host`

### In `package.json`

```json
{
  "scripts": {
    "dev": "vite --host"
  }
}
```

### Why?

* Default: runs on `localhost` (inside container) ❌
* With `--host`: runs on `0.0.0.0` → accessible from browser ✅

---

## 5️⃣ Run the Container

```bash
docker run -p 5173:5173 app-image-name
```

### Breakdown

* `-p 5173:5173` → maps ports
  `localhost:5173 → container:5173`

---

## 🌐 Access the App

Open:

```
http://localhost:5173
```

---

## ⚡ Optional (Dev Mode with Hot Reload)

```bash
docker run -it -p 5173:5173 -v $(pwd):/app app-image-name
```

### Benefits

* live reload
* no rebuild needed
* faster development

---

## 🚫 Add `.dockerignore` (IMPORTANT)

Create `.dockerignore`:

```
node_modules
.git
.env
dist
```

Prevents:

* large files
* slow builds

---

## 🧭 Summary

```
Dockerfile → docker build → Image → docker run → Container → Browser
```

---

## 💡 Key Concepts

| Term       | Meaning                |
| ---------- | ---------------------- |
| Dockerfile | Blueprint              |
| Image      | Built artifact         |
| Container  | Running instance       |
| `-t`       | Tag (name the image)   |
| `.`        | Build context (folder) |
| `EXPOSE`   | Document port          |
| `-p`       | Port mapping           |

```

