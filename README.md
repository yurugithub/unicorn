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

#### Set Up A User For Deployments

It’s not good practice to use your root user to handle deployments, so let’s first start by adding a “deploy” user.

`$ ssh root@YOUR_BOX_IP`

`$ sudo adduser deploy`

`$ su deploy`

#### SSH Keys

Stop using that root password. Setup and upload an SSH key.

Digital Ocean has a [fine document](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2) on creating a SSH key and getting it installed. The useful command is ssh-copy-id. It’s quite handy and easy to install with Homebrew if you're a mac user.


#### Rbenv

There is a better way to ruby than using RVM. Please use Rbenv. Let’s get it installed for your deploy user.

First, some dependencies:

`$ sudo apt-get update`

`$ sudo apt-get install curl git-core build-essential zlib1g-dev libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libcurl4-openssl-dev libxml2-dev libxslt1-dev python-software-properties`

Now, we’ll install rbenv into your home directory and add some commands to your .bashrc for completution and shims.

`$ git clone https://github.com/sstephenson/rbenv.git ~/.rbenv`

`$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc`

`$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc`

Now, let’s restart the shell and make sure Rbenv is install:

`$ exec $SHELL`

`$ type rbenv`

`#=> "rbenv is a function"`

Time to install Ruby!

`$ git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build`

`$ rbenv install 2.1.2`

`$ rbenv global 2.1.2`

This part may take awhile. Go grab a coffee.

Ok, done?

Let’s make sure all is well…

`$ ruby -v`

`ruby 2.1.2p95 (2014-05-08 revision 45877) [x86_64-linux]`

If it’s not working, you’ll probably see this:

`$ ruby -v`

    `The program 'ruby' can be found in the following packages:
    * ruby
    * ruby1.8
    Try: apt-get install <selected package>`

^ That’s not good. Re-visit the instructions above and don’t proceed until you get Ubuntu figuring out where Ruby is.

One last step that always trips people up: install Bundler real quick.

`$ gem install bundler`

You’ll thank me later…

#### PostgreSQL

Use MySQL if you must, but I ain’t helping you there. Let’s set up PostgreSQL.

Don’t try and get fancy with other DB users. Just use the built-in ‘postgres’ user.

`$ sudo apt-get install postgresql postgresql-contrib`

`$ createuser --pwprompt`

Create your database:

`$ su postgres`

`$ psql`
    psql (9.3.5)
    Type "help" for help.

    postgres=# CREATE DATABASE yourapp_production;


#### Memcache, and Redis

These two are a piece of cake. Thanks, ‘apt-get’

Memcache

`$ sudo apt-get install memcached`

And Redis (for Sidekiq and other fun stuff)

`$ apt-get install redis-server`

Boot ‘er up and make sure you can get to the Redis console:

`$ redis-server /etc/redis/redis.conf`

`$ redis-cli`

`127.0.0.1:6379>`

If you’re just using Redis for Sidekiq, you’re done. However, Redis is a pretty powerful datastore. For more, check out [Redis in Action](http://www.amazon.com/gp/product/1617290858/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1617290858&linkCode=as2&tag=mccblo-20&linkId=TQKIMJDHQC5UANZL).


#### Nginx

If you don’t know Nginx well. Don't be ashamed. Everyone was there at some point. 

Install Nginx:

`$ sudo apt-get install nginx`

`$ sudo service nginx start`

You should now be able to view Nginx’s welcome page by visiting your box public IP in your browser.

You can also check to make sure Nginx is running:

`$ ps ax | grep nginx`

Now we can get Nginx set up to point to our (upcoming) Rails install. Let’s open up the nginx.conf file:

`nano /etc/nginx/nginx.conf`

Here’s the config I ended up with: [nginx.conf](../master/nginx.conf)

Here’s my setup. I’m running SSL, so there are two server blocks. The “listen 80” block takes any non-http request and redirects to the other server block. The second block is for “listen 443” (SSL). If you don’t want/need SSL, you can remove the first server block and swap out 443 to 80 in the second block. You’ll want to remove the ssl_certificate lines, too.

`nano /etc/nginx/sites-enabled/default`

You can use my server block setting here:  [default](../master/default)


#### Unicorn

I’m going to assume you know what Unicorn is and why you should use it. 

Now we are ready to install Unicorn.

An easy way to do this is to add it to your application's Gemfile. Open the Gemfile in your favorite editor (make sure you are in your application's root directory):

`$ nano Gemfile`

At the end of the file, add the Unicorn gem with this line:

`gem 'unicorn'`

Save and exit.

To install Unicorn, and any outstanding dependencies, run Bundler:

`$ bundle`

Unicorn is now installed, but we need to configure it.

Here’s my [unicorn.rb](../master/unicorn.rb) Rails initializer (config/unicorn.rb)

#### MINA

We will use [Mina](http://nadarei.co/mina/). A really fast deployer and server automation tool

Add it to your Gemfile.

`gem 'mina'`

Let’s also add two other gems that will help us manage Unicorn and Sidekiq

`gem 'mina-sidekiq', :require => false`

`gem 'mina-unicorn', :require => false`

Create the necessary [“deploy.rb”](../master/deploy.rb):


`$ mina init`

Created config/deploy.rb.

Make sure to add your project’s folder inside ‘/home/deploy’. 

For instance, ‘/home/deploy/YOUR_APP’.

See [deploy.rb](../master/deploy.rb) file for example tweaked setup.


Ok, let’s run that setup task to create the necessary folders and files on your server. If one doesn’t get created that you need, no worries. Just SSH back in there and create the folder yourself.

`$ mina setup`

	-----> Creating folders... done.

#### Set Up database.yml and secrets.yml

Let’s get back on your box and set up your database.yml and secrets.yml files:

`$ ssh deploy@YOUR_IP`

`$ nano /home/deploy/YOUR_APP/shared/config/database.yml`

Tweak that file:

    production:
    	adapter: postgresql
    	encoding: unicode
    	database: APPNAME_production
    	username: postgres
    	password: DB_PASSWORD_SET_ABOVE
    	host: localhost


…and your secrets.yml…

`nano /home/deploy/YOUR_APP/shared/config/secrets.yml`

    production:
        secret_key_base: RUN 'rake secret' TO GENERATE A KEY

#### Deploying

And finally, deploy!

`$ mina deploy`

	-----> Deploying to 2015-10-04-040248
       ...
       Lots of things happening...
       ...
	-----> Done.


Note: In the future, when we grow and scale or multiply, for best practice we should use something like puppet/chef/ansible/salt, where we can create a recipe/manifest that able to provision many nodes on the fly. Where our manifest is portable.