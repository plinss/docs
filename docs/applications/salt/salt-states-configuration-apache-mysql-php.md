---
author:
    name: Joseph Dooley
    email: jdooley@linode.com
description: 'Use Salt States with a LAMP Stack and Fail2ban.'
keywords: 'salt, saltstack, states, apache, mysql, php, fail2ban, debian 8'
license: '[CC BY-ND 3.0](http://creativecommons.org/licenses/by-nd/3.0/us/)'
modified: Thursday, July 2nd, 2015
modified_by:
    name: Joseph Dooley
published: 'Thursday, July 2nd, 2015'
title: Salt States for Configuration of Apache, MySQL, and PHP (LAMP)
---

After Salt is <a href="/docs/networking/salt/install-salt" target="_blank">installed</a> and <a href="/docs/networking/salt/salt-states-apache-mysql-php-fail2ban" target="_blank">Salt States</a> are created, the Minion will need to be configured. This tutorial will configure a Minion's LAMP stack with further use of Salt States. This tutorial is written for Debian 8 but can easily be adjusted for other Linux Distributions.

##Create the LAMP Configuration States
The below steps configure all Salt Minions for a 1GB Linode, feel free to adjust as needed. 

1.  Open the `/etc/salt/base/top.sls` file and add the additional line:
 
    {:.file }
    /etc/salt/base/top.sls
    :  ~~~  
       base:
         '*':
            - lamp
            - extras
            - lampconf
       ~~~

2.  Create and edit the `/etc/salt/base/lampconf.sls` file:

    {:.file }
    /etc/salt/base/lampconf.sls
    :  ~~~
       #Apache Conguration for 1GB Linode
       /etc/apache2/apache2.conf-KA:
         file.replace:
           - name: /etc/apache2/apache2.conf
           - pattern: 'KeepAlive On'
           - repl: 'KeepAlive Off'
           - show_changes: True

       /etc/apache2/apache2.conf-IM:
         file.append:
           - name: /etc/apache2/apache2.conf
           - text: |
               <IfModule mpm_prefork_module>
               StartServers 2
               MinSpareServers 6
               MaxSpareServers 12
               MaxClients 30
               MaxRequestsPerChild 3000
               </IfModule>

       #MySQL Configuration for 1GB Linode
       /etc/mysql/my.cnf-br:
         file.blockreplace:
           - name: /etc/mysql/my.cnf
           - marker_start: '# * Fine Tuning'
           - marker_end: '# * Query Cache Configuration'
           - content: |
               #
               key_buffer             = 32M
               max_allowed_packet     = 1M
               thread_stack           = 128K
               thread_cache_size      = 8
               # This replaces the startup script and checks MyISAM tables if
               # needed the first time they are touched
               myisam-recover         = BACKUP
               max_connections        = 75
               table_cache            = 32
               #thread_concurrency    = 10
               #
           - backup: '.bak'
           - show_changes: True

       #PHP Configuration for 1GB Linode
       /etc/php5/apache2/php.ini-er:
         file.replace:
           - name: /etc/php5/apache2/php.ini
           - pattern: 'error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT'
           - repl: 'error_reporting = E_COMPILE_ERROR|E_RECOVERABLE_ERROR|E_ERROR|E_CORE_ERROR'
           - show_changes: True

       /etc/php5/apache2/php.ini-el:
         file.replace:
           - name: /etc/php5/apache2/php.ini
           - pattern: ';error_log = php_errors.log'
           - repl: 'error_log = /var/log/php/error.log'
           - show_changes: True

       /var/log/php/error.log:
         file.managed:
           - user: www-data
           - group: root
           - dir_mode: 755
           - file_mode: 644
           - makedirs: True

       #Restart
       apache2-run-at-boot-restart:
         service.running:
           - name: apache2
           - enable: True
           - watch:
             - pkg: apache2

       mysql-run-at-boot-restart:
         service.running:
           - name: mysql
           - enable: True
           - watch:
             - pkg: mysql-server
       ~~~

    The above file uses the <a href="http://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html" target="_blank">file</a> and <a href="http://docs.saltstack.com/en/latest/ref/states/all/salt.states.service.html" target="_blank">service</a> Salt State modules.


3.  Transfer the State settings to the Minions:

        salt '*' state.highstate

##Create Virtual Hosts Files
Salt State Modules are used for settings across groups of Minions. To adjust a configuration on a single Minion, try using Salt Execution Modules. Note, there are many ways to use Salt. 

1.  Disable the default Apache virtual host on either a single Minion or all Minions:

    For a specific Minion:

        salt '<hostname or Minion ID>' cmd.run "a2dissite *default"
    
    For all Minions:

        salt '*' cmd.run "a2dissite *default"


2.  Create directories for the website's files, logs, and backups. Replace `example.com` with the name of the website:

        salt '<hostname or Minion ID>' file.makedirs /var/www/example.com/pubic_html/
        salt '<hostname or Minion ID>' file.makedirs /var/www/example.com/log/
        salt '<hostname or Minion ID>' file.makedirs /var/www/example.com/backups/

3.  Create a directory on the Master to hold all of the Minion virtual host files. This directory can act as an index for all of the Minion websites. 

        mkdir /etc/salt/base/minionsites

4.  Create the `/etc/salt/base/minionsites/example.com.conf` vhost file for the specified Minion. Replace `example.com` throughout and in the following commands.

    {:.file }
    /etc/salt/base/minionsites/example.com.conf
    :  ~~~  
       # domain: example.com
       # public: /var/www/example.com/public_html/

       <VirtualHost *:80>
         # Admin email, Server Name (domain name), and any aliases
         ServerAdmin webmaster@example.com
         ServerName  www.example.com
         ServerAlias example.com

         # Index file and Document Root (where the public files are located)
         DirectoryIndex index.html index.php
         DocumentRoot /var/www/example.com/public_html
         # Log file locations
         LogLevel warn
         ErrorLog  /var/www/example.com/log/error.log
         CustomLog /var/www/example.com/log/access.log combined
       </VirtualHost>
       ~~~

5.  Copy the vhost file from the Master to the `/sites-available` directory of the Minion:

        salt-cp '<hostname or Minion ID>' /etc/salt/base/minionsites/example.com.conf /etc/apache2/sites-available/example.com.conf

6.  Enable the new website and restart Apache:

        salt '<hostname or Minion ID>' cmd.run "a2ensite example.com.conf"
        salt '<hostname or Minion ID>' cmd.run "service apache2 reload"

<a href="/docs/networking/salt/salt-states-configuration-apache-mysql-php" target="_blank">


The above section used the <a href="http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.cmdmod.html" target="_blank">cmdmod</a>, <a href="http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.file.html" target="_blank">file</a>, and <a href="http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.cp.html" target="_blank"> cp</a> Salt Execution modules.

You should now have a configured LAMP stack across as many Minions as you wanted. Optionally, use [grains](http://docs.saltstack.com/en/latest/topics/targeting/grains.html) for further customization and to apply specific variables to each host.


