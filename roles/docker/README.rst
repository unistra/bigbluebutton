***********
Docker role
***********

Rôle pour installer ``Docker``, ``docker-compose`` et les dependences Python nécessaires
au fonctionnement des modules Ansible.

Systèmes testés:

* *CentOS 7* (Python 2.7)
* *Ubuntu 18.04* (Python 3)

Dependences
===========

Ce rôle est dépendant du rôle Galaxy ``geerlingguy.docker``.

Un fichier *requirements.yml* est fourni, donc une fois ce rôle déployé:

.. code::

    ansible-galaxy install -r roles/docker/requirements.yml --roles-path roles/

Variables
=========

Les variables sont celles du rôle `geerlingguy.docker
<https://github.com/geerlingguy/ansible-role-docker#role-variables>`__.

Playbook
========

.. code::

    - hosts: all
      remote_user: root
      gather_facts: yes
      tasks:
      - import_role: { name: docker }
