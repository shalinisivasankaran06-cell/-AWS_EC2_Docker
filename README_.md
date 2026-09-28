# Deploy a Python Flask App on AWS EC2 with Docker

A step-by-step guide to running a Flask app on an AWS EC2 instance, packaging it as a Docker image, running it as a container, and publishing it to Docker Hub.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Launch an EC2 Instance](#2-launch-an-ec2-instance)
3. [Configure the Security Group](#3-configure-the-security-group)
4. [Connect to the Instance](#4-connect-to-the-instance)
5. [Create and Run the App Without Docker](#5-create-and-run-the-app-without-docker)
6. [Install Docker](#6-install-docker)
7. [Write the Dockerfile](#7-write-the-dockerfile)
8. [Build the Docker Image](#8-build-the-docker-image)
9. [Create the Container and Verify](#9-create-the-container-and-verify)
10. [Manage the Container (Stop, Start, Status)](#10-manage-the-container-stop-start-status)
11. [Update the App: Rebuild Image and Container](#11-update-the-app-rebuild-image-and-container)
12. [Push the Image to Docker Hub](#12-push-the-image-to-docker-hub)
    - [Docker Hub vs Git/GitHub: how push and pull compare](#docker-hub-vs-gitgithub-how-push-and-pull-compare)
13. [Why Two Different Ports (9000 vs 5000)?](#13-why-two-different-ports-9000-vs-5000)
14. [Why Not Use nohup in the Container CMD?](#14-why-not-use-nohup-in-the-container-cmd)
15. [Troubleshooting](#15-troubleshooting)
16. [Cleanup](#16-cleanup)
17. [Command Cheat Sheet](#17-command-cheat-sheet)

---

## 1. Architecture Overview

```
Browser / curl
      |
      v
EC2 public IP : 9000        <- host port (open in Security Group)
      |
      |  Docker port mapping (-p 9000:5000)
      v
Container : 5000            <- Flask listens here
      |
      v
Flask app (app.py)
```

**Mental model**

| Concept | What it is |
|---|---|
| **Dockerfile** | The recipe (instructions to build the image) |
| **Image** | The packaged app, built from the recipe |
| **Container** | A running instance of the image |
| **Docker Hub** | Online storage for your images |

---

## 2. Launch an EC2 Instance

1. Sign in to the AWS Console and open **EC2 > Instances > Launch instance**.
2. Configure:
   - **Name:** `flask-docker-server`
   - **AMI:** Amazon Linux 2023 (default user is `ec2-user`)
   - **Instance type:** `t2.micro` or `t3.micro` (free tier eligible)
   - **Key pair:** create a new key pair (`.pem` file) and download it. You cannot download it again.
   - **Network settings:** attach a security group (see the next section)
3. Click **Launch instance** and wait until the state is **Running**.
4. Copy the **Public IPv4 address**. It is used in the examples below as `<EC2_PUBLIC_IP>`.

---

## 3. Configure the Security Group

The security group is the instance's firewall. Add these **inbound rules**:

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | **My IP** | Connect to the server |
| Custom TCP | TCP | 9000 | 0.0.0.0/0 | Access the Dockerized app |
| Custom TCP | TCP | 5000 | 0.0.0.0/0 | Only needed for Step 5 (running Flask without Docker) |

> **Security tip:** never open SSH (22) to `0.0.0.0/0`. Restrict it to your own IP.
> To change rules later: **EC2 > Security Groups > select group > Edit inbound rules**.

---

## 4. Connect to the Instance

From your local machine:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

---

## 5. Create and Run the App Without Docker

First confirm the app works on the plain server, so any later problem is clearly a Docker problem.

```bash
mkdir ~/flask-app && cd ~/flask-app
```

**app.py**

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello! Python application is running successfully."

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**requirements.txt**

```
flask
```

Install and run:

```bash
sudo dnf install python3-pip -y
pip3 install -r requirements.txt
python3 app.py
```

Test from another terminal or your browser (port 5000 must be open in the security group):

```bash
curl http://<EC2_PUBLIC_IP>:5000
```

Expected output: `Hello! Python application is running successfully.`

Stop it with `Ctrl + C`.

> **Important:** `host="0.0.0.0"` is required. With the default `127.0.0.1`, the app is only reachable from inside the machine or container, and Docker port mapping will not work.

---

## 6. Install Docker

```bash
sudo dnf install docker -y
sudo systemctl start docker
sudo systemctl enable docker          # start Docker automatically on reboot
sudo usermod -aG docker ec2-user      # run docker without sudo
```

Log out and back in (or run `newgrp docker`), then verify:

```bash
docker --version
docker ps
```

---

## 7. Write the Dockerfile

Create a file named `Dockerfile` (no extension) in the same folder as `app.py` and `requirements.txt`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python3", "app.py"]
```

**What each line does**

| Line | Meaning |
|---|---|
| `FROM python:3.11-slim` | Start from a small official **Python** image |
| `WORKDIR /app` | Set the working folder inside the image |
| `COPY requirements.txt .` | Copy the dependency list first (enables layer caching) |
| `RUN pip install ...` | Install Flask at build time |
| `COPY . .` | Copy the rest of your code |
| `EXPOSE 5000` | Document the port the app listens on |
| `CMD [...]` | The command that runs when the container starts |

Optional `.dockerignore`, to keep the image small and clean:

```
.git
__pycache__
*.pyc
.env
```

---

## 8. Build the Docker Image

```bash
docker build -t appimage .
```

- `-t appimage` gives the image a name (tag).
- `.` means "use the Dockerfile in the current folder".

Verify the image exists:

```bash
docker images
```

Expected output (values will differ):

```
REPOSITORY   TAG      IMAGE ID       CREATED          SIZE
appimage     latest   eb5483035f2c   10 seconds ago   ~150MB
```

---

## 9. Create the Container and Verify

### Create and start the container

```bash
docker run -d -p 9000:5000 --name=appcontainer appimage
```

| Part | Meaning |
|---|---|
| `docker run` | Create and start a container |
| `-d` | Detached mode (run in the background) |
| `-p 9000:5000` | Map **host port 9000** to **container port 5000** |
| `--name=appcontainer` | Name the container |
| `appimage` | The image to create it from |

### Verify it is working

```bash
# 1. Is it running? STATUS should say "Up ..."
docker ps

# 2. Check the application logs
docker logs appcontainer

# 3. Test the app
curl http://<EC2_PUBLIC_IP>:9000
```

Expected response:

```
Hello! Python application is running successfully.
```

Expected logs:

```
 * Serving Flask app 'app'
 * Running on all addresses (0.0.0.0)
 * Running on http://172.17.0.2:5000
```

> The `WARNING: This is a development server` line is normal. For real production traffic, use a production WSGI server such as Gunicorn.
> A `404` for `/favicon.ico` is also normal. Browsers request it automatically.

### Other useful checks

```bash
docker ps -a                                        # all containers, including stopped
docker inspect -f '{{.State.Status}}' appcontainer  # running / exited
docker stats appcontainer                           # live CPU and memory
docker exec -it appcontainer sh                     # open a shell inside (type "exit" to leave)
```

---

## 10. Manage the Container (Stop, Start, Status)

| Task | Command |
|---|---|
| Check status | `docker ps -a --filter "name=appcontainer"` |
| Status word only | `docker inspect -f '{{.State.Status}}' appcontainer` |
| Stop | `docker stop appcontainer` |
| Start again | `docker start appcontainer` |
| Restart | `docker restart appcontainer` |
| View logs | `docker logs appcontainer` |
| Follow logs live | `docker logs -f appcontainer` |
| Delete (must be stopped) | `docker rm appcontainer` |
| Force delete | `docker rm -f appcontainer` |

**`docker start` vs `docker run`**

- `docker start appcontainer` restarts the **existing** container.
- `docker run ...` creates a **new** container. It fails with "name already in use" if `appcontainer` already exists, so run `docker rm appcontainer` first.

**Auto-restart after a crash or server reboot**

```bash
docker run -d -p 9000:5000 --restart unless-stopped --name=appcontainer appimage
```

---

## 11. Update the App: Rebuild Image and Container

A Docker image is a **snapshot**. Changing `app.py` does not change a running container. You must rebuild the image and replace the container.

### Scenario A: You changed the code

```bash
docker stop appcontainer
docker rm appcontainer
docker build -t appimage .
docker run -d -p 9000:5000 --name=appcontainer appimage
docker logs appcontainer
```

### Scenario B: You changed `requirements.txt` or the Dockerfile, or the build looks stale

Force a clean build without cached layers:

```bash
docker stop appcontainer
docker rm appcontainer
docker build --no-cache -t appimage .
docker run -d -p 9000:5000 --name=appcontainer appimage
```

### Scenario C: The container keeps crashing

```bash
docker ps -a                    # look for "Exited (1)"
docker logs appcontainer        # read the error
```

Fix the code or Dockerfile, then follow Scenario A or B.

### Scenario D: You cannot delete an image ("image is being used by container")

Containers, even stopped ones, keep a reference to their image. Remove the container first:

```bash
docker rm appcontainer
docker rmi appimage
```

Or force it: `docker rmi -f appimage`

---

## 12. Push the Image to Docker Hub

1. Create a free account at [hub.docker.com](https://hub.docker.com).
2. (Recommended) Create an access token: **Account Settings > Security > New Access Token**. Use it as your password when logging in.

```bash
docker login
docker tag appimage <your-dockerhub-username>/appimage:latest
docker push <your-dockerhub-username>/appimage:latest
```

Add a version tag so you can roll back later:

```bash
docker tag appimage <your-dockerhub-username>/appimage:v1
docker push <your-dockerhub-username>/appimage:v1
```

### Pull and run it on any other server

```bash
docker pull <your-dockerhub-username>/appimage:latest
docker run -d -p 9000:5000 --name=appcontainer <your-dockerhub-username>/appimage:latest
```

### Pushing an update

After rebuilding the image (Section 11), tag and push again:

```bash
docker tag appimage <your-dockerhub-username>/appimage:latest
docker push <your-dockerhub-username>/appimage:latest
```

### Docker Hub vs Git/GitHub: how push and pull compare

Docker Hub works a lot like GitHub, but for **images** instead of source code.

| Git / GitHub | Docker / Docker Hub |
|---|---|
| GitHub is remote storage for code | Docker Hub is remote storage for images |
| `git push` uploads your work | `docker push` uploads your image |
| `git pull` / `git clone` downloads it | `docker pull` downloads it |
| Branches and tags (`v1.0`) | Image tags (`latest`, `v1`, `v2`) |

**Key differences**

1. **No need to pull before you push.** If you build on the same machine where you changed the code, just build, tag and push. Docker Hub has no merge conflicts, so a push simply replaces what the tag (for example `latest`) points to.
2. **Images are replaced, not edited.** Git keeps a history of file changes. Docker stores full snapshots. After you push a new `latest`, the old image is no longer tagged and is hard to get back, so push version tags (`v1`, `v2`) too.
3. **Pulling is for other machines, and nothing updates automatically.** After you push an update, other servers keep running the old image until you pull and recreate the container on each of them:
   ```bash
   docker pull <your-dockerhub-username>/appimage:latest
   docker stop appcontainer
   docker rm appcontainer
   docker run -d -p 9000:5000 --name=appcontainer <your-dockerhub-username>/appimage:latest
   ```
   A running container never changes by itself. Always recreate it from the new image.
4. **Docker Hub does not track your source code.** Keep `app.py` and the `Dockerfile` in GitHub, and keep the built image in Docker Hub. Most projects use both.

**Typical workflow**

```
Edit code -> git commit + git push          (GitHub: source code)
          -> docker build + docker push     (Docker Hub: image)
          -> on the server: docker pull     -> recreate the container
```

---

## 13. Why Two Different Ports (9000 vs 5000)?

```
-p  HOST_PORT : CONTAINER_PORT
        9000  :  5000
```

| Port | Where it lives | Who decides it |
|---|---|---|
| **5000** | Inside the container | Your app. Flask listens on 5000 (set in `app.py`) |
| **9000** | On the EC2 server | You. Any free port works |

Traffic flow: `Browser -> EC2:9000 -> Docker forwards -> container:5000 -> Flask`

**Rules of thumb**

- The **right side** (5000) must match the port your app actually listens on.
- The **left side** (9000) is the port you use in the browser or `curl`, and it must be open in the Security Group.

**Nothing forces you to use 9000.** `-p 5000:5000` also works. People pick a different host port because:

1. **Avoiding conflicts.** If something on the server already uses 5000, Docker reports "port is already allocated".
2. **Running multiple copies of the same app.** Each needs its own host port, but all can use 5000 inside:
   ```bash
   docker run -d -p 9000:5000 --name=app1 appimage
   docker run -d -p 9001:5000 --name=app2 appimage
   ```
3. **Matching firewall rules.** Only ports opened in the Security Group are reachable from outside.
4. **Using standard ports.** `-p 80:5000` lets users visit the app without typing a port number.

This is also why `curl http://<EC2_PUBLIC_IP>:5000` fails when you ran `-p 9000:5000`. Port 5000 exists only inside the container and is not published to the host.

---

## 14. Why Not Use nohup in the Container CMD?

`nohup` keeps a process alive after you close an SSH session. That problem does not exist in Docker, and using it there causes several issues.

| Problem | Explanation |
|---|---|
| **Not needed** | `docker run -d` already runs the container in the background, independent of your terminal |
| **Container exits immediately with `&`** | A container lives only as long as its main process (PID 1). `CMD nohup python3 app.py &` returns instantly, so Docker thinks the job is done and stops the container |
| **Logs disappear from `docker logs`** | `nohup` redirects output to `nohup.out` or a file. `docker logs` only shows stdout/stderr of the main process, so it will look empty |
| **Signals are not handled properly** | `docker stop` sends SIGTERM to PID 1. Wrapping the app in `sh -c "nohup ..."` means the signal goes to the shell, not to Flask. The app may not shut down cleanly and Docker waits 10 seconds before force-killing it |
| **Hides crashes** | Errors go to a file inside the container, making failures harder to find |

**Recommended**

```dockerfile
CMD ["python3", "app.py"]
```

```bash
docker run -d -p 9000:5000 --name=appcontainer appimage
```

Let Docker handle backgrounding (`-d`), restarts (`--restart unless-stopped`), and logging (`docker logs`).

---

## 15. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'flask'` | Wrong base image (for example `node`), or `pip install` did not run | Use `FROM python:3.11-slim`, ensure `requirements.txt` lists `flask`, and rebuild with `--no-cache` |
| Container exits right away (`Exited (1)`) | Application error | `docker logs appcontainer` |
| `name "/appcontainer" is already in use` | Old container still exists | `docker rm appcontainer` |
| `curl` connection refused or timeout | Port closed or wrong port | Check `docker ps` shows `0.0.0.0:9000->5000/tcp`, and open 9000 in the Security Group |
| Works inside the container but not outside | App bound to `127.0.0.1` | Use `app.run(host="0.0.0.0", port=5000)` |
| `port is already allocated` | Host port in use | Use another host port, e.g. `-p 9001:5000` |
| `unable to remove repository reference ... container is using` | Container still references the image | `docker rm <container>` and then `docker rmi <image>` |
| `permission denied` on the Docker socket | User is not in the `docker` group | `sudo usermod -aG docker ec2-user`, then log out and back in |

**Tip:** `docker inspect appcontainer` shows environment variables, the entrypoint and the port bindings. A `NODE_VERSION` variable in a Python project is a sign of a wrong base image.

---

## 16. Cleanup

```bash
docker stop appcontainer
docker rm appcontainer
docker rmi appimage

# Remove all stopped containers, unused images, networks and build cache
docker system prune -a
```

> These commands are destructive and cannot be undone.

Also remember to **stop or terminate the EC2 instance** in the AWS Console when you are done, to avoid charges.

---

## 17. Command Cheat Sheet

```bash
# Build and run
docker build -t appimage .
docker images
docker run -d -p 9000:5000 --name=appcontainer appimage

# Check
docker ps
docker ps -a
docker logs appcontainer
docker inspect -f '{{.State.Status}}' appcontainer
curl http://<EC2_PUBLIC_IP>:9000

# Lifecycle
docker stop appcontainer
docker start appcontainer
docker restart appcontainer
docker rm appcontainer

# Rebuild after changes
docker stop appcontainer && docker rm appcontainer
docker build -t appimage .
docker run -d -p 9000:5000 --name=appcontainer appimage

# Docker Hub
docker login
docker tag appimage <username>/appimage:latest
docker push <username>/appimage:latest
docker pull <username>/appimage:latest
```

---

## License

MIT
