# B.U.R.P.S.
## Building An Ubuntu Rails Production Server

Michael Beyer, 04-2016, rev. 1.04
<br/>
<michael.beyer@gesis.org>
<br/>
<mgbeyer@gmx.de>

<br/>

**Change notes:**
* 2016-05-17: Added “Advanced logfile configuration” and information about log levels
* 2016-04-21: Added “How to deploy multiple apps via sub-URI”

<br/>

## Preface

This is a condensed step-by-step tutorial to guide you through the 
process of setting up a production-server environment for one or multiple
Ruby on Rails application(s) with a clear focus on C- / Matz-Ruby. In
general it deals with all issues concerning a runtime-environment from
the basic Linux server-setup via the database server on basis of MySQL
through to a proper container setup with Passenger and Nginx. In
particular it is also meant to cover all necessary steps to deploy, run
and maintain multiple instances of the Open Source thesaurus management
system “*iQvoc*”. But most of the topics covered here can be considered
generally valid standard practice as well.

**What to expect… and what not to expect**

What you can expect is a condensed but detailed and complete path to
show you *WHAT* needs to be done and *HOW* and *WHEN* it needs to be
done. What you can’t expect (and what is beyond the scope of this little
tutorial) is an answer to the question *WHY* things are done in a
particular way and what the alternatives might be. What you can’t this
tutorial expect to cover either is basic knowledge of general crossover
technologies like Linux, Git, SSH, MySQL or the Ruby on Rails framework.
It is up to you to fill in any such knowledge gaps beforehand.

<br/>

## The Server

Production (as well as development) run Ubuntu 14.04 LTS (Trusty Tahr)
Linux. There’s a typical LAMP environment for practical purposes besides
a specialized Phusion Nginx/Passenger setup as a container for the Rails
apps.

Before you install additional packages over the course of this tutorial,
make sure all existing packages are up-to-date: sudo apt-get update

For general convenience it is recommended to install a text-based UI
style filemanager like “mc” (Midnight Commander) with console mouse
support via “gpm” (easy copy+paste). But this is of course not
mandatory.

**Database server basic setup (MySQL):**

	User for iQvoc: name= iqvoc, server= localhost
	Database privileges: iqvoc\_% all privileges, no GRANT
	Global privileges: [data= select, insert, update, delete], [structure= create, alter, index, drop, create temporary tables]
*We’re using the “phpMyAdmin” web frontend for easy database maintenance
just in case (sudo apt-get install phpmyadmin):*

In case “mcrypt” is missing, check if the php5 extension is installed
AND activated (sudo php5enmod mcrypt).

*Safety considerations:*

