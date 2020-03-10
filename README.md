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
root@controller:~# openstack service create --name cinderv3 volumev3
root@controller:~# openstack endpoint create --region microstack volumev3 public http://192.168.1.15:8776/v3/%\(project_id\)s
root@controller:~# openstack endpoint create --region microstack volumev3 internal http://192.168.1.15:8776/v3/%\(project_id\)s
root@controller:~# openstack endpoint create --region microstack volumev3 admin http://1192.168.1.15:8776/v3/%\(project_id\)s
```

## Modify configuration files
The microstack `/snap/microstack/196/etc/cinder/cinder.conf` configuration file can not be edited directly. Instead, use the location `/var/snap/microstack/common/etc/cinder/cinder.conf.d/`. A configuration file is provided that you can work off of (change the IP at least). Copy the config to the common directory.

You should also verify the uwsgi configuration file is there.
```
root@controller:~# cat /var/snap/microstack/common/etc/cinder/uwsgi/snap/cinder-api.ini
[uwsgi]
wsgi-file = /snap/microstack/196/bin/cinder-wsgi
uwsgi-socket = /var/snap/microstack/common/run/cinder-api.sock
buffer-size = 65535
master = true
enable-threads = true
processes = 4
thunder-lock = true
lazy-apps = true
home = /snap/microstack/196/usr
pyargv = --config-file=/snap/microstack/196/etc/cinder/cinder.conf --config-dir=/var/snap/microstack/common/etc/cinder/cinder.conf.d
```

## Restart services and verify API is working
From outside the snap environment.
```
sudo systemctl restart snap.microstack.*
```

Now **we have to start the cinder uwsgi server!** Microstack has the cinder snaps installed, but they are now started. So, we have to do it manually. You can later add this to your rc.local or use cron to automatically start this. For now, it good to do it manually so you can see if there are any errors.

```
sudo sudo microstack.cinder-uwsgi
```

You should now be able to go to http:{YOUR_IP}:8776 and see some JSON data.
```
{"versions": [{"id": "v3.0", "status": "CURRENT", "version": "3.59", "min_version": "3.0", "updated": "2018-07-17T00:00:00Z", "links": [{"rel": "describedby", "type": "text/html", "href": "https://docs.openstack.org/"}, {"rel": "self", "href": "http://192.168.1.15:8776/v3/"}], "media-types": [{"base": "application/json", "type": "application/vnd.openstack.volume+json;version=3"}]}]}
```
This means the endpoint is up and running.

## Cinder-scheduler
Similar to cinder-uwsgi, we have to manually start the service.
```
sudo microstack.cinder-scheduler
```

We can verify our services are up and running. From with the environement. 
```
root@node1:~# cinder service-list
+------------------+-----------+------+---------+-------+----------------------------+-----------------+
| Binary           | Host      | Zone | Status  | State | Updated_at                 | Disabled Reason |
+------------------+-----------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | node1     | nova | enabled | up    | 2020-03-10T23:01:39.000000 | -               |
+------------------+-----------+------+---------+-------+----------------------------+-----------------+

