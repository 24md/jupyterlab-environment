# Set Up JupyterLab Environment on a Vultr Cloud Instance

The following are the prerequisites and the steps to set up Jupyterlab Enivornment on a Vultr Cloud instance.

## Prerequisites

* Deploy a [Vultr Cloud instance](https://www.vultr.com/servers/ubuntu) with Ubuntu.
* Map a subdomain to the instance using an `A` record for permanent deployment.

## Deploy Temporary Enivornment

Add a new user and log into it.

```console
# adduser NEW_USER
# usermod -aG sudo NEW_USER
# su - NEW_USER
```

Install the Python virtual environment package.

```console
$ sudo pip install -U virtualenv
```

Create and enter a new virutal environment for JupyterLab.

```console
$ virtualenv --system-site-packages -p python3 jupyterlab_env
$ source jupyterlab_env/bin/activate
```

Update pip and exit the virtual environment.

```console
$ pip install --upgrade pip
$ deactivate
```

Install the JupyterLab package using `pip`.

```console
$ sudo pip install -U jupyterlab
```

Create a password hash for protecting JupyterLab.

```console
$ python3 -c "from jupyter_server.auth import passwd; print(passwd('YOUR_PASSWORD'))"
```

Create and edit the JupyterLab configuration file.

```console
$ jupyter lab --generate-config
$ nano /home/NEW_USER/.jupyter/jupyter_lab_config.py
```

Find and replace the following values in the file.

```python
c.ServerApp.password = 'PASSWORD_HASH'
c.ServerApp.allow_remote_access = True
```

Disable the firewall temporarily to allow connections to JupyterLab.

```console
$ sudo ufw disable
```

Run the JupyterLab server temporarily.

```console
$ jupyter lab --ip 0.0.0.0
```

The above command will spawn a JupyterLab server that listens on port 8888. Verify the deployment by opening the instance's public IP in your web browser.


## Deploy Permanent Enivornment

Uninstall existing Jupyter kernel.

```console
$ sudo python3 -m pip uninstall ipykernel
```

Enter the virtual environment

```console
$ source jupyterlab_env/bin/activate
```

Create a new Jupyter kernel.

```console
$ pip install ipykernel
$ python3 -m ipykernel install --user --name=venvpy3
```

Copy $PATH into clipboard.

```console
$ echo $PATH
```

Exit the virtual environment.

```console
$ deactivate
```

Create a new service named `jupyterlab`.

```console
$ sudo nano /lib/systemd/system/jupyterlab.service
```

Add the following contents to the file.

```systemd
[Unit]
Description=JupyterLab Server

[Service]
User=NEW_USER
Group=NEW_USER
WorkingDirectory=/home/NEW_USER/jupyterlab
Environment="PATH=PATH_HERE"
ExecStart=/usr/local/bin/jupyter-lab --config=/home/NEW_USER/.jupyter/jupyter_lab_config.py

[Install]
WantedBy=multi-user.target
```

Create the working directory for JupyterLab.

```console
$ mkdir ~/jupyterlab
```

Initialize the `jupyterlab` service.

```console
$ sudo systemctl daemon-reload
$ sudo systemctl start jupyterlab
$ sudo systemctl status jupyterlab
```

Install the Nginx package using `apt`.

```console
$ sudo apt install nginx
```

We will use Nginx as the reverse proxy server for channeling traffic to our JupyterLab server.

Swap the Nginx configuration file with the reverse procy configuration.

```console
$ sudo rm -f /etc/nginx/nginx.conf
$ sudo nano /etc/nginx/nginx.conf
```

Add the following contents to the file.

```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    server {
        listen 80;
        server_name YOUR_HOSTNAME;

        location / {
            proxy_pass http://127.0.0.1:8888;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header X-Scheme $scheme;

            proxy_buffering off;
        }

        location ~ /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }
}

events {}
```

Restart the `nginx` server.

```console
$ sudo systemctl restart nginx
```

Install the `certbot` package using `snap`.

```console
$ sudo snap install --classic certbot
```

Create a new Let's Encrypt certificate for your hostname.

```console
$ sudo certbot --nginx -d YOUR_HOSTNAME
```

Set the firewall rules and enable the firewall.

```console
$ sudo ufw allow 'Nginx Full'
$ sudo ufw enable
$ sudo ufw status
```

You can now acecss the JupyterLab environment using your subdomain/hostname via the web browser.
