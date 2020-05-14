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
  bbb-1.example.org
  bbb-2.example.org

  [scalelite]
  bbb-scalelite.example.org

  [greenlight]
  bbb-greenlight.example.org

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

The *certificates* role (inside the *roles/* directory of this repository) is in charge to
deploy certificates on remote hosts and it also expose the paths where the certificate's files
have been deployed as variables. These variables are used by other roles inside this
repository to know paths to certificates files when configuring applications.

The role use a YAML file per certificate, containing parts of a x509 certificate:
the private key (``privkey``), the certificate (``cert``) and the certificate chain for signed
certificate (``chain``). As the private key is a sensible information (even more for
wildcards!), this file need to be encrypted. A benefit of encrypting the file is that it
can be put inside a version control system - even I would not recommend a public
repository! - and put alongside the Ansible inventory.

For telling the role where to find these YAML files and which certificate to deploy to
each host, these variables must be set:

* ``certificates_dir``: where to find YAML files (for example:
  *inventories/<ENV>/group_vars/certs*)
* ``certificates``: which certificate(s) to deploy (as list)

These variables can be set either per host, in the case you have one certificate per host,
or for all hosts (in *group_vars/all.yml*) in the case you have a wildcard.

`More details (in French). <https://github.com/unistra/bigbluebutton/tree/master/roles/certificates>`_

Create the certificate YAML file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For generating this file, you must have access to the files of a valid x509 certificate.


.. code::

  $ ansible-vault create inventories/<ENV>/group_vars/certs/<CERT_NAME>.yml --vault-id bbb@vault
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

From the name of the file (*<CERT_NAME>*) is deduced the value to pass to ``certificates``
variable and the name of the dynamically generated variables containing paths of certificates
files on remote hosts. The ``set_fact`` module is used to generate dynamic variables so the
file name must not contains some character (like dots and dashes), except for the *.yml*
extension. A safe way is to replace special characters by underscores.

Exemple with self-signed certificate
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You will have warnings about the certificate in your browser as the certificate is not
signed by a CA and for the exemple, a wildcard is generated.

.. code::

  # Generate self-signed certificate
  $ mkdir certs/
  $ openssl req -newkey rsa:2048 -nodes -keyout certs/example.org.key -x509 -out certs/example.org.crt -days 365 -subj "/CN=*.example.org"

  # Generate file used by Ansible role
  $ cat certs/bbb.example.org.key
  -----BEGIN PRIVATE KEY-----
  MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC9EKFF6FMPY2FG
  WH9bRs3Ui4Mb2XcpJtV5PYo13He+KQcpJcw6k9kde8EFeHRo33NUbAUGj0sZOC1e
  ...

  $ cat certs/bbb.example.org.crt
  -----BEGIN CERTIFICATE REQUEST-----
  MIICXzCCAUcCAQAwGjEYMBYGA1UEAwwPYmJiLmV4YW1wbGUub3JnMIIBIjANBgkq
  hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvRChRehTD2NhRlh/W0bN1IuDG9l3KSbV
  ...

  $ ansible-vault create inventories/<ENV>/group_vars/certs/wildcard_example_org.yml --vault-id bbb@vault
  privkey: |
    -----BEGIN PRIVATE KEY-----
    MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQC9EKFF6FMPY2FG
    WH9bRs3Ui4Mb2XcpJtV5PYo13He+KQcpJcw6k9kde8EFeHRo33NUbAUGj0sZOC1e
    ...

  cert: |
    -----BEGIN CERTIFICATE REQUEST-----
    MIICXzCCAUcCAQAwGjEYMBYGA1UEAwwPYmJiLmV4YW1wbGUub3JnMIIBIjANBgkq
    hkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvRChRehTD2NhRlh/W0bN1IuDG9l3KSbV
    ...

  # Tell Ansible where to find certificates and which one to use
  $ vim inventories/<ENV>/group_vars/all.yml
  certificates_dir: "{{ inventory_dir }}/group_vars/certs/
  certificates: [wildcard_example_org]

  # We don't need the certificate's files anymore
  $ rm -rf certs/

Exemple with a valid wildcard
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code::

  # Generate YAML file containing parts of the certificates
  $ ansible-vault create inventories/<ENV>/group_vars/certs/wildcard.yml --vault-id bbb@vault
  ...

  # Tell Ansible where to find certificates and which one to use
  $ vim inventories/<ENV>/host_vars/all.yml
  certificates_dir: "{{ inventory_dir }}/group_vars/certs/
  certificates: [wildcard]

Exemple with one certificate per host
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When there is one certificate per host, an encrypted YAML file need to be created for each
host. We also need to tell Ansible which certificate to take for each host.

.. code::

  $ ansible-vault create inventories/<ENV>/group_vars/certs/bbb_1.yml --vault-id bbb@vault
  ...
  $ ansible-vault create inventories/<ENV>/group_vars/certs/bbb_2.yml --vault-id bbb@vault
  ...
  # Do the same for bbb-greenlight and bbb-scalelite hosts

  $ vim inventories/<ENV>/host_vars/all.yml
  certificates_dir: "{{ inventory_dir }}/group_vars/certs

  $ vim inventories/<ENV>/host_vars/bbb-1.example.org.yml
  # reference the bbb_1.yml file in the directory defined by certificates_dir variable
  certificates: [bbb_1]

  $ vim inventories/<ENV>/host_vars/bbb-2.example.org.yml
  # reference the bbb_2.yml file in the directory defined by certificates_dir variable
  certificates: [bbb_2]

  # Do the same for bbb-greenlight and bbb-scalelite hosts

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
