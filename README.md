# Gitea-Done-Next 🚀

This project provides a complete, self-hosted CI/CD (Continuous Integration/Continuous Deployment) pipeline using Docker Compose. It integrates Gitea, Drone, and Nexus to create a powerful, Git-based workflow for building, testing, and storing software artifacts.

It is designed for developers, small teams, and homelab enthusiasts who want a private, all-in-one solution for source control management and automated builds.

### 🧩 Core Components

*   **🐙 Gitea**: A lightweight, self-hosted Git service. It serves as the source control management system where your code lives.
*   **🚁 Drone CI**: A modern, container-native CI/CD platform. Drone automatically runs your build and test pipelines when you push code to Gitea.
*   **📦 Sonatype Nexus**: A universal artifact repository. It is used to store the Docker images and other artifacts produced by your CI pipeline.
*   **🚦 Traefik**: A modern reverse proxy that provides automatic HTTPS for all services.

## 1. 📋 Prerequisites

Before you begin, ensure you have the following installed:

*   **Docker**: The container runtime used to run the services.
*   **Docker Compose**: The tool used to define and manage the multi-container application.
*   **A valid email address**: For Let's Encrypt SSL certificate generation.
*   **`htpasswd`**: A utility to create the basic authentication credentials for the Traefik dashboard. It's typically included in the `apache2-utils` or `httpd-tools` package.

---

## 2. ⚙️ Project Setup
### Configuration (`.env` file)
Before starting, all configuration is managed in a `.env` file to keep secrets and settings separate from the main configuration.

1.  **Create your environment file:** Copy the example template to create your own local configuration file.
    ```bash
    cp .env.example .env # This file is ignored by Git to protect your secrets.
    ```
2.  **Update `.env`:** Open the new `.env` file and fill in the required values.
    *   `GITEA_ADMIN_USER`: Set this to the username you will create in Gitea. This user will automatically be granted admin rights in Drone.
    *   `GITEA_HOST`: The domain for Gitea (e.g., `gitea.localhost` or `gitea.your.domain`).
    *   `DRONE_HOST`: The domain for Drone (e.g., `drone.localhost` or `drone.your.domain`).
    *   `NEXUS_HOST`: The domain for Nexus (e.g., `nexus.localhost` or `nexus.your.domain`).
    *   `NEXUS_REGISTRY_HOST`: The dedicated domain for the Nexus Docker registry (e.g., `nexus-registry.your.domain`).
    *   `TRAEFIK_HOST`: The domain for the Traefik dashboard (e.g., `traefik.your.domain`).
    *   `DRONE_RPC_SECRET`: A strong, unique secret. You can generate one with `openssl rand -hex 16`.
    *   `LETSENCRYPT_EMAIL`: Your email address, for SSL certificate notifications.
    *   `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`: Set the credentials for the Gitea database.
    *   `TRAEFIK_AUTH`: Basic auth credentials for the Traefik dashboard. Generate with `echo $(htpasswd -nb user password)`.
    *   The `DRONE_GITEA_CLIENT_ID` and `DRONE_GITEA_CLIENT_SECRET` variables will be filled in during the Gitea setup step below.

    *   **(Optional) Image Versions**: You can override the default image versions if needed.
        *   `GITEA_VERSION`
        *   `NEXUS_VERSION`
        *   `DRONE_VERSION`
        *   `DRONE_RUNNER_VERSION`
        *   `TRAEFIK_VERSION`

> **Note:** The `docker-compose.yml` file is pre-configured to read all variables from the `.env` file. You should not need to edit the `docker-compose.yml` file directly.

## 3. 🐙 Gitea Setup (Source Control)
First, we'll create the external network for our reverse proxy and then start all services.
1.  **Create the Proxy Network:** Traefik needs to be on a network that can be shared with other Docker Compose projects if needed.
    ```bash
    docker network create proxy
    ```
2.  **Start all services:**
    ```bash
    docker-compose up -d
    ```
3.  **Access Gitea UI:** Open `https://<your_gitea_host>` (e.g., `https://gitea.your.domain`). It may take a moment for Traefik to provision the SSL certificate.
4.  **Complete Initial Setup:** Follow the on-screen instructions. The database settings will be pre-filled to use the PostgreSQL container. Ensure the **Base URL** is correct (`https://...`) and complete the installation.
5.  **Create Admin User:** Register a new user. This will be your admin account. **Important:** Use the exact same username you set for `GITEA_ADMIN_USER` in your `.env` file. This ensures the user is automatically granted admin privileges in Drone.
6.  **Create OAuth2 Application for Drone:**
    *   Log in and navigate to **Settings** > **Applications**.
    *   Under the "Manage OAuth2 Applications" section, click **Create New Application**.
    *   **Application Name**: `Drone CI`
    *   **Redirect URI**: `https://<your_drone_host>/login` (e.g., `https://drone.your.domain/login`).
