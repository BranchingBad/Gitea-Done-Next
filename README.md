## 1. Project Setup: `docker-compose.yml`
This file is the foundation of the project, defining the Gitea, Nexus, and Drone services required for a complete CI/CD pipeline.

First, create a project folder and add the following two files.

#### `docker-compose.yml`
```dockercompose
version: '3.8'
 
networks:
  devops-net:
    driver: bridge

volumes:
  gitea-data:
  nexus-data:
  drone-data:
 
services:
  # 1. Gitea (Git Server)
  gitea:
    image: gitea/gitea:1.21.11 # Pinned version for stability
    container_name: gitea
    volumes:
      - gitea-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"  # Web UI
      - "2222:22"    # SSH
    networks:
      - devops-net
    restart: unless-stopped

  # 2. Nexus (Artifact Repository)
  nexus:
    image: sonatype/nexus3:3.68.1 # Pinned version for stability
    container_name: nexus
    user: "nexus" # <-- IMPROVEMENT: Run as non-root nexus user
    volumes:
      - nexus-data:/nexus-data
    ports:
      - "8081:8081"  # Web UI
      - "8082:8082"  # Docker Registry Port
    environment:
      # <-- IMPROVEMENT: Allocate 1GB RAM to Nexus. Adjust as needed.
      - EXTRA_JAVA_OPTS=-Xms1g -Xmx1g
    networks:
      - devops-net
    restart: unless-stopped

  # 3. Drone (CI Server)
  drone-server:
    image: drone/drone:2.22.0 # Pinned version for stability
    container_name: drone-server
    ports:
      - "80:80"
    volumes:
      - drone-data:/data
    networks:
      - devops-net
    environment:
      # --- Gitea Connection ---
      - DRONE_GITEA_SERVER=http://gitea:3000
      - DRONE_GITEA_CLIENT_ID=${DRONE_GITEA_CLIENT_ID}
      - DRONE_GITEA_CLIENT_SECRET=${DRONE_GITEA_CLIENT_SECRET}
      
      # --- Drone Config ---
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET}
      - DRONE_SERVER_HOST=${DRONE_SERVER_HOST}
      - DRONE_SERVER_PROTO=http
      - DRONE_USER_CREATE=username:${GITEA_USER},admin:true # Auto-make your Gitea user a Drone admin
    restart: unless-stopped
    depends_on:
      - gitea

  # 4. Drone Runner (The "Worker")
  drone-runner:
    image: drone/drone-runner-docker:1.8.3 # Pinned version for stability
    container_name: drone-runner
    volumes:
      - ${DOCKER_HOST:-/var/run/docker.sock}:/var/run/docker.sock # Mount container runtime socket
    networks:
      - devops-net
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-server # Connects to the server above
      - DRONE_RPC_SECRET=${DRONE_RPC_SECRET} # <-- MUST MATCH
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_RUNNER_NAME=docker-runner
    restart: unless-stopped
    depends_on:
      - drone-server
```

### Configuration
Before running `docker-compose up`, update the `environment` variables in `docker-compose.yml`:

*   **`DRONE_RPC_SECRET`**: Change `YOUR_STRONG_SECRET_HERE` to a unique, strong secret (e.g., generated with `openssl rand -hex 16`). This value must be identical for both `drone-server` and `drone-runner`.
*   **`DRONE_USER_CREATE`**: Replace `YOUR_GITEA_USERNAME` with the username you intend to create in Gitea.
*   **`DRONE_SERVER_HOST`**: If you are running this on a remote server, change `localhost` to its public IP address or domain name.
*   **`DRONE_GITEA_CLIENT_ID` & `DRONE_GITEA_CLIENT_SECRET`**: These will be populated in the next step.

## 2. Gitea Setup (Source Control)
First, we'll start Gitea to generate OAuth2 credentials for Drone.

1.  **Start the Gitea service:**
    ```bash
    docker-compose up -d gitea
    ```
2.  **Access Gitea UI:** Open `http://localhost:3000` in your browser.
3.  **Complete Initial Setup:**
    *   **Database Type**: Select **SQLite3** for simplicity.
    *   **Base URL**: Ensure it is `http://localhost:3000/`. This URL is used for user-facing links and OAuth2 redirects and must be accessible from your browser.
4.  **Create Admin User:** Register a new user. Use the same username you set for `YOUR_GITEA_USERNAME` in the `docker-compose.yml` file.
5.  **Create OAuth2 Application:**
    *   Log in and navigate to **Settings** > **Applications**.
    *   Under the "Manage OAuth2 Applications" section, click **Create New Application**.
    *   **Application Name**: `Drone CI`
    *   **Redirect URI**: `http://localhost/login` (or `http://<your_domain>/login` if you changed `DRONE_SERVER_HOST`).
6.  **Get Credentials:** Click **Create Application**. Copy the generated **Client ID** and **Client Secret**.
7.  **Update Docker Compose:** Paste the Client ID and Secret into the `DRONE_GITEA_CLIENT_ID` and `DRONE_GITEA_CLIENT_SECRET` environment variables in your `docker-compose.yml`.

