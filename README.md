hydre
====

HyDre - Hydroweb Distribution server - est l'application de distribution des séries temporelles de hauteurs d'eau (lacs et fleuves) du projet Hydroweb.

L'application HyDre est développé par le [cnes] (http://www.cnes.fr). Elle repose sur les applications [RESTo] (http://github.com/jjrom/resto), [mapshup] (http://github.com/jjrom/mapshup) et [iTag] (http://github.com/jjrom/itag)

Voir la [présentation de HyDre sur slideshare](http://fr.slideshare.net/GasperiJerome/20140422-hydre-toulouse)

Installation
============

Pré-requis
----------

* git (ligne de commande)
* Apache (v2.0+) avec le support de mod_rewrite
* PHP (v5.3+) avec les extension curl et XMLWriter
* PostgreSQL (v9.0+) avec l'extension hstore (installée par défaut avec postgres)
* PostGIS (v1.5.1+)

Note: HyDre peut être installé avec des logiciels de versions inférieures à celles spécifiées...mais le fonctionnement nominal ne peut être garanti dans ce cas.


Préparation
-----------
Nous supposons que $HYDRE_HOME est le répertoire dans lequel sont installées les sources de l'application

        export HYDRE_HOME=/repertoire/sources/hydre

Par ailleurs, $HYDRE_TARGET est le répertoire dans lequel sera installée l'application.

        export HYDRE_TARGET=/repertoire/installation/hydre

Si ce n'est pas déjà fait, téléchargez les sources de HyDre dans $HYDRE_HOME

        git clone https://github.com/jjrom/hydre.git $HYDRE_HOME


Installation de RESTo
---------------------

### Téléchargement des sources

        # Positionnement de la variable $RESTO_HOME
        export RESTO_HOME=/tmp/resto
        rm -Rf $RESTO_HOME

        # Récupération des sources à jour de RESTo v1 
        git clone https://github.com/jjrom/resto-deprecated.git/ $RESTO_HOME
        
        # Positionnement de RESTo sur une version stable pour HyDre
        cd $RESTO_HOME
        git checkout 510deb126fc9b1309468bdad4790d0641349bee3
        

### Installation de la base de données

Effectuez UNIQUEMENT les étapes suivantes de la [procédure d'installation de RESTo] (https://github.com/jjrom/resto/blob/master/README.md) :
* Installation de la [base de données] (https://github.com/jjrom/resto-deprecated/blob/master/README.md#install-resto-database)
* Installation du [Gazetteer] (https://github.com/jjrom/resto-deprecated/blob/master/README.md#install-gazetteer)
* Installation de [iTag] (https://github.com/jjrom/resto-deprecated/blob/master/README.md#install-itag-optional)


Installation de HyDre
--------------------

### Installation de la base de données

        # Creation du schema 'hydre' dans la base de donnees 'resto'
        $HYDRE_HOME/installation/hydreInstallDB.sh -g $HYDRE_HOME -s postgres -F


Déploiement
===========

Lancez le script de deploiement sur le repertoire cible $HYDRE_TARGET

        $HYDRE_HOME/installation/deploy.sh -r $RESTO_HOME -s $HYDRE_HOME -t $HYDRE_TARGET


Configuration
=============

### Apache

Mettre en place un alias vers le repertoire d'installation $HYDRE_TARGET. Pour cela editer le fichier de configuration
apache (généralement /etc/apache2/httpd.conf) et rajouter la règle suivante (en prenant soin de bien remplacer
$HYDRE_TARGET par sa valeur) :

        Alias /hydre/ "$HYDRE_TARGET"
        <Directory "$HYDRE_TARGET">
            Options FollowSymLinks
            AllowOverride All
            Order allow,deny
            Allow from all
        </Directory>

Relancez Apache

        sudo apachectl restart

### mapserver

Editer le fichier $RESTO_HOME/mapserver/hydroweb.map et modifier les valeurs des champs suivants :
* SHAPEPATH
* wms_onlineresource

### RESTo

Editer le fichier $RESTO_TARGET/resto/resto.ini afin de mettre à jour les mots de passe
des comptes 'resto' et 'sresto' [comme expliqué dans la procédure d'installation de RESTo] (https://github.com/jjrom/resto/blob/master/README.md#resto-configuration)

Note : si l'alias Apache est different de /hydre/ (cf. configuration ci-dessus), il faut modifier en conséquence la règle
'RewriteBase' du fichier $RESTO_TARGET/.htaccess 


### Insertion de la collection dans RESTo

Lancez la commande suivante pour ajouter la collection HyDre dans la base de données RESTo

        curl -X POST -F "file[]=@$HYDRE_HOME/installation/Hydre.json" http://admin:nimda@localhost/hydre/

Note : remplacez 'admin:nimda' par le login et le mot de passe du compte admin définis dans RESTo (cf. étape précédente)


Utilisation
===========

### Insertion d'un jeu de données "river"
La commande suivante insère un jeu de données de test sur la station virtuelle JA2 du fleuve Bramaphoutre (Inde)

        curl -X POST -F "file[]=@$HYDRE_HOME/examples/R_bra_bra_JA2_0053_01.txt" http://admin:nimda@localhost/hydre/hydroweb

### Insertion d'un jeu de données "lake"
La commande suivante insère un jeu de données de test sur la station virtuelle du lac Albert (Ouganda/RDC)

        curl -X POST -F "file[]=@$HYDRE_HOME/examples/L_albert.txt" http://admin:nimda@localhost/hydre/hydroweb

### Mise à jour de la description de la collection
Toute modification de la description de la collection (i.e. fichier $HYDRE_HOME/installation/Hydre.json) doit être reportée
dans la base de données RESTo.

Pour cela executez les commandes suivantes :

        # Suppression de la collection - connection à la base de données
        psql -d resto -U postgres << EOF
        DELETE FROM admin.osdescriptions WHERE collection='hydroweb';
        DELETE FROM admin.collections WHERE collection='hydroweb';
        EOF
        
        # Insertion de la collection modifiée
        curl -X POST -F "file[]=@$HYDRE_HOME/installation/Hydre.json" http://admin:nimda@localhost/hydre/

