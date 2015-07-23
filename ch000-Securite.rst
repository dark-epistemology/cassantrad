.. include:: glossaire.rst
.. include:: url.rst
===========
Sécurité
===========

.. image:: img/swine.jpg
   :scale: 25 %
   :align: center

.. contents:: Contenu de cette section

--------------------
Sécurisation de |C|
--------------------

|C| dispose des caractéristiques de sécurité suivantes:

- Cryptage des communications entre clients et noeuds base de données
  |C| propose une forme de communication sécurisée optionnelle entre
  les clients et les noeuds base de données. Le cryptage SSL entre 
  clients et serveurs assure la sécurité et l'intégrité des communications.

- Authentification par login/mot de passe
  L'administrateur peut créer des utilisateurs authentifiés via la commande
  CREATE USER. En interne, |C| gère les comptes utilisateurs et les accès
  avec des mots de passe. Les comptes peuvent être modifiés ou supprimés 
  en `CQL`_ (Cassandra Query Language)

- Gestion des permissions sur les objets
  Une fois l'utilisateur authentifié sur le cluster, reste à déterminer
  quelles actions lui sont autorisées; |C| utilise la terminologie familière
  du GRANT/REVOKE pour gérer les permissions.

--------------
Cryptage SSL
--------------

Cryptage des communications entre clients et noeuds |C|
========================================================     

Le cryptage entre les clients et le cluster établit un
canal sécurisé entre le client et son noeud |coordinator|.

Prérequis
----------

Chaque noeud doit disposer des certificats SSL requis. Voir `Préparation des certificats SSL`_.

Pour activer le cryptage SSL client/serveur, il faut positionner le paramètre ``client_encryption_options``
dans le fichier `cassandra.yaml`_.

Procédure
--------------

Sur chaque noeud, dans la section  ``client_encryption_options`` du fichier `cassandra.yaml`_:

- Activez le cryptage
- Spécifiez les chemins de vos fichiers ``.keystore`` et ``.truststore``
- Spécifiez les mots de passe; ils doivent correspondre aux mots de passe
  utilisés lors de la génération des fichiers ``.keystore`` et ``.truststore``.
- Finalement, on active l'authentification cliente par certificat en positionnant
  ``require_client_auth`` à ``true``. (disponible depuis |C| 1.2.3)

Exemple
--------

::

   client_encryption_options:
      enabled: true
      keystore: conf/.keystore ## chemin de votre fichier .keystore 
      keystore_password: <mdp keystore> ## Mot de passe utilisé pour la génération du keystore
      truststore: conf/.truststore
      truststore_password: <mdp truststore>
      require_client_auth: <true ou false>

**Référence**
Le fichier de configuration `cassandra.yaml`_

Cryptage des communications entre les noeuds |C|
==================================================
Le cryptage SSL inter-noeuds protège les transferts de données entre les noeuds d'un cluster,
et notamment les échanges Gossip.

Prérequis
----------

Chaque noeud doit disposer des certificats SSL requis. Voir `Préparation des certificats SSL`_.

Pour activer le cryptage SSL internoeuds, il faut positionner le paramètre ``server_encryption_options``
dans le fichier `cassandra.yaml`_.

Procédure
----------

Sur chaque noeud, dans la section ``server_encryption_options`` du fichier `cassandra.yaml`_:

- Activez le cryptage avec ``internode_encryption``.
  Les options possibles sont:

      * all
      * none
      * dc: |C| crypte les échanges entre |DC|.
      * rack: |C| crypte les échanges entre les racks.

- Spécifiez les chemins de vos fichiers ``.keystore`` et ``.truststore``
- Spécifiez les mots de passe; ils doivent correspondre aux mots de passe
  utilisés lors de la génération des fichiers ``.keystore`` et ``.truststore``.
- Finalement, on active l'authentification cliente par certificat en positionnant
  ``require_client_auth`` à ``true``. (disponible depuis |C| 1.2.3)

Exemple
--------

::

   server_encryption_options:
      internode_encryption: <option choisie>
      keystore: conf/.keystore ## chemin de votre fichier .keystore 
      keystore_password: <mdp keystore> ## Mot de passe utilisé pour la génération du keystore
      truststore: conf/.truststore
      truststore_password: <mdp truststore>
      require_client_auth: <true ou false>

