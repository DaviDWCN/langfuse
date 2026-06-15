# Langfuse Offline Deployment Guide

This guide explains how to deploy Langfuse in a completely offline environment (no internet access) using Docker. We will use a GitHub Action to bundle all necessary Docker images and configuration files into a single downloadable artifact.

## 1. Generate the Offline Bundle (Online Environment)

We have provided a GitHub Action workflow to automatically download and package all required Docker images.

1. Navigate to your repository on GitHub.
2. Go to the **Actions** tab.
3. Select the **Build Offline Docker Bundle** workflow from the left sidebar.
4. Click the **Run workflow** dropdown on the right side.
5. Provide the desired `langfuse_version` (e.g., `3`) and `postgres_version` (e.g., `17`) if you wish to override the defaults.
6. Click the **Run workflow** button.
7. Once the workflow completes successfully (this may take a few minutes as it downloads several GBs of images), open the run details.
8. Scroll down to the **Artifacts** section and download the `langfuse-offline-package.zip`.

## 2. Transfer the Bundle to the Offline Machine

Transfer the downloaded `langfuse-offline-package.zip` to your target offline machine using a USB drive, secure file transfer, or any other approved method for your environment.

## 3. Deploy in the Offline Environment

Ensure that Docker and Docker Compose are installed and running on the target machine.

1. Unzip the package:
   ```bash
   unzip langfuse-offline-package.zip
   cd offline-bundle
   ```

2. Load the Docker images from the tarball:
   ```bash
   docker load -i langfuse-offline-bundle.tar.gz
   ```
   *Note: This will extract and load all the necessary images (`langfuse-web`, `langfuse-worker`, `postgres`, `clickhouse`, `redis`, `minio`) directly into your local Docker daemon without requiring internet access.*

3. Set up your environment variables:
   Copy the provided example environment file to `.env`:
   ```bash
   cp .env.example .env
   ```
   **Important:** Open `.env` and `docker-compose.yml` to replace any `# CHANGEME` placeholder values (like database passwords, encryption keys, and secrets) with secure values.

4. Start the services:
   ```bash
   docker compose up -d
   ```
   Docker Compose will find the required images locally (because you just loaded them) and start the containers without attempting to pull them from the internet.

5. Verify the deployment:
   Open your browser and navigate to `http://localhost:3000` (or the IP address of the machine) to access the Langfuse Web UI.

## Using the MCP Server Offline

Langfuse includes a built-in Model Context Protocol (MCP) server that works entirely offline, just like the rest of the application.

If you are using offline agents or tools (like an offline LLM or local IDE), they can connect to the Langfuse MCP server at:
`http://<your-offline-machine-ip>:3000/api/public/mcp`

You will need to generate API Keys from the local Langfuse UI and encode them as a BasicAuth token as described in the `MCP_QUICKSTART.md` guide.