For Apache restrict the IP range and disable directory listings:

	/etc/apache2/apache2.conf (in the corresponding “Directory” sections):
	Require ip 10.6
	Options -Indexes

	/etc/apache2/site-available/*.conf (e.g. “000-default.conf”, in the corresponding “Location” section):
	Deny from all
	Allow from 10.6

For phpMyAdmin change the default alias (e.g. name it “dbadm”) and restrict the IP range:

	/etc/phpmyadmin/apache.conf:
	Alias /dbadm /usr/share/phpmyadmin
	Deny from all
	Allow from 10.6

*Other considerations:*

For Apache we’ll change the standard port (80) it listens to. We’re
using standard ports (80, 81, …) later for Nginx to serve our
applications. You might want to change this to 8080 for example:

	/etc/apache2/ports.conf:
	Listen 8080

	/etc/apache2/site-available/\*.conf (e.g. “000-default.conf”):
	<VirtualHost *8080>

*Rails preparation:*

In order for the “mysql2” gem to install and function properly later you
have to install the “libmysqlclient-dev” package (as root user).

**User for deployment**

First you want to configure a dedicated deployment user (which is simply
named “deploy” in this scenario) and setup appropriate SSH
authentification via keys. Make sure you are able to connect to the
server as “deploy” without the need to enter a password. Make sure your
private key has no passphrase set.

**Copy your public key over to the server**

*The easiest way is to copy it over to root first and then change
ownership:*

	mkdir -p /home/deploy/.ssh
	mv key /home/deploy/.ssh/authorized\_keys
	chown -R deploy /home/deploy/.ssh
	chmod 600 /home/deploy/.ssh/authorized\_keys

The next step is to create a SSH keypair for the deployment user for
Git(Lab) access. Copy the public key to the Git(Lab) server as a deploy
key. You might want to consider SSH agent forwarding for login as the
deploy user, so you could omit this step. But this makes potential
deployment automation (e.g. via “Capistrano”) much more complex and
error-prone, so we won’t do this here.

**JavaScript runtime:**

Install the “node.js” package. This will be used by Rails later in order
to have the corresponding rake task properly precompile JavaScript
assets for production mode (e.g. “minify” JS code):

	sudo apt-get install nodejs

<br/>
	
## Ruby and Rails installation

**The following installation tasks must be done while logged in as the
deployment user.**

First there are some basic prerequisites to consider. Install the
following dependencies for Ruby:

	sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev

Next we will install the Ruby version manager “rbenv” and use it to
install Ruby (here: v. 2.2.4):

	git clone git://github.com/sstephenson/rbenv.git .rbenv
	echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
	echo 'eval "$(rbenv init -)"' >> ~/.bashrc
	exec $SHELL

	git clone git://github.com/sstephenson/ruby-build.git
	~/.rbenv/plugins/ruby-build
	echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
	exec $SHELL

	git clone https://github.com/sstephenson/rbenv-gem-rehash.git
	~/.rbenv/plugins/rbenv-gem-rehash

	rbenv install 2.2.4
	rbenv global 2.2.4   # global default version

Now we tell Ruby gems not to install the documentation for each package
locally (we don’t want this in production) and then install Bundler (the
Ruby dependency resolver and package (gem) manager):

	echo "gem: --no-ri --no-rdoc" > ~/.gemrc
	gem install bundler

Now it’s time to install Rails (here: version 4.2.5) and make the Rails
executable available to rbenv:

	gem install rails –v 4.2.5
	rbenv rehash

<br/>

## Passenger and Nginx installation

In this section we will install Passenger and Nginx via Phusion’s APT
repository. This is the easiest way to have a proper container setup
without the need to compile Nginx yourself. This package will install a
version of Ngnix with an already compiled-in Passenger module.

Here we will install the “full” flavor of Nginx. It’s the most universal
repository with all the core modules and several extended capabilities
from third-party modules in place. But in contrast to the “extras”
version it does not contain heavy-weight and rarely used modules we
don’t need here.

	# Install Phusion’s PGP key and add HTTPS support for APT
	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
	sudo apt-get install -y apt-transport-https ca-certificates

	# Add Phusion’s APT repository
	sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main > /etc/apt/sources.list.d/passenger.list'
	sudo apt-get update

	# Install Passenger + Nginx ("full" version)
	sudo apt-get install -y nginx-full passenger

**Configuration**

Include or uncomment the following lines in your Nginx config. file
„/etc/nginx/nginx.conf”, in the http section:

	passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
	passenger_ruby /home/deploy/.rbenv/shims/ruby; # if you use rbenv

**Configuration approach one: Multiple apps runs under different
sub-domains**

Configure virtual hosts for your applications under
“/etc/nginx/sites-enabled/\*”. For simplicity you might want to just
edit “default” for all your apps. For our scenario (each app accessible
under its individual sub-domain, all listening to the same port) we use
one separate ” server” section for each app. Or you also can use one
dedicated config. file per app (like
“/etc/nginx/sites-enabled/myapp.conf”):

	server {
		listen 80; # or whatever port you want your app(s) to run under
		listen [::]:80;
		server_name www.yourhost.com;
		root /home/deploy/rep/appname/public; # be sure to point to 'public'!
		passenger_enabled on; # this will connect nginx and passenger
		passenger_ruby /path-to-ruby-interpreter; # only if the app needs a different or specific version
		rails_env production;
		error_page 500 502 503 504 /50x.html;
		location /50x.html {
			root html;
		}
	}

Make sure to delete (or comment-out) any pre-defined “location”
section(s).

**Note**: To test your new configuration based on the sub-domain
approach, make sure the sub-domain is known to your network’s DNS
servers. If this is (not yet) the case, you have to “simulate” DNS
resolution and add the subdomain to your local host file accordingly.

**Alternative configuration approach two: Deploying multiple apps via sub-URI**

In the section above we described, how to deploy multiple applications
on different sub-domains. This is the most straight-forward way in terms
of configuration and therefore recommended. However there might be
situations where the requirement of the project dictates that each app
is accessible via sub-URI (or subdirectory) in the form
“www.lorem.ipsum/myapplication/”.

For this approach there is no need for multiple “server” sections. In
your central configuration file “/etc/nginx/sites-enabled/default” add
the following “location” section inside the main “server” section. Here
“suburi\_name” means the portion of the URL under which your application
should be accessible (like: “www.lorem.ipsum/suburi\_name/”) and
“app\_name” is the name of the folder your application is located
physically in the file-system:

	location ~ ^/suburi\_name(/.*|$) {
		alias /home/deploy/rep/app_name/public$1;
		passenger_base_uri /suburi_name;
		passenger_app_root /home/deploy/rep/app_name;
		passenger_document_root /home/deploy/rep/app_name/public;
		passenger_enabled on;
		rails_env production;
		passenger_env_var RAILS_RELATIVE_URL_ROOT /suburi_name;
	}

This tells Ngnix how to handle requests to the app itself. If you want
the Rails asset pipeline to serve requests to static files in the
“public/” directory directly (via Sprockets middleware) instead of Nginx
(this would be “config.serve\_static\_files = true” in the config.
file “/config/environments/production.rb”), we need another “location”
section to address this behavior for sub-URIs:

	location ~ ^/suburi_name/assets(/.*|$) {
		alias /home/deploy/rep/app_name/public/assets$1;
	}

To make serving assets from a relative sub-URI work generally you
(later) have to pre-compile your assets as follows (for further
information about the Rails asset-pipeline see next section: “iQvoc
application setup”):

	bundle exec rake assets:precompile RAILS_ENV=production RAILS_RELATIVE_URL_ROOT=/suburi_name

This will make the pre-compiler rake task prefix your asset paths with
“/suburi\_name/assets/” instead of an absolute “/assets/”.

When you are done, restart Nginx:

	sudo service nginx restart

Passenger (as enabled and configured in your configuration) will now
take care that your applications will be served and run under the user
account specified by the owner of “config.ru” in your app’s root
directory (based on the “convention-over-configuration” principle).

**Advanced logfile configuration**

Until now we left the logfile settings to its default. In this section we’ll have a deeper look into more advanced aspects of Nginx/Passenger logging.
Most of the following configuration entries need to be included in your main Nginx config. file „/etc/nginx/nginx.conf”.

**Anonymized IP addresses**

In Germany you need to anonymize ip address where ever you work with such data. This also applies for webserver logs, such as Nginx. The following snippet will take care of the anonymization of IP4 and IP6 addresses. It was taken from http://www.mbr-is.com/central-anonymized-ip-addresses-for-nginx-for-all-virtual-hosts/. 

	map $remote_addr $ip_anonym1 {
	 default 0.0.0;
	 "~(?P<ip>(\d+)\.(\d+)\.(\d+))\.\d+" $ip;
	 "~(?P<ip>[^:]+:[^:]+):" $ip;
	}
	map $remote_addr $ip_anonym2 {
	 default .0;
	 "~(?P<ip>(\d+)\.(\d+)\.(\d+))\.\d+" .0;
	 "~(?P<ip>[^:]+:[^:]+):" ::;
	}
	map $ip_anonym1$ip_anonym2 $ip_anonymized {
	 default 0.0.0.0;
	 "~(?P<ip>.*)" $ip;
	}
	log_format anonymized '$ip_anonymized - $remote_user [$time_local] ' 
	   '"$request" $status $body_bytes_sent ' 
	   '"$http_referer" "$http_user_agent"';
	access_log /var/log/nginx/access.log anonymized;

For reasons of clarity you might want to separate this from your main configuration and put it in a new config-file (e.g. “/etc/nginx/logging.conf”) so you can include it in the main config-file (and comment-out existing logging directives beforehand). Make sure file permissions and ownership are set analogous to the main config-file.

**Logfile rotation**

This is already set-up for you under Ubuntu via “Logrotate”. But you might want to change the defaults. In case of Nginx, Logrotate puts a config file into the “/etc/logrotate.d” directory named “nginx”. 

For testing or debugging a specific configuration you can invoke Logrotate manually:

	/usr/sbin/logrotate –vdf /etc/logrotate.d/nginx   # verbose, debug, force

Additional clues can be found in a file where Logrotate stores information about when it last rotated each logfile: “/var/lib/logrotate/status”.

For more detailed information about “Logrotate” please see appendix at the end of this document.
	 
**Log level**

You might want to consider lowering the log level of your error logfile suitable for production in your central „/etc/nginx/nginx.conf”:

	error_log /var/log/nginx/error.log warn;

<br/>

## iQvoc application setup
**(for general information see “<http://iqvoc.net/>”)**

1.  **Git(Lab) setup**: Within your deployment-user’s home directory or
    relative (e.g. “/home/deploy/rep/”) and logged in as the
    deployment-user, fetch your (corr.) app from your repository (we’re
    using GESIS GitLab server here):

	* git clone your-project-url.git
	  app\_name (use the git url as specified in your app’s Gemfile)

	* “git clone” will create the directory “app\_name” for you, pull
	  the application’s master branch to it and it will perform an
	  implicit “git init” for the directory as well as a “git remote
	  add” for the corr. Git repository’s master branch.

	* At this point you might want to consider some global
	  configuration options for git in general (git config --global),
	  like "core.editor", "user.name" and "user.email"

2.  **database credentials**: configure the database for production via
    “config/database.yml”

3.  **secret token**: Make sure you have “config/secrets.yml” in place
    and properly set-up for production (use “rake secret” to generate a
    new token).

4.  **security precautions**: If you keep your database credentials and
    the secret token in their respective configuration files (as done
    above), it is imperative for security reasons those file won’t be
    included in your Git(Lab) repository (.gitignore). There are other
    ways to address this subject on basis of environment variables (if
    you use rbenv for Ruby versioning you might for example consider the
    “rbenv-vars” plugin). In order to maintain a clean and simple setup
    we stick to the usual config. files here.

5.  **bundle / gem installation**: Run “bundle install”. If you want to
    omit gems solely used during development or testing, use “--without
    development test” parameter.\
    Common pitfall: If development takes place on a Windows machine and
    so there are platform specific binaries for certain gems,
    “Gemfile.lock” need NOT to be included in the git repository. Let
    bundler go through the whole process on the production server
    from scratch. A fresh Gemfile.lock will be created automatically.\
    In case of an application update you might want to initiate bundle
    update (or bundle update --source gemname to update just a single
    gem according to its :git source in the Gemfile, e.g. in case only
    parts of the iQvoc engine have changed).

6.  **database setup**: Create, migrate and (optionally) seed the
    database:

	* bundle exec rake db:create RAILS\_ENV=production

	* bundle exec rake db:migrate RAILS\_ENV=production

	* optional step: bundle exec rake db:seed RAILS\_ENV=production

	* consider your time zone: Make sure the proper time-zone is
	  configured (config/application.rb) for your app and
	  active-record:\
	  config.time\_zone = ‘Europe/Berlin’\
	  config.active\_record.default\_timezone = :local

7.  **asset pipeline**:

	* in “/config/environments/production.rb” of your application set\
	  config.assets.precompile += \['\*.js', '\*.css', '\*.css.erb', '\*.css.scss'\]\
	  config.serve\_static\_files = false

	* prepare static assets for production: bundle exec rake
	  assets:precompile RAILS\_ENV=production\
	  Note: If your applications are deployed under different
	  sub-URIs (subdirectories) you have to pre-compile your assets
	  with the “RAILS\_RELATIVE\_URL\_ROOT=/suburi\_name” parameter.

	* clean-up static assets for production in case of an update
	  (prior to the previous step): bundle exec rake assets:clean
	  RAILS\_ENV=production (you might want to purge the whole asset
	  cache completely by initiating bundle exec rake
	  assets:clobber RAILS\_ENV=production). Don’t forget to commence
	  a “precompile” afterwards.

	* note: for simple WEBrick testing of production mode:\
	  config.serve\_static\_files = true\
	  config.assets.digest = true

8.  **background jobs**: Some features as „import“, „export“ or reverse
    mapping relations (backlinks) for federated concept mappings store
    their workload as asynchronous jobs. You can utilize the one-off
    jobworker via its rake task and schedule it to periodically run via
    a cron job for the deployment user. When logged in as deploy run
    “crontab -e” (or run “sudo crontab –e –u deploy” as root). For
    example you might want to run the task every 4 hours, on the hour:\
	0 \*/4 \* \* \* export
    PATH=/home/deploy/.rbenv/shims:/home/deploy/.rbenv/bin:/usr/bin:\$PATH;
    eval "\$(rbenv init -)"; cd /home/deploy/rep/appname && bundle exec
    rake jobs:workoff RAILS\_ENV=production\
    Expanding the path and rbenv initialization are necessary because
    cron executes under a limited environment.\
    You might want to send all output to nirvana
    (“&gt;/dev/null 2&gt;&1”) in order to keep your syslog clean if you
    have no MTA (mail service) installed (which by default is the case
    with Ubuntu).

