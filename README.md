# cinder-microstack
Manual installation of cinder with microstack

This is a walkthrough of how to install cinder with [microstack](https://microstack.run/). As of the stein release of microstack, cinder was not fully implemented. Only the database was configured. The official [cinder install guide](https://docs.openstack.org/cinder/latest/install/index.html) provides the basis for this tutorial, but there are a few nuances when using microstack this guide addresses.

You can follow the [microstack docs](https://microstack.run/docs/) for the initial installation.

These steps were tested on Ubuntu 18.04.

At the end of the this tutorial, you should be able to create volumes and attach them to instances in openstack.

## Verify database and setup cinder user

Microstack uses snap containers. You can either run commands as `microstack.*` or open a shell to the container environment.

```
sudo snap run --shell microstack.init
source $SNAP_COMMON/etc/microstack.rc
```

