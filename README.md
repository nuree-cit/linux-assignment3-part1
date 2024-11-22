# Linux Assignment 3 Part 1
This assignment will help you to setup a Bash script that generates a static run automatically every day at 05:00 using a index.html file that contains some system information. 

It includes following files:
- `generate_index`
- `generate_index.service`
- `generate_index.timer`
- `nginx.conf`
- `webgen.conf`

In the next steps, you will be able to move and run those files properly with the instructions.

## Prerequisites
To successfully run the scripts, users are required:

- An Arch Linux OS system with Bash shell.
- A user account with permission to access to root through `sudo` privilege.
- Have installed packages:
	- git
	- nvim
	- nginx
	- ufw
- And finally, download files from this repository!
## Instructions

### 1. Login to your user account.
```bash
ssh USERNAME
```

### 2. Clone the repository.
```bash
git clone https://github.com/nuree-cit/linux-assignment3-part1
```
- It will generate `linux-assignment3-part1` directory and given files.

### 3. Create a system user `webgen`
Create with a home directory at `/var/lib/webgen` and a login shell appropriate for a non-login user.
```bash
sudo useradd -r -m -d /var/lib/webgen -s /usr/sbin/nologin webgen
```
`-r`: create a system user
`-m`: create a home directory
`-d`: define a path of home directory
`-s`: specify shell, `nologin` specifies that is non-login user

By creating a system user and blocking login, the service can be isolated, which improves security.

### 4. Create a `bin` and a `HTML` directory in the `/var/lib/webgen`
```bash
sudo mkdir /var/lib/webgen/bin /var/lib/webgen/HTML
```

### 5. move `generate_index` to the `/var/lib/webgen/bin` 

```bash
sudo mv ./linux-assignment3-part1/generate_index /var/lib/webgen/bin
```

### 6. change the permission of `generate_index`
```bash
sudo chmod +x /var/lib/webgen/bin/generate_index
```
- This prevents issues with permission being denied when running the script as a different user.

### 7. Run the script `generate_index`
```bash
sudo /var/lib/webgen/bin/generate_index
```

now, if you check the file tree,
```bash
sudo tree /var/lib/webgen
```

you will see the output like:
```bash
/var/lib/webgen/
├── bin
│   └── generate_index
└── HTML
    └── index.html
```
### 8. Change ownership to user `webgen`
```bash
sudo chown -R webgen:webgen /var/lib/webgen/
```
- `-R`: recursively change ownership.

### 9. Move a service file `generate-index.service`
This `generate-index.service` file will run the `generate_index` script.
```bash
sudo mv ./linux-assignment3-part1/generate_index.service /etc/systemd/system/generate_index.service
```

`generate-index.service` file has contents like:
```bash
[Unit]
Description=Run generate_index script
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/var/lib/webgen/bin/generate_index
User=webgen
Group=webgen

[Install]
WantedBy=multi-user.target
```
- `Wants=`: it will request to run `network-online.target`, but not force to run it.
- `After=`: it will run this service after `network-online.target` has started.
- `ExecStart=`: what script it will run.
- `User=`: it will run the script as a user `webgen`.
- `Group=`: it will run the script as a group `webgen`.

### 10. Enable and start `generate-index.service` 
```bash
sudo systemctl daemon-reload
```
- reload service after brought a new service file.

```bash
sudo systemctl enable --now generate_index.service
```
- `--now`: it will enable and start the service file at the same time. 

To check the status of `generate_index.service`,
```bash
sudo systemctl status generate_index.service
```
- This will print information about status of the service file.

To check the log of the `generate_index.service`
```bash
sudo journalctl -u generate_index.service
```
- This will print logs related to the service unit. You can check here to see if the service has triggered as expected.
### 11. Move a timer file `generate-index.timer` 
This `generate-index.timer` file will trigger the `generate_index.service` file at the set time.
```bash
sudo mv ./linux-assignment3-part1/generate_index.timer /etc/systemd/system/generate_index.timer
```

`generate-index.timer` file has contents like:
```bash
[Unit]
Description=Run generate_index.service everyday at 5:00

[Timer]
OnCalendar=*-*-* 5:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
- `OnCalendar=`: first `*` means every year, second `*` means every month, third `*` means every day, so it will trigger everyday 5:00 morning.
- `Persistent=`: If the system is off at the set time, it will run the next time the system is turned on.

### 12. Enable and start `generate-index.service` 
```bash
sudo systemctl daemon-reload
```
- reload service after brought a new service file.

```bash
sudo systemctl enable --now generate_index.timer
```
- `--now`: it will enable and start the timer file at the same time. 

To list all active timers,
```bash
sudo systemctl list-timers
```
- This will print active timers on the system along with their next and last trigger times.

### 13. Move a config file `nginx.conf`
```bash
sudo mv ./linux-assignment3-part1/nginx.conf /etc/nginx/nginx.conf
```
- By default, you already have `nginx.conf` and it will be replaced.

`nginx.conf` file has contents like:
```bash
user webgen;
worker_processes auto;
worker_cpu_affinity auto;

events {
    multi_accept on;
    worker_connections 1024;
}