Utilisation du cryptage SSL avec cqlsh
=======================================

L'utilisation d'un fichier ``cqlshrc`` permet de n'avoir pas à positionner la 
variable SSL_CERTFILE à chaque utilisation de ``cqlsh``.

Procédure
----------

1. pour activer le cryptage SSL pour cqlsh, créez un fichier .cassandra/cqlshrc dans votre 
   répertoire HOME ou dans celui du programme client.
   Des fichiers exemples sont diponibles dans les répertoires suivants:

   + Installation par packages:   /etc/cassandra
   + Installation tarball:      chemin_d_install/conf

2. démarrez cqlsh avec l'option --ssl:

::

   $ cqlsh --ssl            ## Installation par packages
   $ chemin_d_install/bin/cqlsh --ssl   ## Installation tarball

Exemple
--------

::

   [authentication]
   username = fred
   password = !!bang!!$

   [connection]
   hostname = 127.0.0.1
   port = 9042

   [ssl]
   certfile = ~/keys/cassandra.cert
   validate = true ## Optionnel, true par défaut
   userkey = ~/key.pem ## nécessaire lorsque require_client_auth=true
   usercert = ~/cert.pem ## nécessaire lorsque require_client_auth=true

   [certfiles]  ## Section facultative, outrepasse les certificats par défaut de la section ssl
   192.168.1.3 = ~/keys/cassandra01.cert
   192.168.1.4 = ~/keys/cassandra02.cert

**Note:** Lorsque la validation est active (validate=true), l'hôte spécifié dans le certificat
est comparé à celui de la machine à laquelle on est connecté. Le certificat SSL doit être fourni
soit dans le fichier de configuration soit via une variable d'environnement. Les variables 
(SSL_CERTFILE et SSL_VALIDATE) outrepassent les options définies dans le fichier de configuration.

**Référence:**
Le fichier de configuration `cassandra.yaml`_

Préparation des certificats SSL
=================================

On indique ici comment générer des certificats SSL pour le cryptage des communications client/serveur
ou serveur/serveur. Les certificats générés peuvent être utilisés pour les deux types de cryptage. Les 
certificats doivent être distribués sur tous les noeuds. Un |keystore| contient des clés privées. Un 
truststore contient les certificats SSL de tous les noeuds et ne nécessite pas d'être signé par une 
autorité publique de certification.

Procédure
----------

1. Générez des clés publiques et privées pour tous les noeuds du cluster:

::

   keytool -genkey -keyalg RSA -alias <nom_du_noeud> -keystore .keystore

2 - Exportez les certificats  sur chacun des noeuds:

::

   keytool -export -alias <nom_du_noeud> -file <nom_du_noeud>.cer -keystore .keystore

3 - Centralisez les certificats <nom_du_noeud>.cer sur un des noeuds.

4 - Importez tous les certificats dans un |truststore|:

::

   keytool -import -v -trustcacerts -alias <nom_du_noeud1> -file <nom_du_noeud1>.cer -keystore .truststore
   keytool -import -v -trustcacerts -alias <nom_du_noeud2> -file <nom_du_noeud2>.cer -keystore .truststore
   ...

5 - Distribuez le fichier ``.truststore`` ainsi constitué sur tous les noeuds du cluster.

6 - Assurez-vous que le fichier ``.keystore`` n'est accessible en lecture qu'au daemon |C|.

    Pour ajouter de nouveaux utilisateurs, il faut:

   - Créer un nouveau certificat comme indiqué ci-dessus,
   - Importer ce nouveau certificat dans les ``truststore`` de chacun des noeuds du cluster:

::

   keytool -import -v -trustcacerts -alias <utilisateur> -file <fichier certificat> -keystore .truststore

-----------------------------
Gestion de l'authentification
-----------------------------

Authentification interne
=========================

Comme la gestion des permissions, l'authentification interne est basée sur un système de login/mot de passe.
L'authentification interne fonctionne pour les clients suivants lorsqu'on les démarre en spécifiant un
login et un mot de passe:

   + Astyanax
   + cassandra-cli
   + cqlsh
   + Les drivers Datastax - produit et certifiés par Datastax pour l'utilisation avec |C|.
   + Hector
   + pycassa

