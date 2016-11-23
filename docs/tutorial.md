# Tutorial

Here's a tutorial to install and integrate Rocket.chat and LabAdmin on a machine. We are going to
deploy the system on a Vagrant virtual machine. The tutorial will assume a debian based distro,
we recommend the latest Debian stable or Ubuntu LTS.

You need a checkout of this repository on your machine.

## Requirements

Requirements:
- ansible 2.0+
- git

Optional whether using a virtual machine:

- vagrant
- a vagrant provider, *virtualbox* is the more hassle free


## Rocket.Chat

Rocket.chat requires quite a bit of infrastructure to run so we'll leverage Rocket.chat own ansible
scripts to deploy it.

From the dir containing the checkout of this repository:

```
ansible-galaxy install -r chatlab/ansible/requirements.yml -p chatlab/ansible/roles
```

If not using Vagrant, run this:

```
absible-playbook "chatlab/ansible/rocket_chat.yml"
```

Let's start with a simple vagrant file:

```
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/jessie64"

  # uses dhcp
  config.vm.network "public_network"

  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    # path to ansible script in the chatlab repository
    ansible.playbook = "chatlab/ansible/rocket_chat.yml"
  end
end
```

Save the file as *Vagrantfile* on some dir.

In order to create and provision the virtual machine run this command from the same dir:

```
vagrant up --provision
```

After a few we should have our machine up and running, this command will show the virtual machine status:

```
vagrant status
```

## LabAdmin

Connect to the virtual machine:

```
vagrant ssh
```

From inside the virtual machine we are going to install all requirements.
Please note that *mysql-server* will require a password for the root user that you'll need later.

```
sudo apt install build-essential python3 python3-dev python-virtualenv libjpeg-dev libpq-dev libmysqlclient-dev git python2.7 mysql-server
sudo apt-get clean
```

In order to setup the mysql database we need to enter the mysql shell, youl'll be asked the root user password:

```
mysql -u root -p
```

From inside the shell we'll create the db, the user and grant the needed perms, please use a different password than the one below:

```
create database labadmin;
# please use a different password
CREATE USER 'labadmin'@'localhost' IDENTIFIED BY 'apasswordforlabadmin';
GRANT ALL PRIVILEGES ON labadmin.* TO 'labadmin'@'localhost';
FLUSH PRIVILEGES;
```

Then we need to create a user that will run the *LabAdmin* application:

```
sudo useradd -M labadmin
sudo mkdir /var/www/labadmin
sudo chown labadmin /var/www/labadmin
cd /var/www/labadmin
```

Then we'll switch *labadmin* user for the next commands until further notice:

```
sudo su labadmin
```

We'll switch to a more comfortable shell:

```
bash
```

Now as the *labadmin* user we can setup the *labadmin* instance:

```
git clone https://github.com/OfficineArduinoTorino/chatlab
virtualenv -p python3.4 venv
. ./venv/bin/activate
pip install -r chatlab/LabAdmin/requirements.txt
pip install https://github.com/FablabTorino/LabAdmin/archive/master.zip

mkdir bin
cp chatlab/LabAdmin/labadmin bin/
chmod +x bin/labadmin
```

Then we are creating a django project:

```
django-admin startproject labadmin
mkdir labadmin/uploads
```

It's time to configure *labadmin/labadmin/settings.py*.

We need to add the needed apps:

```
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'oauth2_provider',
    'corsheaders',
    'labAdmin',
]
```

And put the *CorsMiddleware* just before the *CommonMiddleware*:

```
MIDDLEWARE_CLASSES = [
    # ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    # ...
]
```

Then we need to setup the path for uploaded files, and various urls:

```
MEDIA_ROOT = '/var/www/labadmin/labadmin/uploads/'
STATIC_URL = '/labadmin/static/'
LOGIN_URL = '/labadmin/accounts/login/'
```

You also need to update these settings dependings on your environment:

```
# See https://docs.djangoproject.com/en/1.10/ref/settings/#databases
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'labadmin',
        'USER': 'labadmin',
        'PASSWORD': 'apasswordforlabadmin',
        'HOST': '/var/run/mysqld/mysqld.sock',
    }
}

# these depends on where the machine is
LANGUAGE_CODE = 'it-it'
TIME_ZONE = 'Europe/Rome'
```

