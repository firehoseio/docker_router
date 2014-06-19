# firehose-router

A Docker container for proxying HTTP and WebSocket connections to multiple Firehose instances. Firehose instances are dynamically added to and removed from the nginx configuration using `confd`. Configuration for Firehose nginx is stored in etcd at `/firehose/nginx` and `/firehose/upstream`.

## Manipulating Firehose Nodes

Firehose instances are added to etcd as a key to `/firehose/upstream/<key_name>` where `<key_name>` is some identifier (such as the Docker instance name). The value of the key is the host IP and container NAT port combination, such as `172.16.32.143:43876`.

You can add a node using `etcdctl`:

    $ etcdctl set /firehose/upstream/firehose-01 172.16.32.90:39283
    172.16.32.90:39283


Removing a node is just deleting the key:

    $ etcdctl rm /firehose/upstream/firehose-01

## Sidekick Unit

Manipulating `/firehose/upstream` keys is usually done through a sidekick systemd unit. This unit would bind to the Firehose unit and start / stop with it. When started, it would lookup the Firehose instance NAT port and use the host IP address to populate the etcd key. When the unit stops, it would remove the key.

An example of this kind of systemd unit might look like:

    [Unit]
    Description=Firehose etcd announcer
    BindsTo=firehose@%i.service

    [Service]
    TimeoutStartSec=30s
    ExecStartPre=/bin/sh -c "echo 'Sleeping for 5 seconds for firehose-%i to startup...'; sleep 5"
    ExecStart=/bin/sh -c "port=$(docker inspect -f '{{range $i, $e := .HostConfig.PortBindings }}{{$p := index $e 0}}{{$p.HostPort}}{{end}}' firehose-%i); echo -n \"Adding socket $COREOS_PRIVATE_IPV4:$port/tcp to /firehose/upstream/firehose-%i\"; while netstat -lnt | grep :$port >/dev/null; do etcdctl set /firehose/upstream/firehose-%i $COREOS_PRIVATE_IPV4:$port --ttl 60 >/dev/null; sleep 45; done"
    ExecStop=/bin/sh -c "echo -n \"Removing etcd key /firehose/instance/firehose-%i\"; etcdctl rm --recursive /firehose/instance/firehose-%i

    [X-Fleet]
    X-ConditionMachineOf=firehose@%i.service
