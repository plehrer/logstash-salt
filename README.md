logstash-salt
=============

A Logstash-Elasticsearch-Kibana stack built with Vagrant and Salt


Installation
============

- Install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
- Install [Vagrant](http://www.vagrantup.com/)
- Install [Salty Vagrant](https://github.com/saltstack/salty-vagrant): `vagrant plugin install vagrant-salt`
- `vagrant up` (This will take a while; Vagrant will download and add the Virtualbox, then Salt will bootstrap and configure the machine)
- `vagrant ssh` to access the machine

Running the Logstash stack
==========================

- From the machine's CLI, `cd /vagrant`
- Download the Logstash [jar](https://download.elasticsearch.org/logstash/logstash/logstash-1.2.2-flatjar.jar) 
  and [sample Apache log file](http://logstash.net/docs/1.2.2/tutorials/10-minute-walkthrough/apache_log.2.bz2)
- Expand the sample log: `bunzip2 apache_log.2.bz2`
- Run Logstash: `java -jar logstash-1.2.2-flatjar.jar agent -f apache-elasticsearch.conf`
- From a separate command line, feed the Apache log to logstash: `nc localhost 3333 < apache_log.2`
- When this is finished, try loading Kibana at [http://localhost:15001/index.html](http://localhost:15001/index.html#/dashboard/file/logstash.json)

Configuring Logstash and Elasticsearch
======================================

I got the central server to work and was able to ship an event from a remote host to the central server and have the results show up in
Kibana as well as a log file on the vagrant central server. I first installed the broker, redis-server on the vagrant instance.
I configured it to bind to ip 10.0.2.15 and to listen on port 6379 which is the default.
The bind ip default is 127.0.0.1 and I commented this out.

I created directories /etc/logstash and /var/log/logstash on the vagrant instance to store the config file central.conf and the log file central.log.

/etc/central.conf:

`#This file should be copied to /etc/logstash`

input {

  redis {

    host => "127.0.0.1"

    type => "redis-input"

    data_type => "list"

    key => "logstash"

  }

}

output {

  stdout { }

  elasticsearch {

    cluster => "logstash"

  }

}

- redis server config file /etc/redis.conf
- It is a large file. See ~/logstash-salt/redis/redis.conf
I changed:
`bind 127.0.0.1` to
`#bind 127.0.0.1`
and added line:
`bind 10.0.2.15`

This was very important. Without getting the bind address the broker, redis, will not work.

- The logstash instance on vagrant was created with the command:
`sudo java -jar logstash-1.2.2-flatjar.jar agent -v -f /etc/logstash/central.conf -l /var/log/logstash/central.log`

The book has an alternative method for creating the instance and running elastic search as a separate instance.
Elastic search runs without creating a separate instance by adding this command in central.conf output section:
`embedded => true`

Thus it would be:

output {

  stdout { }

  elasticsearch {

    cluster => "logstash"

    embedded => true

  }

}

- The book provides an [logstash init script for Ubuntu] (http://logstashbook.com/code/3/logstash-central-init) running logstash as a service the central server.
- [init script for Centos] (http://logstashbook.com/code/3/logstash-central.init)
This files has been included in the repository

ElasticSearch
=============

- I created a yaml file called elasticsearch.yml which currently resides in the same directory as the logstash jar file.
- This is an elasticsearch configuration file.

cluster.name: logstash

node.name: "precise64"

- The node name is the server's name.


- If elastic search is set up as a service, then the file should be in directory: /etc/elasticsearch/

 Logstash Setup on Shipper Servers
 =================================
 Check chapter *Shipping Events*


 shipper.conf

 `#configuration file for remote host to ship events to the central server`

 input {

   file {

     # type => "syslog"

     type => "apache"

     # path => ["/var/log", "/var/asl", "/var/log/messages"]

     path => [ "/var/log/apache2/*" ]

     exclude => ["*.gz", "shipper.log"]

   }

 }


 output {

   stdout { codec => rubydebug }

   redis {

     host => "127.0.0.1"

     data_type => "list"

     key => "logstash"

   }

 }

 I changed the type to apache and the path to the apache log files so I can see events triggered from my local apaches web server.  To see this, go to any local website and click on links, load pages, etc and it should ship the event.

 The logstash instance on the remote (my local OS X machine) host was created with the line:
 `sudo java -jar logstash-1.2.3.dev-flatjar.jar agent -f /etc/logstash/shipper.conf -l /var/log/logstash/shipper.log`

 Note you have to specify the configuration file and log file in a similar fashion as on the central server (the vagrant instance).  There is an alternative to this which is specified. Check chapter “Shipping events”

- [init script for running logstash as a service on Ubuntu] (http://logstashbook.com/code/3/logstash-agent-init)
-  [init script for running logstash as a service on Centos] (http://logstashbook.com/code/3/logstash-agent.init)
- This files has been included in the repository

Installing logstash and drupal syslog module running on shipper vagrant instance on blackangus
==============================================================================================

Pre-Installation
----------------

- `sudo su - jenkins`
- `ln -s dosomething-vagrant dosomething-qa`
- `cd dosomething-qa/`
- `vagrant status` (Check if vagrant is running)
- `vagrant ssh`
- `cd /vagrant/html` (Move to Drupal directory)
- `sudo -h` (super user)

Enable syslog module
--------------------

- `drush pm-list | grep log` (check if syslog module is enabled)
- `drush pm-enable syslog` (enable module)

Install Logstash
----------------

- `mkdir /opt/logstash`
- `cd /opt/logstash`


- Get logstash: `wget https://download.elasticsearch.org/logstash/logstash/logstash-1.3.2-flatjar.jar -O logstash.jar`

- `cd /etc/init.d`
- Install agent init script for Ubuntu: `wget http://logstashbook.com/code/3/logstash-agent-init -O logstash-agent`

- `chmod 755 /etc/init.d/logstash-agent`
- `chown root:root /etc/init.d/logstash-agent`
- `update-rc.d logstash-agent enable`

- `mkdir /etc/logstash`
- `touch /etc/logstash/shipper.conf`
- `vim shipper.conf`

Insert into shipper.conf

input {

  file {

    type => "syslog"

    path => ["/var/log/syslog*"]

    exclude => ["*.gz"]

  }

}

output {

  stdout { codec => rubydebug }

  redis {

    host => "blackangus.dosomething.org"

    port => 9379

    data_type => "list"

    key => "logstash"

  }

}

Start logstash
--------------

- `/etc/init.d/logstash-agent start`