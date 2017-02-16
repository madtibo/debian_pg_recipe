# Guide de PostgreSQL sur Debian Jessie 

## Installation de PostgreSQL 

On ajoute le dépôt de la communauté à la liste des sources :
```
# echo 'deb http://apt.postgresql.org/pub/repos/apt/ jessie-pgdg main' > /etc/apt/sources.list.d/postgresql.org.list
# echo $'Package: *\nPin: release o=apt.postgresql.org\nPin-Priority: 500\n' >  /etc/apt/preferences.d/pgdg.pref
# wget -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | apt-key add -
# apt-get update
# apt-get install pgdg-keyring
# apt-get install postgresql-9.6
```

## Configuration de PostgreSQL

Dans le cas d'un serveur dédié à la Base de Données, il peut être intéressant de configurer les options du noyau pour un meilleur fonctionnement avec PostgreSQL :
```
# echo $'vm.overcommit_memory = 2\nvm.zone_reclaim_mode = 0\nvm.swappiness = 5\nvm.dirty_ratio = 2\nvm.dirty_background_ratio = 1\n' >> /etc/sysctl.d/30-postgresql-shm.conf
# sysctl -p /etc/sysctl.d/30-postgresql-shm.conf 
```

Les fichiers de configurations pour l'instance par défaut sont situés dans `/etc/postgresql/9.6/main`. On peut mettre à jour certains paramètres.

Fichier de configuration _postgresql.conf_ :
```
shared_buffers = 128MB   # 1/4 de la mémoire totale, maximum 4GB
effective_cache_size = 512MB   # 1/2 à 2/3 de la mémoire totale pour un serveur dédié à la BdD
listen_addresses = '*'
```

Il est possible de configurer les logs pour mieux suivre le fonctionnement de l'instance :
 * `log_checkpoints = on` : tracer tous les checkpoints réalisés, qu'ils soient déclenchés automatiquement ou manuellement ;
 * `log_lock_waits = on` : tracer toutes les attention d'obtention d'un verrou qui ont duré plus longtemps que deadlock_timeout ;
 * `log_temp_files = 0` : tracer toutes les requêtes qui ont généré des fichiers temporaires, quelle que soit la taille du fichier ;
 * `log_min_duration_statement = 5000` : tracer les requêtes dont le temps d'exécution dépasse la durée indiquée (en millisecondes). Il peut être intéressant de tracer les requêtes longues en production afin de les remonter aux développeurs pour optimisation ou pour tenter de déterminer la cause et le moment d'un ralentissement global du système ;
 * `log_autovacuum_min_duration = 0` : tracer toutes les actions déclenchées par l'autovacuum.
 * `log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d '` : indique le contenu du préfixe de chaque ligne de trace, à savoir timestamp [PID de la session]: [numéro de ligne de la session-1] user=<nom de l'utilisateur>,db=<base de données connectée>
 * `lc_messages = 'C'` : permet d'avoir les messages dans les trace en anglais.
 * `log_directory = '/var/log/postgresql'` : écrire les logs dans un répertoire dédié, non relatif au PGDATA ;
 * `log_filename = postgresql-<version>-<instance>.log` : écrire dans un fichier portant un nom ne contenant pas de variables afin de pouvoir utiliser logrotate (on peut également inclure le nom de l'application ou de l'instance dans le nom du fichier si besoin) ;
 * `log_rotation_age = 0` : désactiver la rotation de PostgreSQL liée à l'âge du fichier ;
 * `log_rotation_size = 0` : désactiver la rotation de PostgreSQL liée à la taille du fichier ;
 * `log_truncate_on_rotation = off` : désactiver l'écrasement du fichier à la rotation.


## Exploitation de PostgreSQL

Le service PostgreSQL est démarré automatiquement au lancement du système. Le système d’initialisation systemd permet de contrôler l'instance, les arguments possibles sont:
 * _start_ : démarrer le service,
 * _stop_ : arréter le service,
 * _reload_ : recharger la configuration,
 * _restart_ : redémarrer le service.

La commande `systemctl` est utilisée pour agir sur l'instance:
```
# systemctl <commande> postgresql@<version>-<instance>.service
```

Pour notre instance, la commande de rechargement de la configuration est :
```
# systemctl reload postgresql@9.6-main.service
```

Les journaux applicatifs (logs) sont écrits dans le fichier /var/log/postgresql/postgresql-<version>-<instance>.log.
Ces log sont archivés automatiquement toutes les semaines et supprimés au bout de 10 semaines. Cette configuration doit-être revue si les niveaux de logs de l'application sont remontés (fichier `/etc/logrotate.d/postgresql-common`) :
```
/var/log/postgresql/postgresql-*.log {
  daily
  rotate 10
  copytruncate
  delaycompress
  compress
  notifempty
  missingok
  create 0660 postgres postgres
  postrotate
    su -l postgres -c 'psql -c "select pg_rotate_logfile()" -o /dev/null'
  endscript
}
```
