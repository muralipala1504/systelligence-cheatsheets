# RHEL 10 Podman Quadlet Cheatsheet
# RHEL 10 Mastery Series — Episode 5
# youtube.com/@SysTelligence

---

## 1. File Locations
```bash
# Rootless (User) containers
~/.config/containers/systemd/

# Rootful (System) containers
/etc/containers/systemd/

# Quadlet Generator (Built-in)
/usr/lib/systemd/system-generators/podman-system-generator

2. Rootless Persistence (Linger)

# Check if persistence is enabled for your user
loginctl show-user $USER | grep Linger

# Enable persistence (survive logout) - requires sudo
sudo loginctl enable-linger $USER

3. Anatomy of a .container File

[Unit]
Description=My App Container
After=network-online.target

[Container]
Image=docker.io/library/nginx:latest
PublishPort=8080:80
Volume=/data/nginx:/usr/share/nginx/html:Z
Environment=APP_ENV=production
AutoUpdate=registry

[Install]
WantedBy=default.target

4. The systemd Magic

# Create the directory and file, then trigger the generator
systemctl --user daemon-reload

# Start the container (Note: Do NOT use 'enable')
systemctl --user start nginx.service

# Check status
systemctl --user status nginx.service

# View the generated unit file
systemctl --user cat nginx.service

5. SELinux and Volumes (The Gotcha)

# If you mount a host volume, you MUST append the :Z flag
# Without :Z, SELinux blocks the container from reading the files.
Volume=/data/nginx:/usr/share/nginx/html:Z

# Check the context after applying
ls -laZ /data/nginx

6. Grouping Containers (.pod Files)

# Create app.pod
[Pod]
PodName=myapp

# In app.container and db.container, add this under [Container]
Pod=app.pod

Containers in the same pod share a network namespace and can reach each other on localhost.

7. Troubleshooting

# View service logs
journalctl --user -u nginx.service -f

# Check if the container crashed
podman ps -a

# View container stdout/stderr
podman logs <container_name>

Questions? Drop them in the YouTube comments — I read every single one.
📺 youtube.com/@SysTelligence

