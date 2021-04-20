# Kafka Management using Ansible and Ansible roles. Including SASL and ACL's.

The motivation behind this playbook is to start building a solution that allows management of Kafka and Zookeeper in Ansible including setting up basic authentication and security on Kafka. 

For a start the solution will only be able to cover either PLAINTEXT or SASL_PLAIN authentication.
As the project grows and expands I will start building in multi-listener and multi-authentication modes.

## Installing Kafka and Zookeeper

### Setup the inventories/hosts.yml to match your specific inventory

Zookeeper nodes should fall under *zookeeper_nodes* section and should be eiter a hostname or ip address.

Kafka Broker nodes should fall under the *kafka_brokers* section and should be eiter a hostname or ip address.

### Setup the group_vars

### PLAINTEXT

For PLAINTEXT Authorisation set the following variables in group_vars/all.yml

    kafka_listener_protocol: PLAINTEXT
    kafka_inter_broker_listener_protocol: PLAINTEXT
    kafka_allow_everyone_if_no_acl_found: 'true'    #!IMPORTANT

### SASL

For SASL_PLAINTEXT Authorisation set the following variables in group_vars/all.yml

    configure_sasl: false
    configure_acl: false
    
    kafka_opts:
      -Djava.security.auth.login.config=/opt/kafka/config/jaas.conf
    kafka_listener_protocol: SASL_PLAINTEXT
    kafka_inter_broker_listener_protocol: SASL_PLAINTEXT
    kafka_sasl_mechanism_inter_broker_protocol: PLAIN
    kafka_sasl_enabled_mechanisms: PLAIN
    kafka_super_users: "User:admin"  #SASL Admin User that has access to administor kafka.
    kafka_allow_everyone_if_no_acl_found: 'false' 
    kafka_authorizer_class_name: "kafka.security.authorizer.AclAuthorizer"

Once the above has been set as configuration for Kafka and Zookeeper you will need to setup the topics and SASL users.

For the SASL User list it will need to be set in the group_vars/kafka_brokers.yml
These need to be set on all the brokers and the play will configure the jaas.conf on ever broker in a rolling fashion.

The list is a simple YAML format username and password list.

Please dont remove the admin_user_password this needs to be set so that the brokers can communicate with each other.

The default admin username is **admin**.

## Topics and ACL's

in the group_vars/all.yml there is a list called topics_acl_users.
This is a 2-fold list that manages the topics to be created as well as the ACL's that need to be set per topic.

In a PLAINTEXT configuration it will read the list of topics and create only those topics.

In a SASL_PLAINTEXT with ACL context it will read the list and create topics and set user permissions(ACL's) per topic. 
There are 2 components to a topic and that is has user that can Produce to or Consume from a topic and the list splits that functionality too.

## Installation Steps

Run the playbooks/base.yml file to install SSH Keys and OpenJDK. If applicable to any.

They can individually be toggled on or off with variables in the group_vars/all.yml

    install_ssh_key: true
    install_openjdk: true

    Example play:
    ansible-playbook playbooks/base.yml -i inventories/hosts.yml -u root

Once the above has been setup the environment should be prepped with the basic for the kafka and zookeeper install to connect as root user and install and configure.

They can individually be toggled on or off with variables in the group_vars/all.yml

The variables have been set to use opensource/Apache kafka.
TODO: Add support for confluent kafka

    install_zookeeper_opensource: true
    install_kafka_opensource: true

    ansible-playbook playbooks/install_kafka_zkp.yml -i inventories/hosts.yml -u root   

Once kafka has been installed then the last playbook needs to be run.

Based on either SASL_PLAINTEXT or PLAINTEXT configuration the playbook will

- Configure topics
- Setup ACL's (If SASL_PLAINTEXT)

**Please note that for ACL's to work in Kafka there needs to be an authentiction engine behind it.**

If you want to install kafka to allow any connections and auto create topics please set the following configuration in the group_vars/all.yml

    configure_topics: false
    kafka_auto_create_topics_enable: true

This will disable the topic creation step and allow any topics to be created with the kafka defaults.

Once all the above topic and ACL config has been finalised please run

    ansible-playbook playbooks/configure_kafka.yml -i inventories/hosts.yml -u root

## Testing the plays

You can either run a producer or consumer on the kafka broker you have set or you can use a third party tool to send logs. In this test we have used metricbeat.

Steps

- Start logging tool aka Metricbeat
- Consume messages from topic

Examples

    PLAIN TEXT
    /opt/kafka/bin/kafka-console-consumer.sh  --bootstrap-server $(hostname):9092 --topic metricbeat --group metricebeatCon1

    SASL_PLAINTEXT
    /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server $(hostname):9092 --consumer.config /opt/kafka/config/kafkaclient.jaas.conf --topic metricbeat --group metricebeatCon1

As part of the ACL play it will create a default kafkaclient.jaas.conf file as used in the examples above.
This has the basic setup needed to connect to Kafka from any client using SASL_PLAINTEXT Authentication.

If you feel like contributing please contact myself or any of the team members at https://www.digitlis.io

Thanks.

Brian Stark
