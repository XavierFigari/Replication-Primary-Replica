# Replication Primary-Replica
Database replication using Primary/Replica scheme

## Docker Containers
- `sql-primary` : primary database
- `sql-replica01` : replica database
- `wordpress` : holds the Wordpress application

Database replication using Primary/Replica scheme

Docker Containers
-----------------

*   sql-primary : primary database
*   sql-replica01 : replica database
*   wordpress : holds the Wordpress application

Configure the Primary db
------------------------

Créer un fichier de conf `conf_primary.cnf` à copier dans `/etc/my.cnf`

```text-plain
[mariadb]
log-bin
server_id=1
log-basename=primary1
binlog-format=mixed
```

Dans docker-compose :

```text-plain
  mariadb:
    container_name: sql-primary
    image: mariadb:11.4
    restart: unless-stopped
    volumes:
      - /srv/docker/sql-primary:/var/lib/mysql
      - ./conf_primary.cnf:/etc/my.cnf
```

Pour créer le replication user, on peut :

*   soit créer un fichier SQL `create_replication_user.sql` et l'appeler dans le docker-compose dans le init script :

    ```text-plain
    CREATE USER 'replication_user'@'%' IDENTIFIED BY 'titi';
    GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
    ```

*   soit mettre directement les variables d'environnement suivantes dans le Docker compose de l'image MariaDB (partie sql-primary) :

    ```text-plain
          - MARIADB_REPLICATION_USER=replication_user
          - MARIADB_REPLICATION_PASSWORD=titi
    ```


Configuring the MariaDB Replica
--------------------------------

Idem, on pourrait créer un fichier SQL et le charger au démarrage du container :

```text-x-sql
CHANGE MASTER TO
    MASTER_HOST='sql-primary',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='titi',
    MASTER_USE_GTID = slave_pos;
```

Mais c'est plus simple de mettre des variables d'environnement dans le Docker Compose :

```text-x-dockerfile
  sql-replica01:
    container_name: sql-replica01
    image: mariadb:11.4
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=toto
      - MARIADB_REPLICATION_USER=replication_user
      - MARIADB_REPLICATION_PASSWORD=titi
      - MARIADB_MASTER_HOST=sql-primary
```

Running the replica on the slave
--------------------------------

Si on a pris l'option “fichiers SQL à charger à l'init”, il faut faire en plus dans le slave :

```text-x-sql
START SLAVE;
SHOW SLAVE STATUS \G
```

Sinon, si on a défini `MARIADB_MASTER_HOST` dans le Docker Compose, il le fait tout seul à notre place.

If replication is working correctly, both the values of `Slave_IO_Running` and `Slave_SQL_Running` should be `Yes`:

```text-x-sql
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

Make sure it works
------------------

*   create a post on Wordpress web interface (http://localhost:8080) : Titre du post : “Il fait beau”
*   check database on Primary
*   check database on Replica : it shoud be the same

Example :

```text-plain
> docker exec -it sql-replica01 mariadb -u root -ptoto -e 'use wpdb; select post_title from wp_posts;'
```

On doit avoir la même chose des deux côtés (database Primary et Replica) :

```text-plain
> docker exec -it sql-replica01 mariadb -u root -ptoto -e 'use wpdb; select post_title from wp_posts;'
+-------------------------------+
| post_title                    |
+-------------------------------+
| Bonjour tout le monde !       |
| Page d’exemple                |
| Politique de confidentialité  |
| Brouillon auto                |
| Il fait beau                  |
| Custom Styles                 |
| Il fait beau                  |
+-------------------------------+
7 rows in set (0.000 sec)
```