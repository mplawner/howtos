```markdown
# HOWTO: Configure Fail2Ban on Immich with Nginx Proxy Manager

## Nginx Proxy Manager

### Step 1: Update `docker-compose.yml`
Add the following volume to your `docker-compose.yml`:
```yaml
  - ./modules:/etc/nginx/modules
```

### Step 2: Restart Nginx Proxy Manager
Restart NPM to create the `modules` folder with the correct permissions.

### Step 3: Create `fail2ban-deny.conf`
```bash
touch ./modules/fail2ban-deny.conf
sudo chown root:root ./modules/fail2ban-deny.conf
sudo chmod 644 ./modules/fail2ban-deny.conf
```

### Step 4: Restart Nginx Proxy Manager
Restart NPM again to load the `fail2ban-deny.conf` into the running Nginx configuration.

## Fail2Ban

### Step 1: Create `nginx-deny.conf`
Create the file `/etc/fail2ban/action.d/nginx-deny.conf` with the following content:
```ini
[Definition]
actionstart =
actionstop =
actioncheck =
actionban = echo "deny <ip>;" >> /path/to/nginx-proxy-manager/modules/fail2ban-deny.conf
             docker compose --project-directory /path/to/nginx-proxy-manager/ restart
             # Optional - /usr/bin/curl -s "https://api.telegram.org/{telegram-token}/sendMessage?chat_id={telegram-channel_id}&text=<name>: Banned <ip>"
actionunban = sed -i "/deny <ip>;/d" /path/to/nginx-proxy-manager/modules/fail2ban-deny.conf
              docker compose --project-directory /path/to/nginx-proxy-manager/ restart
              # Optional - /usr/bin/curl -s "https://api.telegram.org/{telegram-token}/sendMessage?chat_id={telegram-channel_id}&text=<name>: Unbanned <ip>"
```

**Note**: If you want to use Telegram notifications, replace `{telegram-token}` and `{telegram-channel_id}` with actual values (without brackets). You can configure any notification method here; the example uses Telegram.

## Immich

### Step 1: Configure Immich to Use `journald`
Refer to [this discussion](https://github.com/immich-app/immich/discussions/3243#discussioncomment-6681948) for details.

Ensure that Immich is configured to use `journald` by adding the following lines under the `immich-server` section of Immich's `docker-compose.yml`:
```yaml
immich-server:
  logging:
    driver: "journald"
    options:
      tag: "immich-server"
```

### Step 2: Restart Immich
Restart the Immich server.

### Step 3: Verify Logging
Run the following command to check if the logging is set up correctly:
```bash
journalctl -u docker -t immich-server 
```

You should see all the relevant logs. (You may want to set limits on journald if not already done. Modify `/etc/systemd/journald.conf` for this purpose.)

### Step 4: Create Fail2Ban Filter for Immich
Create the file `/etc/fail2ban/filter.d/immich.local` with the following content:
```ini
[Definition]
failregex = immich-server.*Failed login attempt for user.+from ip address\s?<ADDR>
journalmatch = CONTAINER_TAG=immich-server
```

### Step 5: Create Fail2Ban Jail for Immich
Create the file `/etc/fail2ban/jail.d/immich.local` with the following content:
```ini
[immich]
enabled = true
filter = immich
backend = systemd
chain = DOCKER-USER
port = http,https
banaction = %(banaction_allports)s
maxretry = 3
bantime = 14400
findtime = 14400
action = nginx-deny
```

### Step 6: Restart Fail2Ban
Restart the Fail2Ban service.

### Step 7: Testing and Monitoring

- **Watch Jail**: Monitor the jail status using:
  ```bash
  sudo fail2ban-client status immich
  ```

- **Ban/Unban IP**: Manually ban or unban an IP with:
  ```bash
  sudo fail2ban-client set immich banip <IP>
  sudo fail2ban-client set immich unbanip <IP>
  ```

- **Verify Nginx Configuration**: Check the `fail2ban-deny.conf` in the `modules` folder. It should add IPs when banned and remove them when unbanned.
```

This `README.md` is now clean, organized, and ready to guide users through the process of configuring Fail2Ban with Nginx Proxy Manager for Immich.
