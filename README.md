# Replication Primary-Replica
Database replication using Primary/Replica scheme

## Docker Containers
- `sql-primary` : primary database
- `sql-replica01` : replica database
- `wordpress` : holds the Wordpress application

To start the containers, simply run in the same directory as docker-compose.yml :

`docker compose up -d`

## Configure the Primary db

Create a configuration file `conf_primary.cnf` that will be copied to `/etc/my.cnf`

```text-plain
[mariadb]
log-bin
server_id=1
log-basename=primary1
binlog-format=mixed
```

Inside docker-compose, give access to this file with a *volume* definition :

```yaml
  mariadb:
    container_name: sql-primary
    image: mariadb:11.4
    restart: unless-stopped
    volumes:
      - /srv/docker/sql-primary:/var/lib/mysql
      - ./conf_primary.cnf:/etc/my.cnf
```

To create the replication user, we can :

*   Either create an SQL file `create_replication_user.sql` and call it from docker-compose in the init script :

    ```SQL
    CREATE USER 'replication_user'@'%' IDENTIFIED BY 'titi';
    GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
    ```

*   or set directly the environnement variables inside docker-compose, for the MariaDB image (sql-primary) :

    ```yaml
          - MARIADB_REPLICATION_USER=replication_user
          - MARIADB_REPLICATION_PASSWORD=titi
    ```


## Configuring the MariaDB Replica

Again, we could create an SQL file and execute it at container boot :

```SQL
CHANGE MASTER TO
    MASTER_HOST='sql-primary',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='titi',
    MASTER_USE_GTID = slave_pos;
```

But it's much simpler to set environment variables in the docker-compose file :

```yaml
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

## Running the replica on the slave

If you've selected the “SQL files to be loaded at init” option, you'll also need to do the following in the slave :

```sql
START SLAVE;
SHOW SLAVE STATUS \G
```

Otherwise, if you have defined `MARIADB_MASTER_HOST` in the Docker Compose file, it will do it for you.

If replication is working correctly, both the values of `Slave_IO_Running` and `Slave_SQL_Running` should be `Yes`:

```sql
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

## Make sure it works

*   create a post on Wordpress web interface (http://localhost:8080) : Titre du post : “Il fait beau”
*   check database on Primary
*   check database on Replica : it shoud be the same

Example :

```bash
docker exec -it sql-replica01 mariadb -u root -ptoto -e 'use wpdb; select post_title from wp_posts;'
```

We must have the same thing on both sides (Primary and Replica databases):

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
