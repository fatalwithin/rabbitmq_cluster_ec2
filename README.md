# Ansible Role: RabbitMQ Cluster

Table of Content:
<!-- MarkdownTOC -->

- Ansible Role: RabbitMQ Cluster
- Requirements
- Must-have Role Variables
- Optional Role Variables and Defaults
- Example
- License
- Author Information

<!-- /MarkdownTOC -->

An Ansible Role, that installs a RabbitMQ 3-node cluster in 3 availability zones across your AWS region.

This project is inspired by `alexey-medvedchikov/ansible-rabbitmq`.

**It supports Debian/Ubuntu only.**

## Requirements

None, but note that `hostname -f` should work on every node, and this is especially worth paying some attention if you work with AWS EC2 instances, since in aws the host name is like `ip-xxx-xxx-xxx-xxx.eu-central-1.compute.internal`.

## Must-have Role Variables

Must have variables are listed below:

    rabbitmq_cluster_master: hostname

This parameter is used to determine which node is the master, because some tasks should only be run on master or slave, like slave joins into a cluster. By default it uses first deployed node as a master but you may change it to your own preferences.  

    region

The AWS region where you want your cluster to be deployed

    image_id
    image_name

ID and name of the EC2 instance image (AMI) for RabbitMQ cluster nodes

    key_pair_name

The name of EC2 key pair to apply to newly created nodes in order to being able to connect to them afterwards

    vpc_id

The ID of the VPC where you want your cluster to be deployed

    instance_type

AWS instance type of the cluster nodes. `t3a.small`, `t2.large`, etc.

    private_subnet_id_{01..03}
    public_subnet_id_{01..03}

Here you should set the IDs of the private and public subnets where you want your cluster nodes' network interfaces to be attached. Note that you need exactly 3 public and 3 private subnets (3-node cluster) so if you have for example just 2 of each in your VPC - create another ones before deploying the cluster.

## Optional Role Variables and Defaults

All the other variables are optional and listed below along with default values (see `install_rabbitmq/defaults/main.yml`):

    update_hosts: false

Whether you need to update hosts file or not, default false. This is useful when you are using AWS EC2 instance, whose default hostname is too long and doesn't have a meaning, like "ip-10-101-50-12.eu-central-1.compute.internal", but you want to use something shorter and meaningful as hostname. In this case you need to set this variable to true in order to update the hosts file, and you need to define a variable named "rabbitmq_hosts", with the following format:

    rabbitmq_hosts: |
      node-1-ip node-1-FQDN
      node-2-ip node-2-FQDN

example:

    rabbitmq_hosts: |
      10.0.0.10 eu-central-1-mq-master   (whatever the command `hostname -f` outputs on this host)
      10.0.0.11 eu-central-1-mq-slave-01 (whatever the command `hostname -f` outputs on this host)
.

    rabbitmq_create_cluster: yes

To use cluster or not. Default yes, which means slave will join the master as a cluster.

    rabbitmq_erlang_cookie: WKRBTTEQRYPTQOPUKSVF

RabbitMQ nodes and CLI tools (e.g. rabbitmqctl) use a cookie to determine whether they are allowed to communicate with each other. For two nodes to be able to communicate they must have the same shared secret called the Erlang cookie.

For more info see: <https://www.rabbitmq.com/clustering.html>

    rabbitmq_use_longname: 'false'

When set to true this will cause RabbitMQ to use fully qualified names to identify nodes.  
Note that it is not possible to switch between using short and long names without resetting the node.  
For AWS EC2 it is recommended to set this setting to `true`.  

For more info see: <https://www.rabbitmq.com/configure.html#define-environment-variables>

    rabbitmq_logrotate_period: weekly
    rabbitmq_logrotate_amount: 20

Log rotation settings.

    rabbitmq_ulimit_open_files: 4096

The main setting that needs adjustment is the max number of open files, also known as ulimit -n. The default value on many operating systems is too low for a messaging broker (eg. 1024 on several Linux distributions). RabbitMQ recommends allowing for at least 65536 file descriptors for user rabbitmq in production environments. 4096 should be sufficient for most development workloads.

More info at: <https://www.rabbitmq.com/install-debian.html>

    rabbitmq_tls_port: 5671
    rabbitmq_amqp_port: 5672
    rabbitmq_epmd_port: 4369
    rabbitmq_node_port: 25672

RabbitMQ default ports.

    rabbitmq_plugins:
      - rabbitmq_management
      - rabbitmq_management_agent
      - rabbitmq_shovel
      - rabbitmq_shovel_management

Plugins installed by default, mainly for HTTP API monitor.

    enable_tls: false

Enable TLS/SSL or not.

    tls_only: false

If true, only TLS port is open; default amqp port 5672 will be disabled.

    tls_verify: "verify_none"
    tls_fail_if_no_peer_cert: false

Settings for TLS. More info at: <https://www.rabbitmq.com/ssl.html>

    cacertfile: ""
    cacertfile_dest: "/etc/rabbitmq/cacert.pem"

    certfile: ""
    certfile_dest: "/etc/rabbitmq/cert.pem"

    keyfile: ""
    keyfile_dest: "/etc/rabbitmq/key.pem"

If TLS is enabled, you need to specify cacert/certfile/keyfile location, and where to put them on the server.

    backup_queues_in_two_nodes: false

By default, queues within a RabbitMQ cluster are located on a single node (the node on which they were first declared). Queues can optionally be made mirrored across all nodes, or exactly N number of nodes. By enabling this variable to true, there will be 1 queue master and 1 queue mirror. If the node running the queue master becomes unavailable, the queue mirror will be automatically promoted to master.

More info see: <https://www.rabbitmq.com/ha.html>

## Load balancing

This playbook contains the `load_balancing` role which can automatically deploy a pair of AWS ELB load balancers based on the public/private IP addresses and instance IDs of the RabbitMQ cluster nodes.  
By default it deploys the ALB for management access (HTTP port, without SSL/TLS) and the dedicated NLB for data access (TCP traffic for MQ protocol). For both cases nodes' public IP addresses are used so you could alter the code if you need to expose just private addresses for your cluster and use it in inter-VPC or inter-region scenarios.

## Monitoring

This role installs the Prometheus exporter on each RabbitMQ node <https://github.com/kbudde/rabbitmq_exporter> - it is necessary if you need per-queue monitoring.  
In the case you do not need per-queue monitoring and standard per-node and per-cluster metrics are enough then you can skip this role and add the `prometheus_plugin` in a list of RabbitMQ plugins in the `install_rabbitmq` role instead.  

## Inventory file

You do not need to specify any hosts in your inventory file except localhost. Instead, a dynamic inventory is used and contains IP addresses and hostnames of the newly created EC2 instances.

## Example

Playbook:

    ---
    - hosts: localhost
        connection: local
        roles:
        - deploy_instances

    - hosts: rabbit
        roles:
        - install_rabbitmq
        - cluster_setup
        - monitoring_setup

    - hosts: localhost
        connection: local
        roles:
        - load_balancing

Inventory file:

Go to the [global.yml](group_vars/all/global.yml) file and set all the variables according to what you like.
Note that this installation require you to have 3 private and 3 public networks in your AWS VPC.  

## License

MIT / BSD
