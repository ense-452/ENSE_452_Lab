# Dev Container Environment Setup & Distribution Guide

This guide explains how to recreate the development container
environment (Ubuntu + Java + Tools) and how to
distribute it to students.

------------------------------------------------------------------------

## 📁 Folder Structure

    dev-env/
    │
    ├─ .devcontainer/
    │   ├─ devcontainer.json
    │   └─ Dockerfile
    │
    └─ workspace/      # Students create Flutter, Java, Node, etc. projects here

------------------------------------------------------------------------

## ⚙️ devcontainer.json

``` json

  "name": "SSE-ENSE370-2026",
  "dockerFile": "Dockerfile",

  // Mount your host folder into the container
  "workspaceFolder": "/workspace",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",

  // VS Code extensions to automatically install inside the container
  "customizations": {
    "vscode": {
      "extensions": [
        "redhat.java",
        "vscjava.vscode-java-pack",
        "ms-vscode.cpptools",
        "ms-python.python"
      ]
    }
  },

  // Run as root inside container (simpler for installs)
  "remoteUser": "root",


```

------------------------------------------------------------------------

## 🐳 Dockerfile

Use the Dockerfile exactly as created earlier, containing:

-   Ubuntu 22.04 base
-   OpenJDK 17
-   Build essentials & tools

Place it inside `.devcontainer/Dockerfile`.

------------------------------------------------------------------------

## 🚀 How to Recreate the Container (Instructor or Students)

1.  **Install VS Code**

2.  Install the **Dev Containers** extension.

3.  Open the folder `dev-env/` in VS Code.

4.  VS Code will detect the devcontainer and prompt:

    -   **"Reopen in Container"**

5.  If not, use:

    -   `Ctrl+Shift+P` → **Dev Containers: Reopen in Container**

6.  The container builds automatically from the Dockerfile.

7.  After build:

    -   Open terminal inside VS Code → you are now inside the container.

8.  Verify tools:

        java -version
        
------------------------------------------------------------------------

## 📦 How Students Should Use the Environment

1.  Students open **your folder** in VS Code.

2.  They run **Reopen in Container**.

3.  Inside the container:

        cd /workspace
        
------------------------------------------------------------------------

## 🎉 Done!

Your fully reproducible, consistent dev environment is ready for use and
distribution.
