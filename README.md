Elasticsearch Workshop
======================

What is it?
-----------

This is an ansible playbook to create elasticsearch cluster on few
vanilla Ubuntu 14.04 machines (e.g. the ones you can provision at
digital ocean). The cluster can be useful for experimentation or, with
some tweeking to your particular use case even in production.

Background
----------

On 26.03.2015 I'll conduct a workshop on Elasticsearch at Poznań
University of Technology, i.e. my *Alma Matter*. In order to avoid
making students install Elasticsearch on their own laptops I've
created a cluster on [digital ocean][do].

Here is the ansible playbook to easily recreate it.

[do]: https://www.digitalocean.com/?refcode=649b18bb5e59

If you followed my blog you might have known than I'm responsible for
changing my employers in-house search appliance to a new one backed by
Elasticsearch. We had a huge data volume, even though we are not fully
migrated to Elasticsearch and during our day to day operations we see
a lot of interesting cases. I think we reached that point when reading
the documentation, news groups and blogs is not enough to explain
everything and I often feel a need to experiment with an Elasticsearch
cluster. With this playbook it is dead-easy to provision one in a
matter of minutes. Then you can play and break it and then quickly
reprovision.

Cluster architecture
--------------------

* HAProxy in front of an Elasticsearch cluster.
* 1 machine for haproxy (this can be the same box as one of the other
  nodes)
* 1+ machine for elasticsearch nodes; all nodes play all the roles -
  master, client and data
* haproxy accesses the nodes via http protocol on :9200
* the nodes form a cluster and communicate via the native protocol on
  :9300
* all the ports are blocked both from the public internet and the
  private network with the exceptions of:
    * 9300 and 9200 on the private network, they are used to for m a
      cluster and to accept the requests from haproxy
    * 80, 11936 are open for the public access on haproxy node
    * 22 is open everywhere

Prerequisites
-------------

1+ vanilla Ubuntu 14.04 server (it'll probably work with other
Ubuntus, but I've tested it on 14.04).

You need to know IP addresses of the machines in the cluster and have
a root access there (or a sudo user access).

I provisioned it in digital ocean, in the Amsterdam 3 region, with
*private networking* enabled. I've had 3 Ubuntu 14.04 droplets with
1GB ram each. The first node hosted both an ES node and a haproxy.

The playbook is smart enough to calculate number of nodes required to
elect a master avoid [split-brain][split-brain] problem. It also sets
the memory which Elasticsearch will use to be half of the system total
RAM. This is consistent with the [recommendations][es-memory].

[split-brain]: http://blog.trifork.com/2013/10/24/how-to-avoid-the-split-brain-problem-in-elasticsearch/
[es-memory]: www.elastic.co/guide/en/elasticsearch/reference/master/setup-configuration.html#setup-configuration-memory


Deploying the cluster
---------------------

1. Clone the repo
2. Create some Ubuntu 14.04 hosts to put the cluster on
3. Fill in the host details:

        $ cp inventory.in inventory
        $ vim inventory

4. Set the passwords

        $ cp secrets.yml.in secrets.yml
        $ vim secrets.yml

5. Play!

        $ ansible-playbook -i inventory es-workshop.yml

   If you use a use a sudo user you should do this instead

        $ ansible-playbook -i inventory -u <your-user> -s es-workshop.yml

    In digital ocean machines eth0 is the public ip and eth1 is the
    private (within datacenter) interface. In the playbook I use
    private networking to form a cluster, but your configuration may
    be different. You may want to let the nodes talk over the public
    internet (if you do experiments, this is not good for production)
    or simply eth0 maybe your private adapter. Then add ` --extra-vars
    "private_network_interface: ansible_eth0"` to the commands you
    run, e.g.

        $ ansible-playbook -i inventory --extra-vars "private_network_interface: ansible_eth0" es-workshop.yml

You cluster is accessible via the proxy machine

    $ curl -u "user:pass" http://proxy-machine
    {
        "status" : 200,
        "name" : "es-workshop-1",
        "cluster_name" : "es-workshop",
        "version" : {
            "number" : "1.4.4",
            "build_hash" : "c88f77ffc81301dfa9dfd81ca2232f09588bd512",
            "build_timestamp" : "2015-02-19T13:05:36Z",
            "build_snapshot" : false,
            "lucene_version" : "4.10.3"
        },
        "tagline" : "You Know, for Search"
    }


You can also access haproxy stats, just go to http://188.166.17.197:11936/

# Not production ready!

This cluster is for workshops/experiments. In real world you need to at least:

* set up https connection to avoid sending passwords in plaintext
* separate data nodes from clients from masters
* fine-tune Elasticsearch config to your particular use case
* [rolling restart][rolling-restart] to perform the upgrades or
  configuration changes while keeping the cluster on-line
* backups

[rolling-restart]: http://www.elastic.co/guide/en/elasticsearch/guide/current/_rolling_restarts.html

Why ansible?
------------



Ansible Playbooks
-----------------

I've got some inspiration for my ansible playbook when looking at these repos:

- https://github.com/Servers-for-Hackers
- https://github.com/snowplow/ansible-playbooks/
- https://github.com/Traackr/ansible-elasticsearch