L'authentification interne stocke les logins et mots de passe (hachés avec bcrypt) dans la table
system_auth.credentials.

PasswordAuthenticator est une implémentation de IAuthenticator qui permet d'activer l'authentification
interne sans configuration ou presque.

Configuration de l'authentification interne
============================================

Pour configurer l'authentification interne, il suffit de modifier `cassandra.yaml`_ et d'augmenter
le facteur de réplication du keyspace system_auth, comme indiqué dans ce qui suit. On se connectera
ensuite par cqlsh à |C| avec l'utilisateur par défaut cassandra/cassandra et on utilisera les ordres
CQL suivants pour gérer les accès à la base:

   + ALTER USER
   + CREATE USER
   + DROP USER
   + LIST USERS

**Note:** Pour configurer les permissions, se référer à `Configuration des permissions`_.

Procédure
----------

1. Modifiez l'option ``authenticator`` du fichier `cassandra.yaml`_ comme suit:

::

   authenticator: PasswordAuthenticator

Par défaut, l'option est désactivée (AllowAllAuthenticator)

2. Augmentez le facteur de réplication du keyspace system_auth de façon à ce qu'il coincide 
   avec le nombre de noeuds du cluster.
   Le facteur de réplication par défaut de system_auth étant de 1 - ie, la réplication est
   désactivée -, en cas de défaillance du noeud contenant la |replica| unique la connexion
   au cluster sera impossible.

3. Redémarrez le client |C|:

::

   ./cqlsh -u cassandra -p cassandra

4. Créez un autre administrateur 

5. Logez vous sous ce nouveau compte administrateur

6. Modifiez le mot de passe cassandra

7. Révoquez le statut administrateur de cassandra

8. Utilisez CQL pour créer des utilisateurs et leur donner les accès pertinents aux objets de la base.


Connexion via cqlsh
=====================

Typiquement, après la configuration de l'authentification, on se connecte via ``cqlsh`` en spécifiant un login
et un mot de passe via les options -u et -p. Pour éviter d'avoir à les spécifier chaque fois que vous utilisez
``cqlsh``, il est possible de créer un fichier ``cqlshrc`` dans le répertoire ``.cassandra`` et d'y spécifier
les informations de connexion par défaut.

   **Note:** Des fichiers cqlshrc d'exemple sont disponibles dans les répertoires suivants:

   + Installation via packages:   ``/etc/cassandra`` 
   + Installation tarball:      ``repertoire_d_install/conf``

Procédure
----------

1. Avec un éditeur, créez un fichier spécifiant un login et un mot de passe:

::

   [authentication]
   username = fred
   password = !!bang!!$

2. Sauvez le fichier dans votre répertoire ~/.cassandra sous le nom ``cqlshrc``.

3. Modifiez les permissions du fichier de façon à ce qu'il ne soit lisible que par son propriétaire.

------------------------
Gestion des permissions
------------------------

Permissions sur les objets
============================

|C| fournit le paradigme familier des bases de données relationnelle, les ordres GRANT/REVOKE
permettant d'autoriser ou d'interdire l'accès aux données. Un administrateur donne initialement
les accès et par la suite, un utilisateur peut ou non avoir le droit de transférer ces accès.
La gestion des permissions est fondée sur le mécanisme d'autorisation interne. 

L'accès en lecture aux tables suivantes est implicitement donné à tous les utilisateurs 
authentifiés, parce que la plupart des outils |C| les utilisent:

   + system.schema_keyspace
   + system.schema_columns
   + system.schema_columnfamilies
   + system.local
   + system.peers

