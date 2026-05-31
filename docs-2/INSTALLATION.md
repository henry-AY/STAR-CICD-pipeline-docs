# Installation Guide

### Step 1: Binary Placement
Place the `act_runner` binary into a dedicated service directory under an administrative or CI/CD user profile on your machine.
* Target Path: `/home/cicd/gitea-runner/act_runner`
* Configuration Path: `/home/cicd/gitea-runner/config.yaml`

### Step 2: Systemd Daemon Configuration
To ensure the runner survives system crashes and reboots, it must not be run manually in a user shell. Instead, it must be managed via a system supervisor. Create the following unit file:

```bash
sudo nano /etc/systemd/system/gitea-runner.service
```

#### Configuration
```Ini
[Unit]
Description=Gitea Actions Runner (G-STAR Engine)
After=network.target

[Service]
Type=simple
User=cicd
WorkingDirectory=/home/cicd/gitea-runner
ExecStart=/home/cicd/gitea-runner/act_runner daemon -c /home/cicd/gitea-runner/config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### Step 3: Activating the Daemon
Reload the system supervisor, enable the unit file for startup boot-triggering, and initialize the service:

```bash
# Force systemd to register the new unit file
sudo systemctl daemon-reload

# Enable the service to run automatically on boot
sudo systemctl enable gitea-runner.service

# Start the service immediately
sudo systemctl start gitea-runner.service
```

To confirm the service is operational, execute the status command and verify the status reflects `active (running)` in green text:

```bash
sudo systemctl status gitea-runner.service
```



