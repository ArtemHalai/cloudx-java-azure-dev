# Fixing Application Error

> ⚠️ **Disclaimer**: Azure evolves faster than your coffee gets cold ☕ — so if the logs look different, UI buttons moved, or something works "differently today", it’s not broken — it’s just Azure being Azure. You're engineers, not screenshot-following robots. So when the portal says "Application Error", don’t panic — check the docs, search like a ninja, or ask ChatGPT (it’s cheaper than support 😉).

<hr>

## The problem

After deploying a Web App using a Docker container from Azure Container Registry (ACR), the application shows a black screen with the message:

  <img src="images/application-error.png" width="600" style="border: 1px solid #ccc; margin: 20px 0; box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); display: inline-block;"/>

This usually means the container was pulled successfully, but failed to start correctly on Azure App Service.

---

## Root causes

1. Missing environment variable `WEBSITES_PORT=8080`.
2. The container image is not available in ACR or not tagged correctly.
3. App Service failed to pull the container image due to outdated credentials.
4. The container starts, but crashes at runtime (e.g., port issue, Spring Boot error).
5. (If using Apple) The image was built for `linux/arm64`, but App Service supports only `linux/amd64`.

---

## Solution steps

### Step 1: Set `WEBSITES_PORT`

Azure needs to know which internal port your container listens on.

1. Go to your **App Service** in the Azure portal.
2. Navigate to **Configuration** → **Application settings**.
3. Add the setting:
   ```
   Name: WEBSITES_PORT
   Value: 8080
   ```
4. Click **Save**.
5. Restart the App Service from the **Overview** tab.

---

### Step 2: Verify the Docker image in ACR

1. Open your **Azure Container Registry** in the portal.
2. Go to **Repositories**.
3. Make sure your image (e.g. `petstoreapp`) appears in the list.
4. Confirm the correct tag is used (e.g. `latest`).

If the image is missing or mistagged, push it again:

```bash
docker build -t <acr-name>.azurecr.io/petstoreapp:latest .
docker push <acr-name>.azurecr.io/petstoreapp:latest
```

---

### Step 3: Refresh the image in Deployment Center

Sometimes App Service doesn’t refresh credentials after an image change.

1. Open your **App Service**.
2. Go to **Deployment Center**.
3. Re-select the container image and tag.
4. Click **Save** and **Restart** the app.

---

### Step 4: Check container logs

1. Go to your App Service → **Log Stream**.
2. Wait for the app to restart and watch the logs in real time.
3. Look for error messages such as:
    - Image pull failed
    - Port not found or exposed
    - Spring Boot startup errors

This will help identify what went wrong inside the container.

---

### Step 5: Test the image locally

If you're unsure whether your image works at all, test it locally:

```bash
docker pull <acr-name>.azurecr.io/petstoreapp:latest
docker run -p 8080:8080 <acr-name>.azurecr.io/petstoreapp:latest
```

Then open `http://localhost:8080` in your browser.  
If the container fails locally too, the issue is inside your application or Dockerfile.

---

### Step 6: (Apple M1/M2 users only) Build for AMD64, not ARM64

Azure App Service currently does **not support ARM64 images**. If you’re using a Mac with an M1 or M2 chip, Docker will build images for `linux/arm64` by default — and those won’t work in App Service.

To fix this:

Build your image explicitly for `linux/amd64` using Docker Buildx:

```bash
docker buildx build --platform linux/amd64 -t <acr-name>.azurecr.io/petstoreapp:latest --push .
```

Alternatively, if you're using classic `docker build`, add:

```bash
docker build --platform linux/amd64 -t <acr-name>.azurecr.io/petstoreapp:latest .
```

> This forces Docker to build an image compatible with Azure App Service.

If you want to support both platforms (for local testing and deployment), you can build a multi-architecture image:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t <acr-name>.azurecr.io/petstoreapp:latest --push .
```

> For a complete guide on building cross-platform images (including multi-arch builds and Dockerfile examples), check out: [Working with ARM64 on Mac for Azure App Service](./working-with-arm64-on-mac.md)
