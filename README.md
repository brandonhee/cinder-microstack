# cinder-microstack
Manual installation of cinder with microstack

This is a walkthrough of how to install cinder with [microstack](https://microstack.run/). As of the stein release of microstack, cinder was not fully implemented. Only the database was configured. The official [cinder install guide](https://docs.openstack.org/cinder/latest/install/index.html) provides the basis for this tutorial, but there are a few nuances when using microstack this guide addresses.

You can follow the [microstack docs](https://microstack.run/docs/) for the initial installation. You should be able to access the web interface through 10.20.20.1 before starting this.

These steps were tested on Ubuntu 18.04.

At the end of the this tutorial, you should be able to create volumes and attach them to instances in openstack.

## Verify database and setup cinder user

Microstack uses snap containers. You can either run commands as `microstack.*` or open a shell to the container environment.

### Open a shell to the microstack container
```
sudo snap run --shell microstack.init
source $SNAP_COMMON/etc/microstack.rc
```

### Populate the database
```
root@controller:~# cinder-manage db sync
```

### Verify the database is setup and create a user
```
root@controller:~# mysql -h 127.0.0.1 -P 3306
mysql> use cinder;
mysql> show tables;
```
You should see a list of tables (I had 35).

```
root@controller:~# mysql -h 127.0.0.1 -P 3306
mysql> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'cinder';
Query OK, 0 rows affected, 1 warning (0.05 sec)

mysql> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'cinder';
Query OK, 0 rows affected, 1 warning (0.05 sec)

mysql> show grants for cinder;
+----------------------------------------------------+
| Grants for cinder@%                                |
+----------------------------------------------------+
| GRANT USAGE ON *.* TO 'cinder'@'%'                 |
| GRANT ALL PRIVILEGES ON `cinder`.* TO 'cinder'@'%' |
+----------------------------------------------------+
2 rows in set (0.00 sec)
```
## Create the cinder user for openstack
```
root@controller:~# openstack user create --domain default --password-prompt cinder
root@controller:~# openstack role add --project service --user cinder admin
```
## Create the API Endpoints
The official installation guide sets up the version 2 and version 3 endpoints. Since, v2 is deprecated, you can just setup v3.
```
root@controller:~# microstack.openstack service create --name cinderv3 volumev3
root@controller:~# microstack.openstack endpoint create --region microstack volumev3 public http://192.168.1.15:8776/v3/%\(project_id\)s
root@controller:~# microstack.openstack endpoint create --region microstack volumev3 internal http://192.168.1.15:8776/v3/%\(project_id\)s
root@controller:~# microstack.openstack endpoint create --region microstack volumev3 admin http://1192.168.1.15:8776/v3/%\(project_id\)s
```

## Modify configuration files
The microstack `/snap/microstack/196/etc/cinder/cinder.conf` configuration file can not be edited directly. Instead, use the location `/var/snap/microstack/common/etc/cinder/cinder.conf.d/`. A configuration file is provided that you can work off of.
