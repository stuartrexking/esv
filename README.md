# Event Store cluster in Vagrant

 * [Description](#description)
 * [Dependencies](#dependencies)
 * [Setup](#setup)
 * [Observe Processes](#observe-processes)
 * [Create an Event](#create-an-event)

### Description

The Vagrantfile will create a three node [Event Store](https://geteventstore.com/) cluster using [Consul](https://consul.io/) for discovery (via DNS) and [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) for DNS Forwarding.

This Vagrantfile simply builds and runs a basic cluster on Vagrant. It's intention is to show some important pieces working together, primarily a discovery service (consul, but could be etcd or whatever), dns forwarding (using dnsmasq but could be bind or whatever) and a replicated database, in this case [Event Store](https://geteventstore.com/)

### Dependencies

1. Install [Vagrant](https://www.vagrantup.com/)

### Setup

```
git clone https://github.com/stuartrexking/esv
cd esv
vagrant up
```

### Observe Processes

Once the download and installation completes you can login to each node (n1, n2, n3) with

    vagrant ssh n1

Once in you can check the consul cluster

	vagrant@n1:~$ consul members
	Node  Address            Status  Type    Build  Protocol  DC
	n1    172.20.20.10:8301  alive   server  0.5.2  2         dc1
	n2    172.20.20.11:8301  alive   server  0.5.2  2         dc1
	n3    172.20.20.12:8301  alive   server  0.5.2  2         dc1


Check the Event Store cluster from any node

    vagrant@n1:~$ curl http://172.20.20.11:2113/gossip

which will output json containing the three healthy nodes in the cluster

```json
	{
	  "members": [
        {
          "instanceId": "30e3fff4-87f3-4545-8042-46f1affb69a8",
	      "timeStamp": "2015-09-17T10:36:18.536709Z",
          "state": "Slave",
	      "isAlive": true,
          "internalTcpIp": "172.20.20.12",
	      "internalTcpPort": 1112,
          "internalSecureTcpPort": 0
```

You can view the Event Store logs at /var/logs/eventstore

    tail -f /var/log/eventstore/... #logs are partitioned by date

### Create an Event

On any of the nodes create a json file and POST it

    vagrant@n1:~$ echo '[{"eventId":"fbf4a1a1-b4a3-4dfe-a01f-ec52c34e16e4","eventType":"event-type","data":{"a":"1"}}]' >> event.json
	vagrant@n1:~$ curl -i -d @event.json "http://172.20.20.10:2113/streams/newstream" -H "Content-Type:application/vnd.eventstore.events+json"
	HTTP/1.1 201 Created
	Access-Control-Allow-Methods: POST, DELETE, GET, OPTIONS
	Access-Control-Allow-Headers: Content-Type, X-Requested-With, X-Forwarded-Host, X-PINGOTHER, Authorization, ES-LongPoll, ES-ExpectedVersion, ES-EventId, ES-EventType, ES-RequiresMaster, ES-HardDelete, ES-ResolveLinkTo, ES-ExpectedVersion
	Access-Control-Allow-Origin: *
	Access-Control-Expose-Headers: Location, ES-Position
	Location: http://172.20.20.11:2112/streams/newstream/1
	Server: Mono-HTTPAPI/1.0
	Date: Thu, 17 Sep 2015 10:42:32 GMT
	Content-Type: text/plain; charset=utf-8
	Content-Length: 0
	Keep-Alive: timeout=15,max=100%

To then read from the stream as xml from any node

```
vagrant@n3:~$ curl -i -H "Accept:application/atom+xml" "http://172.20.20.10:2113/streams/newstream"
HTTP/1.1 200 OK
Access-Control-Allow-Methods: POST, DELETE, GET, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-Requested-With, X-Forwarded-Host, X-PINGOTHER, Authorization, ES-LongPoll, ES-ExpectedVersion, ES-EventId, ES-EventType, ES-RequiresMaster, ES-HardDelete, ES-ResolveLinkTo, ES-ExpectedVersion
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Location, ES-Position
Cache-Control: max-age=0, no-cache, must-revalidate
Vary: Accept
ETag: "1;-1296467268"
Content-Type: application/atom+xml; charset=utf-8
Server: Mono-HTTPAPI/1.0
Date: Thu, 17 Sep 2015 10:46:35 GMT
Content-Length: 1299
Keep-Alive: timeout=15,max=100

<?xml version="1.0" encoding="utf-8"?><feed xmlns="http://www.w3.org/2005/Atom"><title>Event stream 'newstream'</title><id>http://172.20.20.10:2113/streams/newstream</id><updated>2015-09-17T10:42:32.630844Z</updated><author><name>EventStore</name></author><link href="http://172.20.20.10:2113/streams/newstream" rel="self" /><link href="http://172.20.20.10:2113/streams/newstream/head/backward/20" rel="first" /><link href="http://172.20.20.10:2113/streams/newstream/2/forward/20" rel="previous" /><link href="http://172.20.20.10:2113/streams/newstream/metadata" rel="metadata" /><entry><title>1@newstream</title><id>http://172.20.20.10:2113/streams/newstream/1</id><updated>2015-09-17T10:42:32.630844Z</updated><author><name>EventStore</name></author><summary>event-type</summary><link href="http://172.20.20.10:2113/streams/newstream/1" rel="edit" /><link href="http://172.20.20.10:2113/streams/newstream/1" rel="alternate" /></entry><entry><title>0@newstream</title><id>http://172.20.20.10:2113/streams/newstream/0</id><updated>2015-09-16T10:01:14.792879Z</updated><author><name>EventStore</name></author><summary>event-type</summary><link href="http://172.20.20.10:2113/streams/newstream/0" rel="edit" /><link href="http://172.20.20.10:2113/streams/newstream/0" rel="alternate" /></entry></feed>
```

or json

```
vagrant@n2:~$ curl -i http://172.20.20.10:2113/streams/newstream/0 -H "Accept: application/json"
HTTP/1.1 200 OK
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Headers: Content-Type, X-Requested-With, X-Forwarded-Host, X-PINGOTHER, Authorization, ES-LongPoll, ES-ExpectedVersion, ES-EventId, ES-EventType, ES-RequiresMaster, ES-HardDelete, ES-ResolveLinkTo, ES-ExpectedVersion
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Location, ES-Position
Cache-Control: max-age=31536000, public
Vary: Accept
Content-Type: application/json; charset=utf-8
Server: Mono-HTTPAPI/1.0
Date: Thu, 17 Sep 2015 10:48:36 GMT
Content-Length: 14
Keep-Alive: timeout=15,max=100

{
  "a": "1"
}
```

### TBDs

1. Make cluster accessible from host machine
1. Move this to AWS using [Terraform](https://terraform.io/)
