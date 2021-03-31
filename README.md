# Fresh Ubuntu Server (20.04) Setup (using DigitalOcean)

## Set up server, ssh key, and firewall

1. SSH to the server
```
ssh -i key.file user@IP-Address
```

2. Update and upgrade software
```
apt update && apt upgrade
```

3. Set hostname
```
hostnamectl set-hostname flask-server
```

4. Add hostname to host file
```
nano /etc/hosts
```
Then add server ip address and hostname under localhost. For example:  `1.1.1.1   flask-server`

5. Create limited privileged user, then enter password and optional info
```
adduser mtra
```
Then add new user to sudo group `adduser mtra sudo`

6. Log out `exit` then login (step 1) with the new user

8. Make `.ssh` folder in the server then create ssh-key (local) `ssh-keygen -b 4096` and copy it into `.ssh` folder (server). Note: perform this step if you didn't set up during server configuration.
```
scp ~/.ssh/id_rsa.pub mtra@1.1.1.1:~/.ssh/authorized_keys
```

9. Change folder and files permission
```
sudo chmod 700 ~/.ssh/
sudo chmod 600 ~/.ssh/*
```

10. Change sshd_config
```
nano /etc/ssh/sshd_config
```
Change `PermitRootLogin` to `no`
Change `PasswordAuthentication` to `no`
Then restart systen `sudo systemctl restart sshd`

11. Install firewall and update settings
```
sudo apt install ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 5000
sudo ufw enable
```
Run `sudo ufw status` to check status

## Now, you are ready to deploy the app

1. Copy your project to server
```
scp -r ~/Desktop/Python/ams mtra@1.1.1.1:~/
```

2. Install python, create virtual env, install dependencies
To get python 3.6 run the following
```
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt-get update
sudo apt install python3.6
```
How to fix add-apt-repository: command not found error
```
sudo apt-get install software-properties-common
```
Then run:
```
sudo apt install python3-pip
sudo apt install python3.6-venv
python3 -m venv ams/venv
cd ams
source venv/bin/activate
pip install -r requirements.txt
```

3. Create configuration file
```
sudo touch /etc/config.json
sudo nano /etc/config.json
```
The file format should be:
```
{
        "AMS_SECRET_KEY":"your_secret_key",
        "MAIL_USERNAME":"your_mail_username",
        "MAIL_PASSWORD":"your_mail_pw",
        "POSTGRES_URL":"url",
        "POSTGRES_USER":"pg_user",
        "POSTGRES_PW":"pg_pw",
        "POSTGRES_DB":"db_name"
}
```

4. Test the devel server; go to 1.1.1.1:5000 to see if the server is up; Ctrl+C to stop after checking
```
export FLASK_APP=app.py
flask run --host=0.0.0.0
```

5. Install nginx (web server which handles static files) and gunicorn (handle python code)
```
sudo apt install nginx
pip install gunicorn
```

6. Update config for nginx
```
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/ams
```
ams file format
```
server {
        listen 80;
        server_name 64.227.96.107;

        location /static {
                alias /home/mtra/ams/ams/static;
        }

        location / {
                proxy_pass http://localhost:8000;
                include /etc/nginx/proxy_params;
                proxy_redirect off;
        }
}
```

7. Update firewall traffic and restart nginx
```
sudo ufw allow http/tcp
sudo ufw delete allow 5000
sudo ufw enable
sudo systemctl restart nginx
```

8. Update config for gunicorn (w: # of workers). Check `nproc --all` for number of code. Typically, workers = corex2 + 1
```
gunicorn -w 3 app:app
```

9. Install supervisor, and add config file
```
sudo apt install supervisor
sudo nano /etc/supervisor/conf.d/ams.conf
```
ams.conf file format:
```
[program:ams]
directory=/home/mtra/ams
command=/home/mtra/ams/venv/bin/gunicorn -w 3 app:app
user=mtra
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
stderr_logfile=/var/log/ams/ams.err.log
stdout_logfile=/var/log/ams/ams.out.log
```
Create folder for log files, and add log files
```
sudo mkdir -p /var/log/ams
sudo touch /var/log/ams/ams.err.log
sudo touch /var/log/ams/ams.out.log
```

10. Install postgressql, run migration and add assets
```
sudo apt update
sudo apt install postgresql postgresql-contrib
# swtich to postgres
sudo -i -u postgres
# create db
createdb ams
# type psql to get into PostgreSQL prompt
psql
# Alter postgres password
ALTER ROLE postgres WITH PASSWORD 'testing';
# exit out
\q
# run migration
python manage.py db upgrade
# check that tables are created
sudo -i -u postgres
psql
# connect to ams
postgres=# \c amc
postgres=# \dt
\q
exit
# add assets
python insert_dropdown_tables.py
python insert_assets.py 
```

11. Reload supervisor
```
sudo supervisorctl reload
```

## Other

How to stop/start supervisor?
```
sudo systemctl stop supervior
sudo systemctl start supervior
# check status
sudo systemctl status supervior
```

How to run app and debugging?
```
# enable port 5000
sudo ufw allow 5000
sudo ufw enable
sudo systemctl restart nginx

# then run app
cd ams
export FLASK_APP=app.py
flask run --host=0.0.0.0
```