6.  **Get Credentials & Update `.env`:** Click **Create Application**. Copy the generated **Client ID** and **Client Secret** and paste them into the `DRONE_GITEA_CLIENT_ID` and `DRONE_GITEA_CLIENT_SECRET` variables in your `.env` file.
7.  **Restart Drone:** Apply the new credentials by restarting the stack.
    > **Note:** If you are updating existing environment variables, you should run `docker-compose up -d --force-recreate` to ensure the services are recreated with the new values.
## 4. 📦 Nexus Setup (Artifact Repository)
Next, we'll configure Nexus to host our private Docker images.
1.  **Access Nexus UI:** Open `https://<your_nexus_host>`. It may take a few minutes for Nexus to start up completely.
    > **Patience is key!** Nexus is a Java application and can take 2-5 minutes to become fully available on the first start. You can monitor its status with `docker-compose logs -f nexus`.
2.  **Retrieve Admin Password:** Run the following command to get the initial admin password:
    ```bash
    docker exec nexus cat /nexus-data/admin.password
    ```
3.  **Log In:** Sign in as `admin` with the retrieved password. Complete the setup wizard, which will prompt you to change your password and configure anonymous access.
4.  **Create Docker Repository:**
    *   Click the **Gear Icon** (Administration).
    *   Navigate to **Repositories** > **Create repository**.
    *   Select the **docker (hosted)** recipe.
    *   **Name**: `docker-private`.
    *   **HTTP Port**: Under "Repository Connectors," check the box for HTTP and enter port `8082`. Traefik will route traffic to this internal port.
    *   Scroll down and click **Create repository**.
5.  **Create a Dedicated CI Role:**
    *   Go to **Administration** > **Security** > **Roles** and click **Create role**.
    *   Select **Nexus role**.
    *   **Role ID**: `ci-docker-role`
    *   **Role Name**: `CI Docker Role`
    *   **Privileges**: Search for and add the `nx-repository-view-docker-docker-private-*` privileges (add, edit, read).
    *   Click **Create role**.
6.  **Create a Dedicated CI User (Best Practice):**
    *   Go to **Administration** > **Security** > **Users** and click **Create local user**.
    *   **ID**: `ci-user`
    *   **First Name**: `CI`
    *   **Last Name**: `User`
    *   **Password**: Create a strong, unique password.
    *   **Roles**: Move `ci-docker-role` from the *Available* box to the *Granted* box.
    *   Click **Create local user**.

Your private Docker registry is now available at `https://<your_nexus_registry_host>`.
> **Security Note:** Using a dedicated user with scoped permissions is significantly more secure than using the `admin` account in your pipeline.

## 5. 🚁 Drone Setup (CI Automation)
Finally, let's connect Drone to Gitea and configure the repository pipeline.
1.  **Access Drone UI:** Open `https://<your_drone_host>` (e.g., `https://drone.your.domain`).
2.  **Authorize Drone:** You will be redirected to Gitea. Click **Authorize Application** to grant Drone access to your account.
3.  **Activate Repository:** Once redirected back to Drone, you will see a list of your Gitea repositories. Find your target project and click **Activate**. Confirm the settings and save.
4.  **Configure Secrets:**
    *   In Drone, navigate to your repository's **Settings** tab.
    *   Go to the **Secrets** section and add the following secrets. These will be used by the pipeline to log in to Nexus.
        *   `NEXUS_USER`: `ci-user` (the dedicated user you just created).
        *   `NEXUS_PASS`: The password you set for the `ci-user`.
        *   `NEXUS_REGISTRY_HOST`: The dedicated domain for your Nexus registry (e.g., `nexus-registry.your.domain`).

## 6. 🚀 The Pipeline: `.drone.yml`
Add a `.drone.yml` file to the root of your Git repository. This file defines the CI/CD pipeline steps.

Here is a sample pipeline that builds a Docker image from a `Dockerfile` and pushes it to your private Nexus registry.

> **Note:** The pipeline below is a comprehensive example that demonstrates a "build-scan-promote" workflow. It validates pull requests, scans for vulnerabilities, and creates versioned releases based on Git tags.