Configuration des permissions
===============================

   La classe CassandraAuthorizer est une des nombreuses implémentations possibles de IAuthorizer;
   elle stocke les permissions dans la table system_auth.permissions et supporte tous les ordres
   CQL liés aux permissions. Sa configuration consiste simplement à signaler son utilisation dans
   `cassandra.yaml`_.

   **Note:** Pour configurer l'authentification, se référer à `Gestion de l'authentification`_.

Procédure
----------

1. Dans le fichier `cassandra.yaml`_, modifiez le paramètre ``authorizer`` comme suit:

::

   authorizer: CassandraAuthorizer

Il est possible d'utiliser n'importe quelle classe d'autorisation à l'exception de AllowAll.

2. Configurez le facteur de réplication du keyspace system_auth à une valeur supérieure à 1.

3. Ajustez la période de validité du cache de permissions en positionnant le paramètre 
   permissions_validity_in_ms dans le fichier `cassandra.yaml`_.  Alternativement, vous pouvez 
   désactiver le cache de permissions en valorisant ce paramètre à 0.

Résultat
----------

CQL supporte maintenant les ordres suivants:

   + GRANT
   + REVOKE
   + LIST PERMISSIONS

-------------------------------------------
Configuration des ports d'accès du pare-feu
-------------------------------------------

On décrit dans cette section les ports à ouvrir sur le pare-feu.

Si un pare-feu gère les communications entre les noeuds de votre
cluster |C|, il est nécessaire d'ouvrir les ports suivants pour 
autoriser les échanges entre les noeuds. Si cette ouverture n'est
pas faite, au démarrage les noeuds se considèreront comme isolés
et ne rejoindront pas le cluster.

**Table 2: Ports publics**

+----------+-----------------------------------------------+
| Port     | Description                                   |
+==========+===============================================+
| 22       | Port ssh                                      |
+----------+-----------------------------------------------+
| 8888     | OpsCenter écoute sur ce port les requêtes     |
|          | HTTP en provenance du navigateur.             |
+----------+-----------------------------------------------+

**Table 3: Ports de communication entre noeuds** |C|

+----------+-----------------------------------------------+
| Port     | Description                                   |
+==========+===============================================+
| 7000     | Communication inter-noeuds                    |
+----------+-----------------------------------------------+
| 7001     | Communication inter-noeuds SSL                |
+----------+-----------------------------------------------+
| 7199     | Port d'accès JMX                              |
+----------+-----------------------------------------------+

**Table 4: Ports clients**

+----------+-----------------------------------------------+
| Port     | Description                                   |
+==========+===============================================+
| 9042     | Port client CQL                               |
+----------+-----------------------------------------------+
| 9160     | Port client Thrift                            |
+----------+-----------------------------------------------+

**Table 5: Ports OpsCenter**

+----------+-----------------------------------------------+
| Port     | Description                                   |
+==========+===============================================+
| 61620    | Le daemon OpsCenter écoute sur ce port les    |
|          | communications en provenance des agents       |
+----------+-----------------------------------------------+
| 61621    | Les agents écoutent sur ce port les           |
|          | communications SSL initiées par OpsCenter     |
+----------+-----------------------------------------------+

------------------------------------
Activation de l'authentification JMX
------------------------------------

Par défaut, |C| restreint l'accès à JMX au seul localhost.
Pour autoriser les connexions JMX distantes, il faut modifier
la variable LOCAL_JMX dans le fichier cassandra-env.sh et 
activer l'authentification et/ou l'accès SSL.

Après avoir activé l'authentification, assurez-vous que les
outils qui utilisent JMX, tels que ``nodetool`` et OpsCenter,
sont bien configuré pour l'authentification.

Pour utiliser l'authentification JMX sous OpsCenter, on se
réfèrera à `Modification des connexions cluster de OpsCenter`_.

Procédure
==========

1. Editez le fichier ``cassandra-env.sh`` et modifiez ou ajoutez ces lignes:

::

   JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.authenticate=true"
   JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.password.file=/etc/cassandra/jmxremote.password"
   LOCAL_JMX=no

2. Copiez le fichier ``jmxremote.password.template`` de /jre_install_location/lib/management 
   vers /etc/cassandra, et renommez-le jmxremote.password:

::

   cp /jre_install_dir/lib/management/jmxremote.password.template /etc/cassandra/jmxremote.password

3. Modifiez les permissions du fichier jmxremote.password de façon à ce qu'il 
   appartienne à l'utilisateur cassandra et qu'il soit en lecture seule:

::

   $ chown cassandra:cassandra /etc/cassandra/jmxremote.password
   $ chmod 400 /etc/cassandra/jmxremote.password

4. Editez le fichier jmxremote.password pour y ajouter un utilisateur et 
   un mot de passe à l'usage des outils JMX:

::

   monitorRole QED
   controlRole R&D
   cassandra cassandrapassword

**Note:** Cet utilisateur et son mot de passe ne sont que des exemples.
