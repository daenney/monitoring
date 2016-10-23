# Monitoring

This is my home monitoring stack (ya really, for fun).

It consists of:

  * [Prometheus](https://prometheus.io)
  * [Node Exporter](https://github.com/prometheus/node_exporter)
  * [SNMP Exporter](https://github.com/prometheus/snmp_exporter)
  * [cAdvisor](https://github.com/google/cadvisor)
  * [Grafana](http://grafana.org)

In order to make my life simpler and upgrades less of a pain I don't run these
things directly on the host, they're run through Docker containers. As such
there's a `docker-compose.yml` file included that'll spin everything up for you.

There's also a `monitoring.service` you can drop in `/etc/systemd/system` and
after a `systemctl daemon-reload` you'll be able to `systemctl start` and
`enable` the monitoring setup.

It sets up two port mappings:

  * Prometheus on `<HOST>:9090`
  * Grafan on `<HOST>:3000`

It also configures two volumes, `prometheus_data` and `grafana_data` so we can
persist their data outside of the container.

We also mount some configuration in the right places.

## Caveats

The configuration I provide for the SNMP exporter is specific to my home,
you'll need to update the `targets` accordingly to point to your devices.

Also, if you wish to collect this data more often than the default 15s don't
forget to specify an additional `scrape_interval` in the SNMP job.

## Compose magic

There's a piece of magic or two involved that might warrant explanation b/c
of what Docker Compose does for you. Because this `docker-compose.yml` file
lives in a directory called `monitoring` everything Compose creates for you,
networks, containers, volumes etc will be name mangled so that they start with
`monitoring_`.

### Service discovery and networks

Since I have no specific network modes defined it will create a Docker network
named `monitoring_default`. Within this network every container can refer
or lookup any other one by its service name. This is why I can simply have
`snmp-exporter:9116` in the `prometheus.yml` file and things Just Work&#8482;.

### Port mappings

Though I've added `expose` to every service declaration it's not used. Unless
you `docker run ... -P` there will be no automagic mapping between a random
port on the host and the `expose`d port on the container. It's merely there
for documentation purposes at this point. The services that have an
explicit `ports` mapping will be accessible from the host on the port on the
left side of the `:`.

### Volumes

There are two volumes defined, `prometheus_data` and `grafana_data` which have
an empty hash, `{}` as their options. That's because all I need is to ensure
that these volumes are created and persisted. You'll usually be able to find
them on disk at `/var/lib/docker/volumes`.

It's then possible to simply refer to them in `volumes` mappings in each
service by name. That's how the `prometheus_data:/prometheus` mapping works,
it'll bind mount `/var/lib/docker/volumes/monitoring_prometheus_data` onto
`/prometheus` inside the container. The same thing goes for `grafana_data`.

The volume mappings that start with `./` are relative to the path of this
repository, so for snmp-exporter for example we end up mounting the directory
`snmp_exporter` from this repo onto `/etc/snmp_exporter` into the container.
Don't forget the trailing `/`'s on each side otherwise they'll become files.

### Environment files

The Grafana container looks for two environment variables to configure itself,
one to set the admin password, the other enables or disables user signup. By
default compose will look for a `.env` file but I'm not a huge fan of that, so
instead we tell if where to find the file with the environment variables.