9.  **importing SKOS data**: You might want to initially import data in
    N-Triples format. To obtain N-Triples data
    from SKOS (XML) format you can use freely available converter tools
    like “Raptor” (Linux) or “rdfconvert” (Windows).\
    rake iqvoc:import:url URL=path/to/your_file.nt NAMESPACE=’http://your_namespace/’ RAILS\_ENV=production &gt; data/iqvoc\_import\_output.txt
	
10.	 **Consider your log level in production mode**: Keep in mind that per default Rails logs pretty verbose and so might 	bloat your application logfile. You should edit your “config/environments/production.rb” accordingly (e.g. add “config.log_level = :warn”).

<br/>
	
## Managing the application container (Passenger)

**Restarting an application**

If you run “touch tmp/restart.txt” from your rails root directory,
Passenger will restart the app. You shouldn't have to restart Nginx.
After the timestamp of the “restart.txt” file changes, Passenger will
restart for the next request. If your app takes a while to boot, you may
want to force this by making a request immediately after touching the
file.

You don't need to worry about kicking someone off the site, it won't
restart the server if there is a request in process.

An application restart also can be done very convenient with the
passenger configuration tool in interactive mode: passenger-config
restart-app. Unlike touching “restart.txt” this method will result in
the application being restarted immediately.

