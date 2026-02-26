
# 🚀 Cisco FMC MCP Server: Docker Implementation Guide

This guide covers the specialized "Docker STDIO" implementation of the Cisco FMC MCP server. This method is preferred for visibility, as it allows you to monitor server logs directly via Docker Desktop without cluttering your local Python environment.

## 📋 Prerequisites

* **Docker Desktop:** Must be installed and running.
* **FMC Access:** You need the IP/Hostname, Username, and Password for your Firepower Management Center.
* **Cursor IDE:** Configured for MCP.

---

## 🛠 Step 1: Clone and Organize

We rename the repository during cloning to keep our MCP directory clean and easy to navigate.

```bash
cd ~/MCP
git clone https://github.com/CiscoDevNet/CiscoFMC-MCP-server-community.git fmc-mcp
cd fmc-mcp

```

> **Note:** Adding `fmc-mcp` at the end of the git command renames the folder from the long default name to a short, manageable one.

---

## 🔐 Step 2: Configure Credentials

The server reads FMC details from a hidden `.env` file.

1. Create the file from the template:
```bash
cp .env.example .env

```


2. Edit the `.env` file with your details:
* `FMC_HOST`: Your FMC IP.
* `FMC_USER`: Admin username.
* `FMC_PASSWORD`: Admin password.
* `SSL_VERIFY`: Set to `False` if using self-signed certificates.



---

## 📦 Step 3: Build the Docker Image

Since the `Dockerfile` in this repo is stored in a subfolder, we must point Docker to it while keeping the "context" in the root directory so it can see your `.env` and Python files.

```bash
docker build -t fmc-mcp-server -f docker/Dockerfile .

```

* **`-t fmc-mcp-server`**: Names our image for easy reference.
* **`-f docker/Dockerfile`**: Tells Docker the instructions are in the `docker` subfolder.
* **`.`**: Tells Docker the source code is in the current directory.

---

## 💻 Step 4: Add to Cursor (mcp.json)

This implementation uses **STDIO transport**. Instead of a URL, Cursor runs a Docker command that pipes data in and out of the container.

Add this to your `mcp.json` (Found in Cursor Settings > MCP):

```json
"fmc-mcp": {
  "command": "docker",
  "args": [
    "run",
    "-i",
    "--rm",
    "--name", "fmc-mcp-container",
    "--env-file", "/Users/amitsi4/MCP/fmc-mcp/.env",
    "fmc-mcp-server"
  ]
}

```

### **Argument Breakdown:**

* **`run -i`**: Starts the container in **Interactive** mode, allowing Cursor to "talk" to the Python process inside.
* **`--rm`**: Automatically deletes the container when Cursor stops the server, saving disk space.
* **`--name`**: Gives the container a static name so you can find it easily in the Docker Desktop dashboard.
* **`--env-file`**: Points to the **Absolute Path** of your credentials. Relative paths (`./.env`) will fail in Cursor.

---

## 🔍 Step 5: Visibility and Logs

One of the main benefits of this Docker setup is visibility:

1. Open **Docker Desktop**.
2. Locate **`fmc-mcp-container`** in the "Containers" tab.
3. Click on the name to view **Live Logs**. You will see every search and policy list request as Cursor makes them.

---

## 💾 Step 6: Backup to Personal GitHub

To save your configuration (without leaking your passwords):

1. Ensure `.env` is ignored: `echo ".env" >> .gitignore`
2. Create a new repo on GitHub (e.g., `fmc-mcp-custom`).
3. Push your code:
```bash
git remote add origin-custom https://github.com/amitsi4/fmc-mcp-custom.git
git branch -M main
git push -u origin-custom main

```



---

**Would you like me to help you draft a first test query to run in Cursor to verify the `find_rules_by_ip_or_fqdn` tool?**
