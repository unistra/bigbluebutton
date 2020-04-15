**************************************************************
Ansible playbooks/roles to deploy HA Big Blue Button instances
**************************************************************

This playbook setups a fully operationnal instance of BigBlueButton with a Scalelite load-balancer and Greenlight interface.

Disclaimer
==========

This playbook will not work out of the box as we use some internal roles.

Our specific roles provide:

  * the provisionning virtual machines on OpenNebula
  * the declaration of DNS names
  * the configuration of NFS client on every server.
  * deployment of our certificates

We'll work on it to make it more generic.

For any question, you can contact us at:  dnum-bbb at unistra dot fr

Requirements
============

* An updated system accessible via SSH
* A NFS volume, correclty configured and mounted on each host
* A valid certificate on each server

  * Our links are hard-coded so you need to deploy certificates as:

    * */etc/ssl/certs/mega_wildcard.pem* for the public key (which need to be concatenated with the chain if required)
    * */etc/ssl/private/mega_wildcard.key* for the private key

Deployment
==========

**Note**: We recommend to encrypt your variables. In our case, we generated a key of 32 characters that we put on a file which is passed to `--vault-id` option.

At first, you need to setup a new inventory from the sample inventory:

.. code::

  $ cp inventories/example inventories/<ENV>

Each component - Big Blue Button, Scalelite and Greenlight - has his own group in the *inventories/<ENV>/hosts* file:

.. code::

  [bigbluebutton]
  bbb-backend-1.mydomain.fr
  bbb-backend-2.mydomain.fr
  bbb-backend-3.mydomain.fr
  bbb-backend-4.mydomain.fr

  [scalelite]
  bbb-scalelite.mydomain.fr

  [greenlight]
  bbb-greenlight.mydomain.fr

There is one role per component which use variables. Some variables are used between roles (like using the content of *bigbluebutton* group for generating the scalelite pool) so all variables are set in the *inventories/<ENV>/group_vars/all.yml*. For instance:

.. code::

  ---
  bigbluebutton_configure_firewall: no
  bigbluebutton_config:
    webcamsOnlyForModerator: 'true'
    muteOnStart: 'true'
    allowModsToUnmuteUsers: 'true'

  bigbluebutton_camera_profiles:
  - id: low
    name: Low quality
    default: false
    bitrate: 50
  - id: medium
    name: Medium quality
    default: true
    bitrate: 100
  - id: high
    name: High quality
    default: false
    bitrate: 200
  - id: hd
    name: High definition
    default: false
    bitrate: 500

  # openssl rand -hex 64
  scalelite_secret_key: FIXIT
  # openssl rand -hex 32
  scalelite_loadbalancer_key: FIXIT

  # You can use docker container for the scalelite database, if needed.
  scalelite_db_docker: no
  scalelite_db_host: bdd-scalelite.mydomain.fr
  scalelite_db_user: scalelite
  scalelite_db_pass: FIXIT
  scalelite_db_name: scalelite

  # openssl rand -hex 64
  greenlight_secret_key: FIXIT

  # You can use docker container for the greenlight database, if needed.
  greenlight_db_docker: no
  greenlight_db_host: bdd-greenlight.mydomain.fr
  greenlight_db_user: greenlight
  greenlight_db_pass: FIXIT
  greenlight_db_name: greenlight

  # We use an ldap authentification, in our case.
  greenlight_ldap_server: ldap.mydomain.fr
  greenlight_ldap_port: 389
  greenlight_ldap_method: plain
  greenlight_ldap_uid: uid
  greenlight_ldap_base: FIXIT
  greenlight_ldap_bind_dn: FIXIT
  greenlight_ldap_bind_pwd: FIXIT

  scalelite_datadir: "/var/scalelite"

If you want to start your infrastructure, you need to launch this command.

.. code::

  $ ./playbook.yml -i inventories/<ENV> --vault-id bbb@vault -e scalelite_db_init=yes --skip-tags monitoring

**Note**: The option `-e scalelite_db_init=yes` need to be executed only once to initialize the Scalelite database which is required by Greenlight!

**Note**: We put tasks respectively in `bigbluebutton` and `scalelite` roles for monitoring purposes which are specific to our infrastructure (NRPE and Prometheus). Keep the `skip-tags` option if not required.

We also support these tags:

* *bigbluebutton*: deploy the bigbluebutton backends
* *scalelite*: deploy the Scalelite load balancer
* *greenlight*: deploy the greenlight app

Showing an encrypted variable
=============================

.. code::

  $ ansible -i inventories/<ENV> <INVENTORY_GROUP> -m debug -a "var=<VAR>" --vault-id bbb@vault

For example, if you need to get the scalelite token (`scalelite_loadbalancer_key`), you may execute:

.. code::

  $ ansible -i inventories/example -m debug -a "var=scalelite_loadbalancer_key" --vault-id bbb@vault
  bbb-ansible-scalelite.di.unistra.fr | SUCCESS => {
    "scalelite_loadbalancer_key": "****************************************************************"
  }