**Monitoring**

Use “passenger-config list-instances” or “passenger-status” to get an
overview on your running application(s). The option “--show=requests”
provides detailed information about active requests (routing state).

**Logfiles**

By default and if not configured otherwise, Passenger logs to global
Nginx log files, usually located in “/var/log/nginx/”. The application
itself logs to standard Rails compliant logfiles based on the current
environment (log/production.log).\
If you want to change the log-level or the logfile location, you can configure this in the http section of your central Nginx config. file „/etc/nginx/nginx.conf”:

	passenger_log_level 2;   # warn
	passenger_log_file path_to_nginx_access_log;

<br/>

## Managing the web server (Nginx)

Here are some basic management commands.

	sudo service nginx stop
	sudo service nginx start
	sudo service nginx restart

We can make sure that our web server will restart automatically when the
server is rebooted by typing:

	sudo update-rc.d nginx defaults

This should already be enabled by default, so you may see a message like
this:

	System start/stop links for /etc/init.d/nginx already exist.

This just means that it was already configured correctly and that no
action was necessary. Either way, your Nginx service is now configured
to start up at boot time and so will be your applications.

<br/>

## Database hot backup with AutoMySQLBackup

Install as root, then run the install-script:

	sudo apt-get install automysqlbackup
	automysqlbackup

