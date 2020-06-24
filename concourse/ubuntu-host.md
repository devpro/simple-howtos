# HOWTO Run Concource CI on Ubuntu

This steps were made on an Azure VM Ubuntu 18.04 LTS, in February 2020.

## Steps

### Requirements

#### Give SSH access

Make sure all your workstation public keys are in `~/.ssh/authorized_keys` (one per line).

#### Activate the firewall

```bash
sudo ufw app list
```

> Available applications:  
> OpenSSH  

```bash
sudo ufw allow OpenSSH
```

> Rules updated  
> Rules updated (v6)  

```bash
sudo ufw enable
```

> Command may disrupt existing ssh connections. Proceed with operation (y|n)? y  
> Firewall is active and enabled on system startup  

```bash
sudo ufw status
```

> Status: active  
>  
> To                         Action      From  
> --                         ------      ----  
> OpenSSH                    ALLOW       Anywhere  
> OpenSSH (v6)               ALLOW       Anywhere (v6)  

#### Ubuntu 18.04 patch

See [askubuntu.com/questions/105772](https://askubuntu.com/questions/1057723/why-do-i-need-to-add-nameservers-to-resolv-conf/1057752#1057752) for more details (from an answer on [discuss.concourse-ci.org](https://discuss.concourse-ci.org/t/workers-containers-no-longer-resolve-internal-dns-in-v5-0/1245/9)).

```bash
sudo rm -f /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

This is the fix for the post-installation issue, while getting a git repository in a pipeline.

> fly -t vmazure check-resource -r aspnetcore/git-repository  
> id  name            status   check_error  
> 3   git-repository  errored  resource script '/opt/resource/check []' failed: exit status 128  
>  
> stderr:  
> Cloning into '/tmp/git-resource-repo-cache'...  
> fatal: unable to access 'https://github.com/devpro/cf-dotnet-samples.git/': Could not resolve host: github.com  

#### DNS registry

In Azure, add a record A on the URL and the public IP of the VM in the DNS Zone.

#### Cloud hosting restrictions

Make sure to enable the inbound HTTP requests on ports 80, 8080.

In Azure, in the VM ressource page, it is in the networking page.

### PostgreSQL

```bash
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib
sudo -u postgres createuser concourse
sudo -u postgres createdb --owner=concourse atc
```

### Concourse

#### Command lines

Check [latest version](https://github.com/concourse/concourse/releases/latest)

```bash
cd /tmp

curl -LO https://github.com/concourse/concourse/releases/download/v5.8.0/concourse-5.8.0-linux-amd64.tgz
curl -LO https://github.com/concourse/concourse/releases/download/v5.8.0/fly-5.8.0-linux-amd64.tgz

sudo tar -zxf concourse-5.8.0-linux-amd64.tgz -C /usr/local
sudo tar -zxf fly-5.8.0-linux-amd64.tgz -C /usr/local/concourse/bin

export PATH=$PATH:/usr/local/concourse/bin
source ~/.profile

cd ~
concourse --version
# should display 5.8.0
fly --version
# should display 5.8.0

sudo mkdir /etc/concourse

sudo ssh-keygen -t rsa -q -N '' -f /etc/concourse/tsa_host_key
sudo ssh-keygen -t rsa -q -N '' -f /etc/concourse/worker_key
sudo ssh-keygen -t rsa -q -N '' -f /etc/concourse/session_signing_key
sudo cp /etc/concourse/worker_key.pub /etc/concourse/authorized_worker_keys

sudo vi /etc/concourse/web_environment
sudo vi /etc/concourse/worker_environment

sudo adduser --system --group concourse
sudo chown -R concourse:concourse /etc/concourse
sudo chmod 600 /etc/concourse/*_environment

sudo vi /etc/systemd/system/concourse-web.service
sudo vi /etc/systemd/system/concourse-worker.service

sudo ufw allow 8080
sudo ufw default allow routed
sudo systemctl start concourse-web concourse-worker
sudo systemctl status concourse-web concourse-worker
sudo systemctl enable concourse-web concourse-worker
```

#### File contents (1)

- `/etc/concourse/web_environment`

```ini
CONCOURSE_SESSION_SIGNING_KEY=/etc/concourse/session_signing_key
CONCOURSE_TSA_HOST_KEY=/etc/concourse/tsa_host_key
CONCOURSE_TSA_AUTHORIZED_KEYS=/etc/concourse/authorized_worker_keys
CONCOURSE_POSTGRES_SOCKET=/var/run/postgresql
CONCOURSE_EXTERNAL_URL=http://my.domain.com:8080
CONCOURSE_MAIN_TEAM_LOCAL_USER=test
CONCOURSE_ADD_LOCAL_USER=test:mysecretpassword
```

- `/etc/concourse/worker_environment`

```ini
CONCOURSE_WORK_DIR=/var/lib/concourse
CONCOURSE_TSA_WORKER_PRIVATE_KEY=/etc/concourse/worker_key
CONCOURSE_TSA_PUBLIC_KEY=/etc/concourse/tsa_host_key.pub
CONCOURSE_TSA_HOST=127.0.0.1:2222
CONCOURSE_GARDEN_DNS_SERVER=8.8.8.8
```

- `/etc/systemd/system/concourse-worker.service`

```ini
[Unit]
Description=Concourse CI worker process
After=concourse-web.service

[Service]
User=root
Restart=on-failure
EnvironmentFile=/etc/concourse/worker_environment
ExecStart=/usr/local/concourse/bin/concourse worker

[Install]
WantedBy=multi-user.target
```

- `/etc/systemd/system/concourse-web.service`

```ini
[Unit]
Description=Concourse CI web process (ATC and TSA)
After=postgresql.service

[Service]
User=concourse
Restart=on-failure
EnvironmentFile=/etc/concourse/web_environment
ExecStart=/usr/local/concourse/bin/concourse web

[Install]
WantedBy=multi-user.target
```

#### Investigate

```bash
# in case of error, how to debug
# set environment variables from the file we created, then run manually the process
sudo -E -u concourse /usr/local/concourse/bin/concourse web
```

### Nginx

See [How To Install Nginx on Ubuntu 18.04 by DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04)

```bash
sudo apt-get update
sudo apt-get install nginx

sudo ufw app list
# Available applications:
#   Nginx Full
#   Nginx HTTP
#   Nginx HTTPS
#   OpenSSH

sudo ufw allow 'Nginx HTTP'
sudo ufw status

systemctl status nginx
# open http://my.public.ip/ and make sure you can see the default nginx landing page (Welcome to nginx!)

sudo vi /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
sudo vi /etc/concourse/web_environment
sudo systemctl restart concourse-web
sudo netstat -plunt | grep 8080
sudo ufw delete allow 8080
```

Useful commands:

```bash
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl disable nginx
sudo systemctl enable nginx
```

### Let's Encrypt

See [How To Secure Nginx with Let's Encrypt on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-18-04)

```bash
sudo add-apt-repository ppa:certbot/certbot
sudo apt install python-certbot-nginx

sudo ufw delete allow 'Nginx HTTP'
sudo ufw status

sudo certbot --nginx -d example.com

sudo certbot renew --dry-run
```

#### File contents (2)

- `/etc/nginx/sites-enabled/default`

```ini
upstream concourse {
         server 127.0.0.1:8080;
}

server {
    if ($host = my-domain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

        listen 80 default_server;
        listen [::]:80 default_server;

        server_name my-domain.com;

        return 301 https://$server_name$request_uri;
}

server {
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server;
        #include snippets/ssl-my-domain.com.conf;
        #include snippets/ssl-params.conf;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;

        server_name my-domain.com;

        location / {
                include proxy_params;
                proxy_http_version 1.1;
                proxy_read_timeout 90;

                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";

                proxy_pass http://concourse;
        }

        location ~ /.well-known {
                allow all;
        }

    ssl_certificate /etc/letsencrypt/live/my-domain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/my-domain.com/privkey.pem; # managed by Certbot
}
```

- `/etc/concourse/web_environment`

```bash
CONCOURSE_SESSION_SIGNING_KEY=/etc/concourse/session_signing_key
CONCOURSE_TSA_HOST_KEY=/etc/concourse/tsa_host_key
CONCOURSE_TSA_AUTHORIZED_KEYS=/etc/concourse/authorized_worker_keys
CONCOURSE_POSTGRES_SOCKET=/var/run/postgresql
CONCOURSE_EXTERNAL_URL=https://my-domain.com
CONCOURSE_BIND_IP=127.0.0.1
CONCOURSE_MAIN_TEAM_LOCAL_USER=test
CONCOURSE_ADD_LOCAL_USER=test:mysecretpassword
```

### HashiCorp Vault

#### Installation of Vault

See [How To Securely Manage Secrets with HashiCorp Vault on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-securely-manage-secrets-with-hashicorp-vault-on-ubuntu-16-04) for the original steps.

```bash
# Download the latest version (get the id from https://www.vaultproject.io/downloads/)
wget https://releases.hashicorp.com/vault/1.3.2/vault_1.3.2_linux_amd64.zip

# Install unzip
sudo apt-get update
sudo apt-get install unzip

# Unzip the zip file (will create vault file)
unzip vault_1.3.2_linux_amd64.zip

# Move the exe file to user local bin
sudo mv vault /usr/local/bin/

sudo setcap cap_ipc_lock=+ep /usr/local/bin/vault

# Display version
vault --version

# Enable autocompletion
vault -autocomplete-install

# Add vault user
sudo useradd -r -d /var/lib/vault -s /bin/nologin vault

# Set ownership on Vault lib directory
sudo install -o vault -g vault -m 750 -d /var/lib/vault

# Set up Vault’s configuration file
sudo vi /etc/vault.hcl

sudo chown vault:vault /etc/vault.hcl
sudo chmod 640 /etc/vault.hcl

sudo vi /etc/systemd/system/vault.service

sudo groupadd pki
sudo chgrp pki -R /etc/letsencrypt/{archive,live}
sudo chmod g+rx -R /etc/letsencrypt/{archive,live}
sudo gpasswd -a vault pki

echo 127.0.0.1 example.com | sudo tee -a /etc/hosts

sudo systemctl start vault
sudo systemctl status vault

export VAULT_ADDR=https://example.com:8200
vault status

operator init
operator unseal
```

#### Debug with Vault

```bash
/usr/local/bin/vault server -dev
```

#### File contents (Vault step)

Replace `example.com` with your domain.

```ini
# /etc/vault.hcl
backend "file" {
        path = "/var/lib/vault"
}

listener "tcp" {
        tls_disable = 0
        tls_cert_file = "/etc/letsencrypt/live/example.com/fullchain.pem"
        tls_key_file = "/etc/letsencrypt/live/example.com/privkey.pem"
}

# /etc/systemd/system/vault.service
[Unit]
Description=a tool for managing secrets
Documentation=https://vaultproject.io/docs/
After=network.target
ConditionFileNotEmpty=/etc/vault.hcl

[Service]
User=vault
Group=vault
ExecStart=/usr/local/bin/vault server -config=/etc/vault.hcl
ExecReload=/usr/local/bin/kill --signal HUP $MAINPID
#CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
#Capabilities=CAP_IPC_LOCK+ep
#SecureBits=keep-caps
#NoNewPrivileges=yes
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
```

## References

- [Official Install documentation page](https://concourse-ci.org/install.html)
- [How To Install Concourse CI on Ubuntu 16.04 by DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-install-concourse-ci-on-ubuntu-16-04)
