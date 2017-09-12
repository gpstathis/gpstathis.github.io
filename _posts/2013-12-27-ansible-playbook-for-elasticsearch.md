---
layout: post
comments: true
title: Ansible Playbook for Elasticsearch
categories:
- elasticsearch
- ansible
- devops
type: post
disqus: true
crosspost_to_medium: true
---

Over the fall, we tried out [Ansible](http://www.ansibleworks.com/) for some of our latest machine provisioning needs and we liked our experience with it. We had been using [Opscode Chef](https://wiki.opscode.com/) up until recently but a lot of the team members were getting stuck climbing its steep learning curve. We had also given [Juju](https://juju.ubuntu.com/) a spin but the cycle of making changes and testing them was simply too slow and painful to continue using it.
<!--more-->

We are getting more experienced with Ansible as time passes and we can finally start sharing some of our playbooks. First on the list is our [Elasticsearch](http://www.elasticsearch.org/) playbook which has been used to configure our production environment for over a month now. You can grab it here: [https://github.com/Traackr/ansible-elasticsearch](https://github.com/Traackr/ansible-elasticsearch).

## Features
- Support for installing plugins
- Support for installing and configuring EC2 plugin
- Support for installing custom JARs in the Elasticsearch classpath (e.g. custom Lucene Similarity JAR)
- Support for installing the [Sematext SPM](http://www.sematext.com/spm/) monitor

## Testing locally with Vagrant
A sample [Vagrant](http://www.vagrantup.com/) configuration is provided to help with local testing. After installing Vagrant, run `vagrant up` at the root of the project to get an VM instance bootstrapped and configured with a running instance of Elasticsearch. Look at `vars/vagrant.yml` and `defaults/main.yml` for the variables that will be substituted in `templates/elasticsearch.yml.j2`.

## Running Standalone Playbook
### Copy Example Files
Make copies of the following files and rename them to suit your environment. E.g.:

- vagrant-main.yml => my-playbook-main.yml
- vagrant-inventory.ini => my-inventory.ini
- vars/vagrant.yml => vars/my-vars.yml

Edit the copied files to suit your environment and needs. See examples below.

### Edit your my-inventory.ini
Edit your my-inventory.ini and customize your cluster and node names:

```cfg
#########################
# Elasticsearch Cluster #
#########################
[es_node_1]
1.2.3.4.compute-1.amazonaws.com
[es_node_1:vars]
elasticsearch_node_name=elasticsearch-1

[es_node_2]
5.6.7.8.compute-1.amazonaws.com
[es_node_2:vars]
elasticsearch_node_name=elasticsearch-2

[es_node_3]
9.10.11.12.compute-1.amazonaws.com
[es_node_3:vars]
elasticsearch_node_name=elasticsearch-3

[all_nodes:children]
es_node_1
es_node_2
es_node_3

[all_nodes:vars]
elasticsearch_cluster_name=my.elasticsearch.cluster
elasticsearch_plugin_aws_ec2_groups=MyElasticSearchGroup
spm_client_token=<your SPM token here>
```

### Edit your vars/my-vars.yml
See `vars/sample.yml` and `vars/vagrant.yml` for exmaple variable files. These are the files where you specify Elasticsearch settings and apply certain features such as plugins, custom JARs or monitoring. The best way to enable configurations is to look at `templates/elasticsearch.yml.j2` and see which variables you want to defile in your `vars/my-vars.yml`. See below for configurations regarding EC2, plugins and custom JARs.

### Edit your my-playbook-main.yml
Example `my-playbook-main.yml`:

```yaml
---

#########################
# Elasticsearch install #
#########################

- hosts: all_nodes
  user: $user
  sudo: yes

  vars_files:
    - defaults/main.yml
    - vars/my-vars.yml

  tasks:
    - include: tasks/main.yml
```

### Launch
```bash
$  ansible my-playbook-main.yml -i my-inventory.ini \
    -e user=<your sudo user for the elasticsearch installation>
```

## Enabling Added Features
### Configuring EC2
The following variables need to be defined in your playbook or inventory:

- elasticsearch_plugin_aws_version

See [https://github.com/elasticsearch/elasticsearch-cloud-aws](https://github.com/elasticsearch/elasticsearch-cloud-aws) for the version that most accurately matches your installation.

The following variables provide a for now limited configuration for the plugin. More options may be available in the future (see http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-discovery-ec2.html):

- elasticsearch_plugin_aws_ec2_groups
- elasticsearch_plugin_aws_ec2_ping_timeout
- elasticsearch_plugin_aws_access_key
- elasticsearch_plugin_aws_secret_key

### Installing plugins
You will need to define an array called `elasticsearch_plugins` in your playbook or inventory, such that:

```yaml
elasticsearch_plugins:
 - { name: '<plugin name>', url: '<[optional] plugin url>' }
 - ...
```

where if you were to install the plugin via bin/plugin, you would type:
`bin/plugin -install <plugin name>` or `bin/plugin -install <plugin name> -url <plugin url>`

Example for [https://github.com/elasticsearch/elasticsearch-mapper-attachments](https://github.com/elasticsearch/elasticsearch-mapper-attachments) (`bin/plugin -install elasticsearch/elasticsearch-mapper-attachments/1.9.0`):

```yaml
elasticsearch_plugins:
 - { name: 'elasticsearch/elasticsearch-mapper-attachments/1.9.0' }
```

Example for [https://github.com/richardwilly98/elasticsearch-river-mongodb](https://github.com/richardwilly98/elasticsearch-river-mongodb) (`bin/plugin -i com.github.richardwilly98.elasticsearch/elasticsearch-river-mongodb/1.7.1`):

```yaml
elasticsearch_plugins:
 - { name: 'com.github.richardwilly98.elasticsearch/elasticsearch-river-mongodb/1.7.1' }
```

Example for [https://github.com/imotov/elasticsearch-facet-script](https://github.com/imotov/elasticsearch-facet-script) (`bin/plugin -install facet-script -url http://dl.bintray.com/content/imotov/elasticsearch-plugins/elasticsearch-facet-script-1.1.2.zip`):

```yaml
elasticsearch_plugins:
 - { name: 'facet-script', url: 'http://dl.bintray.com/content/imotov/elasticsearch-plugins/elasticsearch-facet-script-1.1.2.zip' }
```

### Installing Custom JARs
Custom jars are made available to the Elasticsearch classpath by being downloaded into the elasticsearch_home_dir/lib folder. An example of a custom jar can include a custom Lucene Similarity Provider. You will need to define an array called `elasticsearch_custom_jars` in your playbook or inventory, such that:

```yaml
elasticsearch_custom_jars:
 - { uri: '<URL where JAR can be downloaded from: required>', filename: '<alternative name for final JAR if different from file downladed: leave blank to use same filename>', user: '<BASIC auth username: leave blank of not needed>', passwd: '<BASIC auth password: leave blank of not needed>' }
 - ...
```

### Enabling Sematext SPM
Enable the [Sematext SPM](http://www.sematext.com/spm/) task in your playbook:

```yaml
tasks:
    - include: tasks/spm.yml
    - ...
```

Set the spm_client_token variable in your inventory.ini to your SPM key.

## Include role in a larger playbook
### Add this role as a git submodule
Assuming your playbook structure is such as:

```txt
- my-master-playbook
  |- vars
  |- roles
  |- my-master-playbook-main.yml
  \- my-master-inventory.ini
```

Checkout this project as a submodule under roles:

```bash
$  cd roles
$  git submodule add git://github.com/traackr/ansible-elasticsearch.git ./ansible-elasticsearch
$  git submodule update --init
$  git commit ./submodule -m "Added submodule as ./subm"
```

### Include this playbook as a role in your master playbook
Example `my-master-playbook-main.yml`:

```yaml
---

#########################
# Elasticsearch install #
#########################

- hosts: all_nodes
  user: ubuntu
  sudo: yes

  roles:
    - ansible-elasticsearch

  vars_files:
    - vars/my-vars.yml
```

## Conclusion
If you prefer using Chef, there is of course the excellent and more feature complete Elasticsearch Chef recipie that you can grab here: [https://github.com/elasticsearch/cookbook-elasticsearch](https://github.com/elasticsearch/cookbook-elasticsearch). But if you are looking for an easier start into automatically provisioning your Elasticsearch cluster, take Ansible and this playbook for a spin and let us know what you think. If you see an issue, feel free to log a ticket in [GitHub](https://github.com/Traackr/ansible-elasticsearch). We'll do our best to address it. Same if you want to see a certain feature supported in the fututre. If you'd like to contribute, feel free to clone and submit a pull request.
