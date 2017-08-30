Linux Server Configuration Project

IP Address:
35.176.125.65

SSH Port:
2200 

URL:
http://35.176.125.65/

Summary:
An installation of Ubuntu on a virtual machine by Amazon Lightsail. It has been configured and prepared so that it hosts playlist-catalog (a basic catalogging web app): https://github.com/rudyc123/playlists-catalog 

Setting up server:
- Start a new Ubuntu Linux server on Amazon Lightsail
- Configure VM to be accessed using custom SSH keys (udacity_keys.rsa)
- ssh into ubuntu user 
	ssh -i ~/.ssh/udacity_keys ubuntu@35.176.125.65
- Update and upgrade packages
	sudo apt-get update
	sudo apt-get upgrade
- Change ssh port from 22 to 2200
	sudo vim /etc/ssh/sshd_config 
		- Find Port line and change to 2200
	- *Amazon Lightsail - add custom port (2200)
- Create new user (grader) and grant them sudo permissions
	sudo adduser grader
	sudo vim /etc/sudoers.d/grader
		- fill in: "grader ALL=(ALL:ALL) ALL"	
- Get auth keys of ubuntu user and copy to new user directory (grader)
	touch /home/grader/.ssh/authorized_keys
	cp ~/.ssh/authorized_keys /home/grader/.ssh/authorized_keys
- Change directory and keys permissions for grader
	sudo chmod 700 /home/grader/.ssh
	sudo chmod 644 /home/grader/.ssh/authorized_keys
- Change owner from ubuntu to new user (grader)
	sudo chown -R grader:grader /home/grader/.ssh
- Exit ubuntu user (exit) and log into grader w. priv ssh keys (udacity_keys.rsa)
	ssh -i ~/.ssh/udacity_key grader@*IPADDRESS* -p 2200    
- Configure local timezone to UTC
	sudo dpkg-reconfigure tzdata
	- ntp daemon ntpd for better sync
		sudo apt-get install ntp
- Enforce key-based authentication
	sudo vim /etc/ssh/sshd_config
		- Find PassswordAuthentication line and change to no
- Disable ssh login for ubuntu user
	sudo vim /etc/ssh/sshd_config
		- Find PermitRootLogin line and change to no
		- Find PasswordAuthentication and change to yes
		- Apend file with AllowUsers grader at end
	sudo service ssh restart
- Config Uncomplicated Firewall (UFW) to only allow incoming connections for ssh (port 2200), http (port 80), and ntp (port 123)
	sudo ufw allow 2200/tcp
	sudo ufw allow 80/tcp
	sudo ufw allow 123/udp
	sudo ufw enable
- Install Apache and mod_wsgi
	sudo apt-get install apache2
	sudo apt-get install libapache2-mod-wsgi python-dev
- Enable mod_wsgi
	sudo a2enmod wsgi
- Install git
	sudo apt-get install git
- Clone app from GitHubb
	cd /var/www
	sudo mkdir p_catalog
	sudo chown -R grader:grader p_catalog
	cd /p_catalog
	git clone https://github.com/rudyc123/playlists-catalog.git
- Configure a wsgi file to serve the app over mod_wsgi
	sudo touch p_catalog.wsgi
	- Add:
		import sys
		import logging
		logging.basicConfig(stream=sys.stderr)
		sys.path.insert(0, "/var/www/p_catalog/")
		
		from p_catalog.__init__ import app as application
		application.secret_key = 'debugging_secret_key'
- Ensure directory structure is like so:
	p_catalog/
	├── p_catalog
	│   ├── client_secrets.json
	│   ├── database_setup.py
	│   ├── fb_client_secrets.json
	│   ├── __init__.py
	│   ├── playlistcatalog.db
	│   ├── README.md
	│   ├── sample_data.py
	│   ├── static
	│   ├── templates
	│   └── venv
	└── p_catalog.wsgi
- Install Virtual Environment and Project Dependencies
	pip: sudo apt-get install python-pip
	virtualenv: sudo pip install virtualenv
- Install dependencies inside project folder (see above) in new venv and activate it
		cd /var/www/p_catalog
		sudo virtualenv venv
		sudo source venv/bin/activate
	- Change permissions of venv folder
		sudo chmod -R 777 venv
	- Install Flask and dependencies
		sudo pip install Flask
		sudo pip install httplib2
		sudo pip install requests
		sudo pip install oauth2client
		sudo pip install sqlalchemy
		sudo pip install python-psycopg2
- Config and enable new virtual host
	sudo vim /etc/apache2/sites-available/p_catalog.conf
	- Add:
<VirtualHost *:80>
		ServerName 35.176.125.65
		ServerAdmin admin@35.176.125.65
		WSGIScriptAlias / /var/www/p_catalog/p_catalog.wsgi
		<Directory /var/www/p_catalog/p_catalog/>
			Order allow,deny
			Allow from all
		</Directory>
		Alias /static /var/www/p-catalog/p-catalog/
		<Directory /var/www/p_catalog/p_catalog/static/>
			Order allow,deny
			Allow from all
		</Directory>
		ErrorLog ${APACHE_LOG_DIR}/error.log
		LogLevel warn
		CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>	
- Enable new virtual host
	sudo a2ensite p_catalog
- Install and config PostgreSQL (do not allow remote connnections, create new db w a new user (catalog) that has limited permissions to the application db).
	sudo apt-get install libpq-dev
	sudo apt-get install postgresql postgresql-contrib
	- Chnage into postgres user 
		sudo su - postgres
		psql
	- Create and config user permissions of db
		CREATE USER catalog WITH PASSWORD 'password';
		ALTER USER catalog CREATEDB;
		CREATE DATABASE catalog WITH OWNER catalog;
	- Connect to db and config (cont.)
		\c catalog
		REVOKE ALL ON SCHEMA public FROM public;
		GRANT ALL ON SCHEMA public TO catalog;
	- Log out
		\q 
		exit
- Config Flask app db engine in all appropriate py files
	engine = create_engine("postgresql://catalog:password@localhost/catalog")
- Setup database and/or any sample data
	sudo python database_setup.py
	sudo python sample_data.py
- Config OAuth JS origins on Google API manager and Facebook Developer websites. Redirect: IP ADDRESS
- Config gconnect() and fbconnect() functions
	Change  oauth_flow = flow_from_clientsecrets("client_secrets.json", scope="")
	TO:  oauth_flow = flow_from_clientsecrets(r"/var/www/p_catalog/p_catalog/client_secrets.json", scope="")
 
	Change  app_id = json.loads(open('fb_client_secrets.json', 'r').read())['web']['app_id']
		app_secret = json.loads(open('fb_client_secrets.json', 'r').read())['web']['app_secret']
	TO:     app_id = json.loads(open('/var/www/p_catalog/p_catalog/fb_client_secrets.json', 'r').read())['web']['app_id']
   		
		 app_secret = json.loads open('/var/www/p_catalog/p_catalog/fb_client_secrets.json', 'r').read())['web']['app_secret']

- Restart Apache to launch app
	sudo service apache2 restart
- Visit IP address/url in browser

Main references used:
Udacity Forums
https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04
https://www.digitalocean.com/community/tutorials/how-to-add-and-delete-users-on-an-ubuntu-14-04-vps
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
https://www.cheatography.com/nielzzz/cheat-sheets/ububtu-server/