## 3. Nexus Setup (Artifact Registry)
Next, we'll configure Nexus to host our private Docker images and create a dedicated user for the pipeline.

1.  **Start all services:** This will launch Nexus and restart Drone with the new Gitea secrets.
    ```bash
    docker-compose up -d
    ```
2.  **Access Nexus UI:** Open `http://localhost:8081`. It may take a few minutes for Nexus to start up completely.
3.  **Retrieve Admin Password:** Run the following command to get the initial admin password:
    ```bash
    docker exec nexus cat /nexus-data/admin.password
    ```
4.  **Log In:** Sign in as `admin` with the retrieved password. Complete the setup wizard, which will prompt you to change your password and configure anonymous access.
5.  **Create Docker Repository:**
    *   Click the **Gear Icon** (Administration).
    *   Navigate to **Repositories** > **Create repository**.
    *   Select the **docker (hosted)** recipe.
    *   **Name**: `docker-private`.
    *   **HTTP Port**: Under "Repository Connectors," check the box for HTTP and enter port `8082`. This exposes the Docker registry on the port defined in `docker-compose.yml`.
    *   Click **Create repository**.
6.  **Create a Dedicated CI Role:**
    *   Go to **Administration** > **Security** > **Roles** and click **Create role**.
    *   Select **Nexus role**.
    *   **Role ID**: `ci-docker-role`
    *   **Role Name**: `CI Docker Role`
    *   **Privileges**: Search for and add the `nx-repository-view-docker-docker-private-*` privileges (add, edit, read).
    *   Click **Create role**.
7.  **Create a Dedicated CI User (Best Practice):**
    *   Go to **Administration** > **Security** > **Users** and click **Create local user**.
    *   **ID**: `ci-user`
    *   **First Name**: `CI`
    *   **Last Name**: `User`
    *   **Password**: Create a strong, unique password.
    *   **Roles**: Move `ci-docker-role` from the *Available* box to the *Granted* box.
    *   Click **Create local user**.

Your private Docker registry is now available at `localhost:8082`.
> **Security Note:** Using a dedicated user with scoped permissions is significantly more secure than using the `admin` account in your pipeline.

## 4. Drone Setup (CI Automation)
Finally, let's connect Drone to Gitea and configure the repository pipeline.

1.  **Access Drone UI:** Open `http://localhost` (or your `DRONE_SERVER_HOST`).
2.  **Authorize Drone:** You will be redirected to Gitea. Click **Authorize Application** to grant Drone access to your account.
3.  **Activate Repository:** Once redirected back to Drone, you will see a list of your Gitea repositories. Find your target project and click **Activate**. Confirm the settings and save.
4.  **Configure Secrets:**
    *   In Drone, navigate to your repository's **Settings** tab.
    *   Go to the **Secrets** section and add the following secrets. These will be used by the pipeline to log in to Nexus.
        *   `NEXUS_USER`: `ci-user` (the dedicated user you just created).
        *   `NEXUS_PASS`: The password you set for the `ci-user`.

## 5. The Pipeline: `.drone.yml`
Add a `.drone.yml` file to the root of your Git repository. This file defines the CI/CD pipeline steps.

Here is a sample pipeline that builds a Docker image from a `Dockerfile` and pushes it to your private Nexus registry.

```yaml
kind: pipeline
type: docker # Specifies the runner type
name: default

trigger:
  branch:
    - main # Only run this pipeline for commits to the main branch

steps:
  - name: test
    image: golang:1.19 # Or node, python, etc.
    commands:
      - echo "Running unit tests..."
      # - go test -v ./...

  - name: build_and_push
    image: plugins/docker # Use the official Docker plugin for Drone
    depends_on: [ test ] # Only run if the 'test' step succeeds
    settings:
      # Credentials for Nexus (from Drone secrets)
      username:
        from_secret: NEXUS_USER
      password:
        from_secret: NEXUS_PASS
      
      # Build & Push Details
      # Use the service name 'nexus' for inter-container communication
      repo: nexus:8082/docker-private/my-app
      tags:
        - ${DRONE_COMMIT_SHA:0:7} # Tag with the short commit hash
        - latest
      dockerfile: Dockerfile # Assumes a 'Dockerfile' is in your repo
      context: .
      insecure: true # Required for connecting to Nexus via HTTP
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

You have now successfully built a complete Git-to-artifact CI/CD pipeline!

---

## Appendix: Running with Podman

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
    export DOCKER_HOST="unix://${XDG_RUNTIME_DIR}/podman/podman.sock"
    ```
2.  **Run with `podman-compose`:** Use `podman-compose` instead of `docker-compose`. The commands are otherwise identical.
    ```bash
    # Start Gitea first
    podman-compose up -d gitea
    
    # After setup, start all services
    podman-compose up -d
    ```

Follow the same setup steps (2-5) as outlined above. The web interfaces will be available on the same ports.

> **Note on Rootless Podman:** When running in rootless mode, Podman cannot bind to privileged ports below 1024. The `docker-compose.yml` maps Drone to port `80`. If you encounter permission errors, change the port mapping for `drone-server` from `"80:80"` to a non-privileged port like `"8080:80"`.
