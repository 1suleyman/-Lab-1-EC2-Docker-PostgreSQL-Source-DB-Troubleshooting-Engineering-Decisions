# 🧠 Troubleshooting & Engineering Decisions

**Lab 1 — EC2 + Docker PostgreSQL (Source DB)**

This section documents the **real-world issues encountered during the lab**, how they were **diagnosed**, and the **engineering decisions used to resolve them**.
The goal is to demonstrate **practical DevOps problem-solving**, not just successful execution.

> In production environments, most work involves debugging misconfigurations, tooling gaps, and unexpected behavior. This lab intentionally captures that reality.

---

## 1️⃣ SSH Key Permissions & Secure Access

### Issue

SSH refused to use the EC2 private key due to overly permissive file permissions.

### Diagnosis

Linux SSH enforces strict permissions on private keys.
Keys readable by group or others are considered insecure.

### Resolution

Restricted the key to **owner-read only**:

```bash
chmod 400 labec2.pem
ls -la labec2.pem
```

### Engineering Insight

Security tooling often fails *loudly by design*.
Understanding *why* permissions matter prevents unsafe workarounds.

---

## 2️⃣ Docker Compose v2 Not Recognized

### Issue

Running `docker compose up -d` resulted in:

```
unknown shorthand flag: 'd' in -d
```

### Diagnosis

Docker was installed, but the **Docker Compose v2 CLI plugin** was missing.
Without the plugin, Docker does not recognize the `compose` subcommand.

### Resolution

Installed Docker Compose v2 as a CLI plugin:

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins

sudo curl -SL \
  https://github.com/docker/compose/releases/download/v5.0.1/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

Verified installation:

```bash
docker compose version
```

### Engineering Insight

Modern Docker relies heavily on **plugin-based architecture**.
Missing plugins can appear as syntax or flag errors rather than explicit install errors.

---

## 3️⃣ Linux `curl` Line Continuation Formatting Error

### Issue

The Docker Compose plugin download initially failed with a “file not found” / path error.

### Diagnosis

The backslash (`\`) used for line continuation **must be the final character on the line**.
Any trailing space causes the shell to misinterpret the command.

### Resolution

Corrected the command formatting so the backslash was the last character.

### Engineering Insight

Shell errors are often **syntactically valid but semantically wrong**.
Attention to formatting is critical when scripting or automating installs.

---

## 4️⃣ Docker Compose YAML Validation Error (Volumes)

### Issue

Docker Compose failed with:

```
services.volumes additional properties 'pgdata' not allowed
```

### Diagnosis

The named volume `pgdata` was declared **inside `services`**, which violates the Docker Compose schema.

### Resolution

Moved the volume declaration to the **root level** of the YAML file:

```yaml
services:
  postgres_db:
    ...
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### Engineering Insight

YAML is indentation-sensitive and schema-validated.
Many Docker issues are **structure errors**, not runtime failures.

---

## 5️⃣ Docker Group Permissions Not Applied Immediately

### Issue

After adding `ec2-user` to the `docker` group, Docker commands still required `sudo`.

### Diagnosis

Linux group membership changes do not apply to existing sessions.

### Resolution

Reloaded group membership without logging out:

```bash
newgrp docker
```

Verified with:

```bash
docker ps
```

### Engineering Insight

Permission changes often require **session reloads**, service restarts, or re-authentication.
Knowing when changes take effect avoids unnecessary troubleshooting.

---

## 6️⃣ PostgreSQL Container & Data Persistence Validation

### Issue

Needed to ensure PostgreSQL data would survive container restarts.

### Diagnosis

Without a named volume, PostgreSQL data would be ephemeral.

### Resolution

Used a named Docker volume (`pgdata`) mapped to PostgreSQL’s data directory.

Verified by:

* Restarting the container
* Confirming tables and data still existed

### Engineering Insight

Data persistence is a **design decision**, not a default behavior in containers.
Explicit volume management is essential for stateful workloads.

---

## 7️⃣ Database Access & Role Verification

### Issue

Required confirmation that the database behaved like a realistic migration source.

### Resolution

Connected directly into the running container:

```bash
docker exec -it postgres_db psql -U admin -d appdb
```

Created:

* Tables
* Sample data
* Read-only and writer roles

### Engineering Insight

Testing from *inside* the runtime environment eliminates network and client-side variables.

---

### 8️⃣ EC2 Instance Connect Fails While SSH Client Works

**Issue**
EC2 Instance Connect (browser-based SSH) failed, while connecting via a local SSH client using a key pair worked successfully.

**Diagnosis**
Initial checks confirmed:

* Instance had a public IPv4 address
* Correct OS and username (`ec2-user`)
* EC2 Instance Connect package was installed:

  ```bash
  rpm -qa | grep ec2-instance-connect
  ```

A controlled test showed:

* SSH allowed from **My IP (/32)** → SSH client ✅ works, EC2 Instance Connect ❌ fails
* SSH allowed from **Anywhere (0.0.0.0/0)** → EC2 Instance Connect ✅ works

This demonstrated that EC2 Instance Connect traffic originates from **AWS-managed IP ranges**, not the user’s local IP.

**Resolution**
Temporarily opened SSH to `0.0.0.0/0` to validate the hypothesis, confirming EC2 Instance Connect functionality.
SSH access was then locked back down to the user’s IP.

**Engineering Insight**
EC2 Instance Connect is not “just SSH in the browser.”
It introduces additional network paths and source IP considerations. In production, teams typically use:

* EC2 Instance Connect Endpoints
* SSM Session Manager
* Corporate NAT / VPN IP ranges

Browser-based tooling can add hidden dependencies that do not exist in native workflows.

---

## ✅ Outcome

By the end of this troubleshooting process:

* EC2 access was secure and reproducible
* SSH behavior and permission models were fully understood
* Docker and Docker Compose were correctly installed
* PostgreSQL ran reliably with persistent storage
* YAML and CLI errors were resolved methodically
* The database behaved like a realistic production source system

This environment is now **fully prepared for migration to Aurora PostgreSQL in subsequent labs**.