```yaml
kind: pipeline
type: docker # Specifies the runner type
name: default
trigger:
  event:
    - push
    - tag
    - pull_request
 
steps:
  - name: lint
    image: golangci/golangci-lint:v1.59
    commands:
      - echo "Running linter..."
      # - golangci-lint run
  - name: test
    image: golang:1.22
    commands:
      - echo "Running unit tests..."
      # - go test -v ./...
  - name: build_for_pr
    image: plugins/docker
    settings:
      repo: ${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}
      tags: [ "pr-${DRONE_PULL_REQUEST}" ]
      dry_run: true # Build but do not push
      registry: ${NEXUS_REGISTRY_HOST}
      username:
        from_secret: NEXUS_USER
      password:
        from_secret: NEXUS_PASS
    trigger:
      event: [ pull_request ]
  - name: build
    image: plugins/docker
    settings:
      username:
        from_secret: NEXUS_USER
      password:
        from_secret: NEXUS_PASS
      repo: ${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}
      tags: [ "${DRONE_COMMIT_SHA:0:7}" ] # Push a temporary tag for scanning
      cache_from: [ "${NEXUS_REGISTRY_HOST}/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}:latest" ]
      registry: ${NEXUS_REGISTRY_HOST}
    trigger:
      event: [ push, tag ]
      branch: [ main ]
    depends_on: [ test ]
  - name: scan
    image: aquasec/trivy:0.52
    settings:
      input: "${NEXUS_REGISTRY_HOST}/${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}:${DRONE_COMMIT_SHA:0:7}"
      severity: "HIGH,CRITICAL" # Fail on high or critical vulnerabilities
      exit_code: 1
      registry_username:
        from_secret: NEXUS_USER
      registry_password:
        from_secret: NEXUS_PASS
    trigger:
      event: [ push, tag ]
      branch: [ main ]
    depends_on: [ build ]
  - name: promote
    image: plugins/docker
    settings:
      username:
        from_secret: NEXUS_USER
      password:
        from_secret: NEXUS_PASS
      repo: ${DRONE_REPO_OWNER}/${DRONE_REPO_NAME}
      tags: > # Create final tags (e.g., 'latest' or git tag 'v1.0.0')
        {{#if build.tag}}
        [ "${DRONE_COMMIT_SHA:0:7}", "{{build.tag}}" ]
        {{else}}
        [ "${DRONE_COMMIT_SHA:0:7}", "latest" ]
        {{/if}}
      registry: ${NEXUS_REGISTRY_HOST}
    trigger:
      event: [ push, tag ]
      branch: [ main ]
    depends_on: [ scan ]
```

### Testing the Pipeline
1.  **Create a `Dockerfile`:** Add a simple `Dockerfile` to your repository (e.g., `FROM busybox\nCMD ["echo", "hello world"]`).
2.  **Commit and Push:** Commit and push the `.drone.yml` and `Dockerfile` to your Gitea repository.
    ```bash
    git add .drone.yml Dockerfile
    git commit -m "feat: Add CI pipeline for Docker builds"
    git push origin main
    ```
3.  **Verify Build:** Go to your Drone dashboard. The pipeline will trigger automatically, build the image, and push it to Nexus. You can verify the new image tag appears in your `docker-private` repository in the Nexus UI.

> **Security Warning:** The Drone runner is configured to mount the host's Docker socket (`/var/run/docker.sock`). This gives your CI jobs root-level access to the Docker daemon on the host machine. This is a common and convenient setup for personal projects or trusted teams, but it is insecure in a multi-tenant environment. **Only run pipelines from repositories you trust.**

### Accessing the Traefik Dashboard
You can monitor Traefik and see all configured routers by visiting the `TRAEFIK_HOST` you set in your `.env` file (e.g., `https://traefik.your.domain`). You will be prompted for the username and password you configured in the `TRAEFIK_AUTH` variable.

You have now successfully built a complete Git-to-artifact CI/CD pipeline!

---

## Appendix: 🐧 Running with Podman

This project can be run using Podman and `podman-compose` as a daemonless alternative to Docker.

### Prerequisites

1.  **Install Podman and Podman Compose:**
    ```bash
    # Example for Fedora/CentOS
    sudo dnf install podman podman-compose
    ```
2.  **Enable the Podman Socket:** The Drone runner requires the Podman socket to be active.
    ```bash
    # Enable the rootless socket for the current user
    systemctl --user enable --now podman.socket
    ```
3.  **Verify Socket Path:** Check the location of your Podman socket.
    ```bash
    echo $XDG_RUNTIME_DIR/podman/podman.sock
    # Expected output: /run/user/1000/podman/podman.sock (with your user ID)
    ```

### Running the Project

1.  **Export the Socket Path:** Before running `podman-compose`, export the `DOCKER_HOST` environment variable pointing to your Podman socket.
    ```bash
    export DOCKER_HOST="unix://$(podman info --format '{{.Host.RemoteSocket.Path}}')"
    ```
2.  **Run with `podman-compose`:** Use `podman-compose` instead of `docker-compose`. The commands are otherwise identical.
    ```bash
    # Create the external network first
    podman network create proxy
    
    # After setup, start all services
    podman-compose up -d
    ```

Follow the same setup steps (2-5) as outlined above. The web interfaces will be available on the same ports.

> **Note on Rootless Podman:** When running in rootless mode, Podman cannot bind to privileged ports below 1024. The `docker-compose.yml` maps the `traefik` service to ports `80` and `443`. If you encounter permission errors, you must change these port mappings to non-privileged ports (e.g., `"8080:80"` and `"8443:443"`) and update your DNS/client configuration accordingly.
