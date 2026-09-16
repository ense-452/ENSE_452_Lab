---
marp: true
theme: default
paginate: true
title: Dev Containers
author: Trevor Douglas
---

# Dev Container Environment Setup & Distribution Guide

This guide explains how to recreate the development container
environment (Ubuntu Devel Tools) and how to
use it.

------------------------------------------------------------------------

## 📁 Folder Structure

    dev-env/
    │
    ├─ .devcontainer/
    │   ├─ devcontainer.json
    │   └─ Dockerfile
    │
    └─ workspace/      # Students create projects here

------------------------------------------------------------------------

## ⚙️ devcontainer.json

``` json
{
  "name": "SSE-ENSE452-2026",
  "dockerFile": "Dockerfile",

  // Mount your host folder into the container
  "workspaceFolder": "/workspace",
  "workspaceMount": "source=${localWorkspaceFolder},target=/workspace,type=bind",

  // VS Code extensions to automatically install inside the container
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools"
      ]
    }
  },
  "capAdd": ["SYS_PTRACE"],
  "securityOpt": ["seccomp=unconfined"]
}


```

------------------------------------------------------------------------

## 🐳 Dockerfile

Use the Dockerfile exactly as created earlier, containing:

-   Ubuntu 24.04 base
-   Build essentials & tools

Place it inside `.devcontainer/Dockerfile`.

------------------------------------------------------------------------

## 🚀 How to Recreate the Container in VSCode

1.  **Install VS Code**

2.  Install the **Dev Containers** extension.

3.  Open the folder `dev-env/` in VS Code.

4.  VS Code will detect the devcontainer and prompt:

    -   **"Reopen in Container"**

5.  If not, use:

    -   `Ctrl+Shift+P` → **Dev Containers: Reopen in Container**

6.  The container builds automatically from the Dockerfile.

------------------------------------------------------------------------

## 🚀 How to Recreate the Container in VSCode


7.  After build:

    -   Open terminal inside VS Code → you are now inside the container.

8.  Verify tools:

        git --version
        
------------------------------------------------------------------------

## 📦 How Students Should Use the Environment

1.  Start Docker

2.  Students open **your folder** in VS Code.

3.  They run **Reopen in Container**.  .... and wait for the build

4.  Inside the container:

        cd /workspace
        
------------------------------------------------------------------------

## 🎉 Done!

Your fully reproducible, consistent dev environment is ready for use.
