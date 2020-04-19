ansible-role-certificates
=========================

Ce rôle permet de déployer les certificats SSL sur des systèmes de type *Debian* et
*RedHat*. Pour chaque certificat, la clé privée, la clé publique, la chaine de
certification et la concaténation clé publique/chaine de certification sont déployés.

L'extension des fichiers varie en fonction du type:

    * *.key*: clé privée
    * *.crt*: clé publique
    * *.chcrt*: chaine de certification
    * *.pem*: concaténation clé publique et chaine


Variables
---------

Où sont déployés les certificats
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Les variables ``ssl_private_path`` et ``ssl_certs_path`` définissent les répertoires où
sont déployés les certificats. La clé publique, la chaine et la concaténation des deux
sont déployés dans ``ssl_certs_path`` et la clé privée est déployée dans
``ssl_private_path``.

Par défaut (voir *defaults/{Debian,RedHat}.yml*), les valeurs de ces variables sont:

    * ``ssl_certs_path``:

      * Debian: */etc/ssl/certs*
      * RedHat: */etc/pki/tls/certs*
    * ``ssl_private_path``:

      * Debian: */etc/ssl/private*
      * RedHat: */etc/pki/tls/private*


Les fichiers suivants sont donc déployés par le role:

    * */etc/ssl/private/$NAME.key*: clé privée
    * */etc/ssl/certs/$NAME.crt*: clé publique
    * */etc/ssl/certs/$NAME.chcrt*: chaine de certification
    * */etc/ssl/certs/$NAME.pem*: concaténation clé publique et chaine

Où **$NAME** est le nom du certificat à déployer.

**Note:** Ces variables peuvent être surchargées ailleurs mais dans ce cas là, les deux
variables doivent être surchargées.

Quels certificats déployer
~~~~~~~~~~~~~~~~~~~~~~~~~~

Le role se base sur la variable ``certificates`` qui doit contenir la liste des
certificats à déployer.

Les certificats déployables sont stockés dans le répertoire *vars/* de ce role.
Il y a un fichier par certificat et le nom du fichier est le nom du certificat. Comme la
clé privée est une information sensible, ces fichiers sont cryptés avec
`ansible-vault <https://docs.ansible.com/ansible/2.4/vault.html>`_
(le mot de passe du vault est le mot de passe infra avec *vault*).

Le format de ces fichiers est le suivant:

.. code-block:: yaml

    ---
    privkey: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      ...

    chain: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      ...

Variables générées
~~~~~~~~~~~~~~~~~~

Lorsque le rôle est exécuté, pour chaque certificats déployés quatre variables (facts)
sont générées contenant le chemin des fichiers:

    * ``$NAME_key_path``
    * ``$NAME_crt_path``
    * ``$NAME_chcrt_path``
    * ``$NAME_pem_path``

A noter qu'une variable Ansible ne peut contenir que lettres, nombres et underscore donc
ces caractères sont remplacés dans les variables générées.

Exemple
-------

Gestion des certificats
~~~~~~~~~~~~~~~~~~~~~~~

Comme indiqué précédemment, les certificats sont stockés et encryptés dans le répertoire
*vars/*. Il suffit donc d'utiliser ansible-vault pour gérer ces fichiers.

Par exemple, pour créer le triple wildcards (*\*.u-strasbg.fr*, *\*.unistra.fr*,
*\*.di.unistra.fr*):

.. code-block:: yaml

    francois@zen:ansible-role-certificates$ ansible-vault create vars/triple_wildcard.yml
    privkey: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      ...

    chain: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
      ...

Usage
~~~~~

Déploiement du triple wildcard sur la machine glpi-test.di.unistra.fr et configuration de
Nginx pour utiliser ce certificat:

.. code-block:: yaml

    ---
    - hosts: glpi-test.di.unistra.fr
      remote_user: root
      gather_facts: yes

      vars:
        # Certificats à déployer.
        certificates:
          - triple_wildcard

        nginx_vhosts:
          - listen: "443 ssl"
            server_name: "{{ server_name | default(inventory_hostname) }}"
            root: /var/www/glpi
            index: index.php
            access_log: /var/log/nginx/{{ server_name | default(inventory_hostname) }}_access.log
            error_log: /var/log/nginx/{{ server_name | default(inventory_hostname) }}_error.log
            extra_parameters: |
              # on utilise les variables définies par le role 'certificates'
              ssl_certificate     {{ triple_wildcard_pem_path }};
              ssl_certificate_key {{ triple_wildcard_key_path }};

              location / {try_files $uri $uri/ index.php;}
              ...

      roles:
        - { role: certificates }
        - { role: geerlingguy.nginx }
