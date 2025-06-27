salt-stack runs the salt minion deployment so that you can connect/manage other minions.

It sends all its events to a mysql database called mariadb, for event sourcing and it is what the front end uses to show this data.

Alcali is used as a front end to view past event and also to send commands via the salt-cli. access via nodeport: http://localhost:30080

Setup:

salt-master does not have a dedicated docker image any more, so one has to be build manually. This is currently done locally and is pointing to my local image.
You have to add the image to the k8s containderd via running docker save salttake3-master:latest | sudo k3s ctr images import -


The Dockerfile does the following:
install salt-master, salt-minion and salt-api
Install a mysqlclient
copy some base master/ srv/salt and srv/pillar configs
run saltutil.sync_all -> this sets it so that alcali is configured as an authenticator (Note that the master config needs auth_dirs: [/srv/salt/auth] as that is where the alcali auth config currently lives, there are other /srv/salt files needed too)
(This is in another dockerfile, it might needs to be added to this: RUN salt-call --local tls.create_self_signed_cert cacert_path='/etc/pki'
This is so that a self signed cert runs so that alcali sends to https. Alternatively we need to set cherryPi to have ssl disabled)

k8s setup:

MariaDB has a config map which has the SQL init script. This is mounted into /docker-entrypoint-initdb.d which is automatically run by mysql on startup.
MariaDB has the secrets for the DB
MariaDB has a service exposed (mysql port 3306)
MariaDB is a stateful set with svc mariadb-data

Alcali has a config map and secret which are loaded as ENV vars into the containers. Includes DB details and salt api details, as well as secrets.
Alcali is a deployment, however it needs to run a command just on the very first deployment which sends a migrate command to the DB, and also saves its admin user to the DB. This is run on the init container alcali-initialisation, however it is wrapped in an if-block, as the init container starts every time, but in most instances other than the first, doesn't need to be run.
You hit alcali on internal port 8000, which is currently a Node Service exposed on 30080 (i.e. http://localhost:30080)

Salt-master has a service file which exposes 8080 (salt-api), 4505 and 4506 which the salt minions hit (not yet tested with external minions)
salt-master is a stateful set with the svc pki-data which stores all the keys for the accepted salt minions.
It also has an init container git-clone-salt-config, which will run a git clone to obtain up-to-date master, state and pillar config. This is then mounted into the main container at the correct file paths.

Currently salt-master config is set up with:
A http server (rest_cherrypy) on port 8080, which has a self signed certificate (might need to be disabled or check the cert is actually auto created during image creation...). This is the salt-api which alcali talks to.
netapi_enable_clients enabled for salt-api to also work
Config to send data to database and alcali
auth_dirs: [/srv/salt/auth] (for the alcali auth - this is loaded into cache during the docker build?)
How to limit what alcali users can do is defined in the following configs, where admin user can do all:
```
eauth_acl_module: alcali
keep_acl_in_token: true

external_auth:
  alcali:
    admin:
      - '.*'
      - '@local'
      - '@runner'
      - '@wheel'
      - '@jobs'
    apiuser:
      - test.ping
      - minion\*:
        - network.*
      - '@wheel':
        - key.list_all
      - minion2:
        - network.*
        - state.*
```


Todo: Remove debug mode on salt-master
Check that all works on fresh startup (notably the salt certs are actually generated)
Use a fully pushed image rather than local image
Tidy up docker image?
Try running with multiple salt-masters