We are done with django *settings.py*, it's time to add to *labadmin/labadmin/urls.py*:

```
from django.conf.urls import url, include
from django.contrib.auth import views as auth_views


labadminpatterns = [
    url(r'admin/', admin.site.urls),
    url(r'^labAdmin/', include('labAdmin.urls')),
    url(r'^o/', include('oauth2_provider.urls', namespace='oauth2_provider')),

    url(r'^accounts/login/$', auth_views.login, {'template_name': 'labadmin/login.html'}, name='login'),
    url(r'^accounts/logout/$', auth_views.logout, name='logout'),
    url(r'^accounts/password_reset/$', auth_views.password_reset, name='password_reset'),
    url(r'^accounts/password_reset/done/$', auth_views.password_reset_done, name='password_reset_done'),
    url(r'^accounts/reset/(?P<uidb64>[0-9A-Za-z_\-]+)/(?P<token>[0-9A-Za-z]{1,13}-[0-9A-Za-z]{1,20})/$', auth_views.password_reset_confirm, name='password_reset_confirm'),
    url(r'^accounts/reset/done/$', auth_views.password_reset_complete, name='password_reset_complete'),
]

urlpatterns = [
    url(r'^labadmin/', include(labadminpatterns)),
]
```

Now that our Django project is configured we can create the database tables and an admin user:

```
cd labadmin/labadmin
./manage.py migrate
./manage.py createsuperuser
```

We are going to remove the shell for the labadmin user. Remember to exit from the 
*labadmin* shell first and then type:

```
sudo usermod -s /bin/false labadmin
```

Finally we are telling the system how to handle the application. If your distribution uses systemd
to manage processes use these instructions:

```
sudo cp /var/www/labadmin/chatlab/LabAdmin/labadmin.service /etc/systemd/system
sudo systemctl enable labadmin.service
sudo systemctl start labadmin.service
```

otherwise if it's upstart based use these:

```
sudo cp /var/www/labadmin/chatlab/LabAdmin/labadmin.upstart /etc/init/labadmin.conf
sudo initctl reload-configuration
sudo initctl start labadmin
```

Now we need to setup nginx to proxy for LabAdmin:

```
sudo cp /var/www/labadmin/chatlab/LabAdmin/labadmin.nginx.conf /etc/nginx/conf.d/labadmin.conf
sudo service nginx testconfig
sudo service nginx reload
```

Now that we have both applications up and running we can configure a way to log in from rocket.chat
with labadmin users. The following urls are for localhost, you have to update them to match the
ip of your virtual machine.

First load [https://127.0.0.1](Rocket.Chat) and register an user, the first registered user
would be the admin user.

Then go to the [https://127.0.0.1/admin/OAuth](OAuth Administration panel) and Add a custom
OAuth provider, the button is on the bottom right. When prompted for a name use *chatlab*.
Once the provider is configured it's time to start configure it. Expand it and update the following
configurations:

```
Enable: True
# use the proper ip / host of your machine
URL: https://127.0.0.1
Token Path: /labadmin/o/token/
Identity Path: /labadmin/labAdmin/user/identity/
Authorize Path: /labadmin/o/authorize/
Scope: read
Button Text: chatlab
```

We are not done yet, but we'll need to create the OAuth application on the LabAdmin side.

In a new tab open the [https://127.0.0.1/labadmin/admin/](Django Admin) and then
[https://127.0.0.1/labadmin/o/applications/register/](register) a new application:

```
Name: rocket.chat
Client type: Confidential
Authorization grant type: Authorization code
# use the proper ip / host of your machine
Redirect uris: https://127.0.0.1/_oauth/chatlab
```

Now you have to copy the *Client id* and *Client secret* respectively as *Id* and *Secret* on the
Rocket.chat OAuth provider configuration and finally save the configuration.

Congratulations you have configured Chatlab! You should now being able to log in Rocket.chat through
the *chatlab* provider.


## TODO

- script the labAdmin configuration with ansible
