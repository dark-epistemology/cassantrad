.. include:: glossaire.rst
.. include:: url.rst

===========================
Initialisation d'un cluster
===========================

--------------------------------------------------------------
Initialisation d'un cluster multi-noeuds (|DC| unique)
--------------------------------------------------------------

Scénario de déploiement d'un cluster |C| sur un |DC| unique.

Prérequis
==========

Chaque noeud doit être correctement configuré avant de démarrer 
le cluster. On procédera de la manière suivante:

+ Acquisition d'une connaissance minimale du fonctionnement général de |C|.
  Assurez-vous d'avoir lu au minimum `Compréhension de l'architecture`_,
  `Réplication de données`_ et `|C| et les racks`_.

+ Installation de |C| sur chacun des noeuds.

+ Choix du nom du cluster.

+ Récapitulation des adresses IP de chacun des noeuds.

+ Choix des noeuds |seed|. **Ne faites pas de tous les noeuds des noeuds |seed|**. 
  Lisez `Communication inter-noeuds (|G|)`_.

+ Choix de la `|replication strategy|`_ et du `|snitch|`_. Pour les environnements
  de production, GossipingPropertyFileSnitch_ et NetworkTopologyStrategy_ sont
  recommandés.

.. Le paragraphe qui suit m'inquiète sur la qualité globale de la documentation.
   Certes, il est possible d'avoir plusieurs racks, mais le principe de base de 
   cette section était bien qu'on n'avait qu'un datacenter?

+ Si on a plus d'un |DC|, on choisira une convention de nommage pour les
  |DC| et les racks, par example: DC1,DC2 ou 100,200 et RAC1,RAC2 ou R101,R102.
  Attention, il n'est pas possible de renommer un |DC|

+ Le fichier de configuration `cassandra.yaml`_ et les fichiers de propriétés tels 
  que cassandra-rackdc.properties fournissent d'autres éléments de configuration 
  possibles.

Procédure
----------

Dans cette section, on décrit l'installation d'un cluster de 6 noeuds sur 2 racks
dans un |DC| unique. Chaque noeud est configuré avec le GossipingPropertyFileSnitch_
et 256 noeuds virtuels (vnodes).

Pour |C|, un |DC| dénote un groupe de noeuds. Un |DC| est un ménechme
d'un groupe de réplication, c'est à dire un ensemble de noeuds configurés aux mêmes
fins de réplication.

1. Supposons qu'on installe |C| sur les noeuds suivants::

   node0 110.82.155.0 (seed1)
   node1 110.82.155.1
   node2 110.82.155.2
   node3 110.82.156.3 (seed2)
   node4 110.82.156.4
   node5 110.82.156.5

   **Note:** La bonne pratique consiste à désigner plus d'un |seed| par |DC|.

2. Si votre |DC| comporte un pare-feu, il sera nécessaire d'ouvrir certains 
   ports de communication entre les noeuds. Voir `Configuration des ports dans le pare_feu`_.

3. Si |C| est en cours d'exécution, il faudra l'arrêter et supprimer les données existantes.
   Cela permet de supprimer le nom de cluster par défaut (Test Cluster) dans les
   tables système. Tous les noeuds doivent utiliser le même nom de cluster.

   Installation via packages:
   

      a. Arrêt de |C|::

         $ sudo service cassandra stop

      b. Suppression des données::

         $ sudo rm -rf /var/lib/cassandra/data/system/*

   Installation via un tarball:

      a. Arrêt de |C|::

         $ ps auxw | grep cassandra
         $ sudo kill <pid>

      b. Suppression des données::

         $ sudo rm -rf /var/lib/cassandra/data/system/*

4. Modification du fichier de configuration cassandra.yaml sur chaque noeud

   **Note:** Après tout changement du fichier cassandra.yaml, il est nécessaire
   de redémarrer le noeuds pour que la modification soit prise en compte.

   Propriétés à modifier:

   + num_tokens:   *valeur recommandée: 256*

   + seeds:   *adresse IP de chacun des noeuds |seed|*
     Les noeuds |seed| ne sont pas soumis au |bootstrap|,
     le processus par lequel un noeud rejoint un cluster
     existant. Pour les nouveaux clusters, le |bootstrap|
     n'est pas effectué sur les noeuds |seed|.

   + listen_address:
     Lorsque ce paramètre n'est pas positionné, |C| récupère 
     auprès du système l'adresse associée au nom d'hôte. Il 
     arrive que l'adresse ainsi produite ne soit pas la bonne
     et qu'il faille renseigner listen_address [#]_.

.. [#] D'après ce que j'ai vu, |C| choisit par défaut 127.0.0.1, 
   ce qui est systématiquement le mauvais choix pour un cluster multi-noeuds.

   + endpoint_snitch:   *nom du |snitch|*

   + auto_bootstrap:   false (N'ajoutez ce paramètre **que**
     lorsque vous initialisez un cluster sans données).

**Note:** Si vos noeuds sont tous semblables en terme de configuration
de stockage, librairies partagées, etc, vous pouvez utiliser le 
même fichier cassandra.yaml sur tous les noeuds.

   Exemple::

      cluster_name: 'MyCassandraCluster'
      num_tokens: 256
      seed_provider:
         - class_name: org.apache.cassandra.locator.SimpleSeedProvider
           parameters:
            - seeds: "110.82.155.0,110.82.155.3"
      listen_address:
      rpc_address: 0.0.0.0
      endpoint_snitch: GossipingPropertyFileSnitch

5. Dans le fichier ``cassandra-rackdc.properties``, spécifiez les noms de |DC| et 
   de rack que vous avez déterminés dans les Prérequis. Par exemple::

      # Indique le rack et le datacenter de ce noeud
      dc=DC1
      rack=RAC1

6. Après avoir installé et configuré |C| sur tous les noeuds, démarrez les noeuds
   |seed| un par un, puis tous les autres noeuds.

**Note:** Si un noeud redémarre du fait d'un reboot automatique, il sera nécessaire
d'arrêter |C| et de nettoyer les répertoires de données comme indiqué ci-dessus.

   Installation via packages::

      $ sudo service cassandra start

   Installation tarball::

      $ cd repertoire_d_install
      $ bin/cassandra

7. Pour vérifier que le |ring| est bien démarré, lancez:

   Installation via packages::

      $ nodetool status

   Installation tarball::

      $ cd repertoire_d_install
      $ bin/nodetool status
   
   Tous les noeuds doivent apparaître en statut UN (Up/Normal):

.. image:: img/nodetool.png

La localisation du fichier de configuration cassandra.yaml 
dépend du type d'installation:

    +---------------------------+--------------------------------------------------------------+
    | Installation via packages | /etc/cassandra/cassandra.yaml                                |
    +---------------------------+--------------------------------------------------------------+
    | Installation tarball      | repertoire_d_install/resources/cassandra/conf/cassandra.yaml |
    +---------------------------+--------------------------------------------------------------+
    

-----------------------------------------------------------------
Initialisation d'un cluster multi-noeuds (|DC| multiples)
-----------------------------------------------------------------
Scénario de déploiement d'un cluster |C| sur plusieurs |DC|. 

Cette section est très exactement la même que la précédente, à la section 5 de la procédure près.

5. Dans le fichier ``cassandra-rackdc.properties``, spécifiez les noms de |DC| et 
   de rack que vous avez déterminés dans les Prérequis. Par exemple:

   **Noeuds 0 à 2**

::

   # Indique le rack et le datacenter de ce noeud
   dc=DC1
   rack=RAC1

   **Noeuds 3 à 5** 

::

   # Indique le rack et le datacenter de ce noeud
   dc=DC2
   rack=RAC1

