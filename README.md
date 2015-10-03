# unicorn
Unicorn on Ubuntu, Nginx, and Rails

This is a simple guide on how to setup the Nginx/Unicorn/Rails stack on Ubuntu server. 
Here’s what we’ll be covering:


    Ubuntu
    Nginx
    Unicorn
    Rbenv
    Rails 4.1
    PostgreSQL
    Redis
    Memcache
    Mina (for deployments)

#### MINA

We will use [Mina](http://nadarei.co/mina/). A really fast deployer and server automation tool

Add it to your Gemfile.

gem 'mina'

Let’s also add two other gems that will help us manage Unicorn and Sidekiq

gem 'mina-sidekiq', :require => false

gem 'mina-unicorn', :require => false

Create the necessary [“deploy.rb”](../master/deploy.rb):


$ mina init

Created config/deploy.rb.

Make sure to add your project’s folder inside ‘/home/deploy’. 

For instance, ‘/home/deploy/YOUR_APP’.

See [deploy.rb](../master/deploy.rb) file for example tweaked setup.


Ok, let’s run that setup task to create the necessary folders and files on your server. If one doesn’t get created that you need, no worries. Just SSH back in there and create the folder yourself.

$ mina setup

-----> Creating folders... done.

#### Set Up database.yml and secrets.yml

Let’s get back on your Droplet and set up your database.yml and secrets.yml files:

$ ssh deploy@YOUR_IP

$ nano /home/deploy/YOUR_APP/shared/config/database.yml

Tweak that file:

`production:
  adapter: postgresql
  encoding: unicode
  database: APPNAME_production
  username: postgres
  password: DB_PASSWORD_SET_ABOVE
  host: localhost
`
…and your secrets.yml…

`nano /home/deploy/YOUR_APP/shared/config/secrets.yml`

production:
  secret_key_base: RUN `rake secret` TO GENERATE A KEY