The installation process will install a script at
“/usr/sbin/automysqlbackup” and a configuration file at
“/etc/default/automysqlbackup”.

Once installed, the script will automatically run once a day (per
default). Backups will be stored in the directory
“/var/lib/automysqlbackup”.

You need to setup a backup user for MySQL with the following
permissions:

	global data privileges: SELECT
	global administration privileges: RELOAD, SHOW DATABASES, LOCK TABLES

You might want to configure the following parameters:

	USERNAME=your_mysql_backup_user
	PASSWORD=mysql_backup_user_password
	DBHOST=localhost
	BACKUPDIR=other_than_default_location
	…

There are several other options, for more information see “man
automysqlbackup”. Usually the defaults for backup schedule (frequency
and rotation) are fine. For details see the section below.

If you do put your password in the configuration file, you may want
issue a

	sudo chmod 600 /etc/default/automysqlbackup

to only allow access to this file by the root user on the server.

AutoMySQLBackup will backup all of your databases and their tables, will
compress these backups with gzip (bzip2 is also an option), and then
organize them into three folders stored in “/var/lib/automysqlbackup/*”*
– daily, weekly, monthly. You can change where these folders are
created by editing the “BACKUPDIR” configuration variable. Each of these
folders will contain a folder for each database on your system.

By default, the *daily* folder will contain all of the last seven days.
The *weekly* folder will grow to contain the database as it was on
Sunday each of the last fifty-two weeks. Similarly, *monthly* will
contain the end of all each of the last twelve months.

If you only want to back up certain databases, you can specify them in
the “DBNAMES” configuration variable. Conversely, if you want to backup
everything except certain databases, you can use the “DBEXCLUDE”
configuration variable to list what to exclude.

**Scheduling**

AutoMySQLBackup will add itself to “/etc/cron.daily/automysqlbackup”
during installation and will so run automatically once a day at the time
specified in the global “/etc/crontab”.

**Restoring a backup**

1)  gunzip \*.sql.gz backup file

2)  copy \*.sql file to temp location

3)  set proper file permissions for the mysql command, like “sudo chmod
    644”

4)  either make sure the desired database exists or omit the DATABASE
    parameter in the following command!

5)  sudo mysql –uroot –p your\_database &lt; your\_database.sql

<br/>

## Further reading and sources of information

**Installing Passenger + Nginx on Ubuntu 14.04 LTS**

<https://www.phusionpassenger.com/library/install/nginx/install/oss/trusty/>

**Deploying a Ruby app on a Linux/Unix production server with Passenger in Nginx mode on Ubuntu 14.04 LTS**

<https://www.phusionpassenger.com/library/walkthroughs/deploy/ruby/ownserver/nginx/oss/trusty/deploy_app.html>

**Deploy Ruby On Rails on Ubuntu 14.04 Trusty Tahr - A guide to setting up a Ruby on Rails production environment**
Just a useful example of many other tutorials on the web.

<https://gorails.com/deploy/ubuntu/14.04>

**The Rails asset pipeline**

<http://guides.rubyonrails.org/asset_pipeline.html>

**iQvoc on GitHub**

<https://github.com/innoq/iqvoc>

**iQvoc Wiki**

<https://github.com/innoq/iqvoc/wiki>

**rbenv on GitHub**

<https://github.com/rbenv/rbenv>

**rdfconvert**
Windows tool to convert RDF/XML data to N-Triples format.

<https://sourceforge.net/projects/rdfconvert/>

**AutoMySQLBackup**

<https://sourceforge.net/projects/automysqlbackup/>

**MySQLTuner-perl on GitHub**
Perl script to give a neat and condensed overview of your database server configuration and metrics. The current configuration variables and status data are retrieved and presented in a brief format along with some basic suggestions for performance and stability.

<https://github.com/major/MySQLTuner-perl>

**MySQL Tuning Primer Script**
A similar script (bash) as the one mentioned above.

<https://launchpad.net/mysql-tuning-primer>

**Managing logs with Logrotate**

<https://serversforhackers.com/managing-logs-with-logrotate>

**Understanding Logrotate on Ubuntu**

<http://articles.slicehost.com/2010/6/30/understanding-logrotate-on-ubuntu-part-1>
<br/>
<http://articles.slicehost.com/2010/6/30/understanding-logrotate-on-ubuntu-part-2>
