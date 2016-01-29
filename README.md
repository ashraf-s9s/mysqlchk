# mysqlchk

Based on ClusterControl's mysqlchk script but built for MySQL Replication. It detects the MySQL replication role on the database node as per below:

* if master (``SHOW SLAVE HOSTS > 1`` AND ``read_only = OFF``),
 * return 'MySQL master is running.'

* if slave (``Slave_IO_Running = Yes`` AND ``Slave_SQL_Running = Yes`` AND (``Seconds_Behind_Master = 0`` OR ``Seconds_Behind_Master < SLAVE_LAG_LIMIT``))
 * return 'MySQL slave is running. (slave lag: 0)'

* else
 * return 'MySQL is \*down\*'

# How to use

0) Deploy or add existing MySQL Replication setup into ClusterControl

1) Install HAProxy on the your selected node. ClusterControl will install the default mysqlchk that need to be replaced with this one on each of the database node. By default, it's located under ``/usr/local/sbin/mysqlchk``

2) Grab the file from this repo and replace it.

3) Update require information (``SLAVE_LAG_LIMIT`` is in seconds):
```bash
SLAVE_LAG_LIMIT=5
MYSQL_HOST="localhost"
MYSQL_PORT="3306"
MYSQL_USERNAME='root'
MYSQL_PASSWORD='password'
```
4) You are good to go. Now ensure in HAProxy you have 2 listeners (3307 for write, 3308 for read) and use ``option tcp-check`` & ``tcp-check expect`` to distinguish master/slave similar to example below:
```bash
listen  haproxy_192.168.55.110_3307_write
        bind *:3307
        mode tcp
        timeout client  10800s
        timeout server  10800s
        balance leastconn
        option tcp-check
        tcp-check expect string MySQL\ master
        option allbackups
        default-server port 9200 inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
        server 192.168.55.111 192.168.55.111:3306 check
        server 192.168.55.112 192.168.55.112:3306 check
        server 192.168.55.113 192.168.55.113:3306 check

listen  haproxy_192.168.55.110_3308_read
        bind *:3308
        mode tcp
        timeout client  10800s
        timeout server  10800s
        balance leastconn
        option tcp-check
        tcp-check expect string is\ running.
        option allbackups
        default-server port 9200 inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
        server 192.168.55.111 192.168.55.111:3306 check
        server 192.168.55.112 192.168.55.112:3306 check
        server 192.168.55.113 192.168.55.113:3306 check
```