```
We are not quite ready to create volume. Cinder needs to know where the blocks will be stored!

## Cinder-volume
I'm going to do these steps on the same machine as the controller. The disk has been partitioned into an OS and a free space that will be used with cinder. Reference the [official guide](https://docs.openstack.org/cinder/latest/install/cinder-storage-install-ubuntu.html) if you have issues.

```
apt install lvm2 thin-provisioning-tools
pvcreate /dev/sda2  #use your partition!
vgcreate cinder-volumes /dev/sda2
```

Modify `/etc/lvm/lvm.conf` so that 
```
devices {
...
filter = [ "a/sda/", "a/sda2/", "r/.*/"]
```

Your filter may be different depending on your partitions and whether the OS is running on the same disk.

Now run

```
service tgt restart
sudo microstack.cinder-volume
```

I ended up getting an error running `microstack.cinder-volume`. It was related to `lvcreate`, so I just ran the command that failed manually and it seemed to fix the problem. For me it was
`
lvcreate -T -L 77.881g cinder-volumes/cinder-volumes-pool
`
Try to start the service again `sudo microstack.cinder-volume`.

We can verify our services are up and running. From with the environement:
```
root@controller:~# cinder service-list
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
| Binary           | Host           | Zone | Status  | State | Updated_at                 | Disabled Reason |
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller     | nova | enabled | up    | 2020-03-10T23:01:39.000000 | -               |
| cinder-volume    | controller@lvm | nova | enabled | up    | 2020-03-10T23:01:38.000000 | -               |
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
```

## Try to create a volume
```
root@controller:~# cinder create 1
+--------------------------------+--------------------------------------+
| Property                       | Value                                |
+--------------------------------+--------------------------------------+
| attachments                    | []                                   |
| availability_zone              | nova                                 |
| bootable                       | false                                |
| consistencygroup_id            | None                                 |
| created_at                     | 2020-03-10T22:47:58.000000           |
| description                    | None                                 |
| encrypted                      | False                                |
| id                             | 3fa0e889-db3c-4f2d-a9d8-642da8b273a9 |
| metadata                       | {}                                   |
| migration_status               | None                                 |
| multiattach                    | False                                |
| name                           | None                                 |
| os-vol-host-attr:host          | controller@lvm#LVM                   |
| os-vol-mig-status-attr:migstat | None                                 |
| os-vol-mig-status-attr:name_id | None                                 |
| os-vol-tenant-attr:tenant_id   | 4d96c87743d54c8fafd10d5dc010118c     |
| replication_status             | None                                 |
| size                           | 1                                    |
| snapshot_id                    | None                                 |
| source_volid                   | None                                 |
| status                         | creating                             |
| updated_at                     | 2020-03-10T22:47:58.000000           |
| user_id                        | f58a3912abae491482724ad3aeaabec9     |
| volume_type                    | None                                 |
+--------------------------------+--------------------------------------+
root@controller:~# cinder list
+--------------------------------------+-----------+------+------+-------------+----------+-------------+
| ID                                   | Status    | Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+------+------+-------------+----------+-------------+
| 3fa0e889-db3c-4f2d-a9d8-642da8b273a9 | available | -    | 1    | -           | false    |             |
+--------------------------------------+-----------+------+------+-------------+----------+-------------+
```

If the status gets stuck in creating or error, then something went wrong. Go back through these steps and see if anything was missed. Make sure cinder-uwsgi, cinder-schedule, and cinder-volume are all running. You can also try restarting the microstack services and then run cinder-uwsgi, cinder-schedule, and cinder-volume again.


## Setup volume on another node
```
apt install lvm2 thin-provisioning-tools
pvcreate /dev/sda2  #use your partition!
vgcreate cinder-volumes /dev/sda2
```

Modify `/etc/lvm/lvm.conf` so that 
```
devices {
...
filter = [ "a/sda/", "a/sda2/", "r/.*/"]
```
Your filter may be different depending on your partitions and whether the OS is running on the same disk.

Copy the provided config (cinder-node.conf) to `/var/snap/microstack/common/etc/cinder/cinder.conf.d/`
Now run

```
service tgt restart
sudo microstack.cinder-volume
```

I ended up getting an error running `microstack.cinder-volume`. It was related to `lvcreate`, so I just ran the command that failed manually and it seemed to fix the problem. For me it was
`
lvcreate -T -L 77.881g cinder-volumes/cinder-volumes-pool
`
Try to start the service again `sudo microstack.cinder-volume`.

We can verify our services are up and running. From with the environement:
```
root@node1:~# cinder service-list
+------------------+-----------+------+---------+-------+----------------------------+-----------------+
| Binary           | Host      | Zone | Status  | State | Updated_at                 | Disabled Reason |
+------------------+-----------+------+---------+-------+----------------------------+-----------------+
| cinder-volume    | node1@lvm | nova | enabled | up    | 2020-03-10T23:01:38.000000 | -               |
+------------------+-----------+------+---------+-------+----------------------------+-----------------+
```

If we go back to the **controller**, hopefully the new node can be seen!
```
root@controller:~# cinder service-list
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
| Binary           | Host           | Zone | Status  | State | Updated_at                 | Disabled Reason |
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller     | nova | enabled | up    | 2020-03-10T23:01:39.000000 | -               |
| cinder-volume    | controller@lvm | nova | enabled | up    | 2020-03-10T23:01:38.000000 | -               |
| cinder-volume    | node1@lvm      | nova | enabled | up    | 2020-03-10T23:01:38.000000 | -               |
+------------------+----------------+------+---------+-------+----------------------------+-----------------+
```