http {
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    log_not_found off;
    types_hash_max_size 4096;
    client_max_body_size 16M;

    # MIME
    include mime.types;
    default_type application/octet-stream;

    # logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```
- `user webgen`: by default, nginx execute master process as a root and worker process as a http user. this change it to user webgen.
- `Persistent=`: If the system is off at the set time, it will run the next time the system is turned on.

You can find more information about the structure of `nginx.conf` here. [nginx - ArchWiki](https://wiki.archlinux.org/title/Nginx)

### 14. Create managing directories
Separating server block files instead of modifying the main `nginx.conf` file isolates issues for better maintenance, and allows enabling, and disabling each specific site easily.
We are going to use **sites-enabled** and **sites-available** approach to manage server block files separately.
```bash
sudo mkdir /etc/nginx/sites-available /etc/nginx/sites-enabled
```
- `sites-available`: this directory contains all sites' server block files including enabled, and disabled files. 
- `sites-enabled`: this directory only contains symbolic links of the enabled sites' server block files. nginx will load files only from this directory. 

### 15. Move a server block file `webgen.conf`
 ```bash
 sudo mv ./linux-assignment3-part1/webgen.conf /etc/nginx/sites-available/webgen.conf
```

`webgen.conf` file has contents like:
 ```bash
 server {
    listen 80;
    listen [::]:80;
    server_name _;
    root /var/lib/webgen/HTML;
    index index.html;
    
 location / {
        try_files $uri $uri/ =404;
    }
}
```
- `listen 80`:  Set the server to listen on port 80 (the default port for HTTP) over IPv4.
- `listen [::]:80`:  Set the server to listen on port 80 over IPv6.
- `server_name _`: This will handle all requests regardless of the hostname. `_` means all host names.
- `root /var/lib/webgen/HTML`: define directory to look for the requested file. 
- `try_files $uri $uri/ =404`: try to find the requested file as a file or directory, if both fails, it will return 404 error.

### 16. Disable `httpd`
```bash
sudo systemctl disable --now httpd
```
- to free port 80 so that nginx can use it
- Since port 80 is already in use when httpd is running, nginx cannot run on port 80.

### 17. Enable and start `webgen.conf`

```bash
sudo systemctl daemon-reload
```
- reload service after brought a new service file.

```bash
sudo ln -s /etc/nginx/sites-available/webgen.conf /etc/nginx/sites-enabled/webgen.conf
```
- creating a symlink in the `enabled` directory works as enabling the server block.

```bash
sudo systemctl start nginx.service
```
- This will start nginx.service

To check if nginx is running and to get its current status
```bash
sudo systemctl status nginx
```
- This will print information about status of nginx.

To test the configuration for syntax errors
```bash
sudo nginx -t
```
- This command checks the configuration files for any syntax errors

### 18. allow `ssh` and `http` to `ufw` 
now we are going to set a firewall by using `ufw` package, which stands for uncomplecated firewall.
```bash
sudo ufw allow ssh
sudo ufw allow http
```
- This will allow access through `ssh` and `http` from everywhere

if you want to restrict by specific ip address,
```bash
sudo ufw allow from 123.456.789.0 ssh
```
- This will allow access only  ip address `123.456.789.0`

### 19. Enable and start `ufw`
```bash
sudo systemctl enable --now ufw.service
```

### 20. Make limit to `ufw`
```bash
sudo ufw limit ssh
```
- This will block the IP address if connection attempts occur more than 5 times in 60 seconds.
- Blocked IPs will be able to connect again after a certain amount of time, and this process will automatically reset.
- This is useful for mitigating brute force attacks.

To check the status of firewall syntax errors
```bash
sudo ufw status
```
- This will show you whether the firewall is active and list all the currently applied rules.


Congraturation! You have successfully set and configure a new web server!

## General 

ip address: 137.184.86.120

To improve `generate_index` with additional system information, I added host name and disk usage data.

1. add variables
```bash
# Variables
HOSTNAME=$(cat /etc/hostname)
RAW_DISK_USAGE=$(df -h | grep '^/dev')
DISK_USAGE=$(echo "$RAW_DISK_USAGE" | awk '{print $3 "/" $2}')
```

2. add html
```bash
# Create the index.html file 
cat <<EOF > "$OUTPUT_FILE"
...
<body> 
	<p><strong>Disk Usage:</strong> $DISK_USAGE </p> 
	<p><strong>Hostname:</strong> $HOSTNAME </p>
</body> 
EOF
```

3. output
```
System Information
Kernel Release: 6.11.9-arch1-1
Operating System: Arch Linux
Date: 20/11/2024
Number of Installed Packages: 213
Disk Usage: 2.6G/25G
Hostname: bob
```

4. the challenge that I encountered
I wanted to show disk usage with following format:
```
Disk Usage:  2.6G/25G
```

so originally I created the variable:
```
DISK_USAGE=$(df -h | grep '^/dev' | awk '{print $3 "/" $2}')
```

however, that generated this following error:
```
/var/lib/webgen/bin/generate_index: line 16: $3: unbound variable
```

that happens because between each field from the output of `df -h` contains more than single space.

so I store data from `df -h | grep '^/dev'` into variable first, which outputs like 
```
Disk Usage: /dev/vda3 25G 2.6G 22G 11% /
```

and then I created another variable to use the command that I wanted to use with the output of the previous variable.
```
echo "$DISK_USAGE" | awk '{print $3 "/" $2}'
```

