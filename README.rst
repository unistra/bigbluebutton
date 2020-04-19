***********************************************************
Ansible playbooks/roles to deploy Big Blue Button instances
***********************************************************

This playbook installs a fully operationnal instance of `Big Blue Button <https://docs.bigbluebutton.org/>`__ with `Scalelite <https://github.com/blindsidenetworks/scalelite>`__ load-balancer and `Greenlight  <https://docs.bigbluebutton.org/greenlight/gl-overview.html>`__ interface for managing rooms.

Disclaimer
==========

The main playbook *playbook.yml* will not work out of the box as we use some internal roles based on our environment. The main playbook just import other playbooks:

* *provisioning.yml*: provision virtual machines on our OpenNebula platform, install some base tools for our environment (monitoring, ntp, ...), configure storage network interface and NFS mounts, ...
* *bigbluebutton.yml*: deploy Big Blue Button instances
* *scalelite.yml*: deploy Scalelite load-balancer
* *greenlight.yml*: deploy Greenligth interface
* *monitoring.yml*: deploy monitoring for Big Blue Button (`this Prometheus exporter <https://github.com/greenstatic/bigbluebutton-exporter>`__; note that will install Docker on the backends) and Scalelite (an NRPE check)

If you want to use the main playbook, you need to comment the import of *provisioning.yml* playbook and, if you don't want this monitoring stack, *monitoring.yml* . Otherwise you can always use playbooks individually.

For any question, you can contact us at:  dnum-bbb at unistra dot fr

Requirements
============

* Updated systems accessible via SSH
* A NFS volume, correclty configured and mounted on bigbluebutton and scalelite hosts (by default */mnt/scalelite-recordings* is used)

Deployment
==========

Retrieve playbook
-----------------

.. code::

  $ git clone https://github.com/unistra/bigbluebutton

Install dependencies
--------------------

Scalelite and Greenlight use Docker images. The local ``docker`` role import `geerlingguy.docker <https://github.com/geerlingguy/ansible-role-docker/releases>`__ role before installing dependencies to use Ansible ``docker_compose`` module.

To install ``geerlingguy.docker`` role:

.. code::

  $ ansible-galaxy install --roles-path roles geerlinguy.docker


Create a vault
--------------

There are many sensitive variables (applications tokens, password of LDAP administrative account, ...), so it is highly recommended to encrypt these variables to prevent pushing them by inadvertance.

There are many ways to manage vaults with Ansible. Our choice was to put a random key in a file. This file can be shared by anyone working on the project and is passed to ``ansible-`` commands using ``--vault-id`` option. Note that *.gitignore* file is configured to ignore any file starting by *vault* to prevent pushing this key.

.. code::

  $ openssl rand -hex 32 > vault

To decrypt variables in an inventory, the simple ``ansible`` command and *debug* module can be used:

.. code::

  $ ansible -i inventories/<ENV> <INVENTORY_GROUP> -m debug -a "var=<VAR>" --vault-id bbb@vault

Create an inventory
-------------------

Initialize an inventory for your environment from the sample inventory:

.. code::

  $ cp -r inventories/example inventories/

Then edit the *hosts* file to put your hosts. There is one group per component:

.. code::

  [bigbluebutton]
  bbb-backend-1.example.com
  bbb-backend-2.example.com

  [scalelite]
  bbb-scalelite.example.com

  [greenlight]
  bbb-greenlight.example.com

Set the required variables in *inventories/<ENV>/group_vars/all.yml*:

* ``scalelite_secret_key``: Rails secret key

.. code::

  ansible-vault encrypt_string --vault-id bbb@vault --name scalelite_secret_key $(openssl rand -hex 64)

* ``scalelite_loadbalancer_key``: Load balancer key (ie: the one to configure client-side)

.. code::

  ansible-vault encrypt_string --vault-id bbb@vault --name scalelite_loadbalancer_key $(openssl rand -hex 32)

* ``greenlight_secret_key``:

.. code::

  ansible-vault encrypt_string --vault-id bbb@vault --name greenlight_secret_key $(openssl rand -hex 32)

Manage SSL certificates
-----------------------

To deploy certificates, local role ``certificates`` is used. This role generate these files:

  * */etc/ssl/private/<CERT_NAME>.key*: the private key of the certificate
  * */etc/ssl/certs/<CERT_NAME>.crt*: the public key of the certificate
  * */etc/ssl/certs/<CERT_NAME>.chain*: the chain of the certificate
  * */etc/ssl/certs/<CERT_NAME>.pem*: the concatenation of the public key and and the chain, required by NGinx

The role exposes these paths as variables which are used by ``bigbluebutton``, ``greenlight`` and ``scalelite`` roles.

To create the certificate file:

.. code::

  $ ansible-vault create inventories/<ENV>/certs/<CERT_NAME>.yml --vault-id bbb@vault
  privkey: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----

  cert: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

  chain: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

Inventory also need to be updated:

.. code::

  $ vim inventories/<ENV>/group_vars/all.yml
  ...
  certificates_dir: "{{ inventory_dir }}/group_vars/certs
  certificates: [<CERT_NAME>]

Usage
=====

To execute the playbook:

.. code::

  $ ./playbook.yml -i inventories/<ENV> --vault-id bbb@vault -e scalelite_db_init=yes

**Note**: The option ``-e scalelite_db_init=yes`` need to be executed only once to initialize the Scalelite database which is required by Greenlight!

We also support these tags:

* *bigbluebutton*: deploy the bigbluebutton backends
* *scalelite*: deploy the Scalelite load balancer
* *greenlight*: deploy the greenlight app
