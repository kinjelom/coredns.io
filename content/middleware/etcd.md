+++
title = "etcd"
description = "*etcd* enables reading zone data from an etcd instance. The data in etcd has to be encoded as a [message](https://github.com/skynetservices/skydns/blob/2fcff74cdc9f9a7dd64189a447ef27ac354b725f/msg/service.go#L26) like [SkyDNS](https://github.com/skynetservices/skydns). It should also work just like SkyDNS. "
weight = 10
tags = [  "middleware" , "etcd" ]
categories = [ "middleware" ]
date = "2017-07-26T08:45:58+01:00"
+++

The etcd middleware makes extensive use of the proxy middleware to forward and query other servers
in the network.

## Syntax

~~~
etcd [ZONES...]
~~~

* **ZONES** zones etcd should be authoritative for.

The path will default to `/skydns` the local etcd proxy (http://localhost:2379).
If no zones are specified the block's zone will be used as the zone.

If you want to `round robin` A and AAAA responses look at the `loadbalance` middleware.

~~~
etcd [ZONES...] {
    stubzones
    path PATH
    endpoint ENDPOINT...
    upstream ADDRESS...
    tls CERT KEY CACERt
    debug
}
~~~

* `stubzones` enables the stub zones feature. The stubzone is *only* done in the etcd tree located
    under the *first* zone specified.
* **PATH** the path inside etcd. Defaults to "/skydns".
* **ENDPOINT** the etcd endpoints. Defaults to "http://localhost:2397".
* `upstream` upstream resolvers to be used resolve external names found in etcd (think CNAMEs)
  pointing to external names. If you want CoreDNS to act as a proxy for clients, you'll need to add
  the proxy middleware. **ADDRESS** can be an IP address, and IP:port or a string pointing to a file
  that is structured as /etc/resolv.conf.
* `tls` followed by:
  * no arguments, if the server certificate is signed by a system-installed CA and no client cert is needed
  * a single argument that is the CA PEM file, if the server cert is not signed by a system CA and no client cert is needed
  * two arguments - path to cert PEM file, the path to private key PEM file - if the server certificate is signed by a system-installed CA and a client certificate is needed
  * three arguments - path to cert PEM file, path to client private key PEM file, path to CA PEM file - if the server certificate is not signed by a system-installed CA and client certificate is needed
* `debug` allows for debug queries. Prefix the name with `o-o.debug.` to retrieve extra information in the
  additional section of the reply in the form of TXT records.

## Examples

This is the default SkyDNS setup, with everying specified in full:

~~~
.:53 {
    etcd skydns.local {
        stubzones
        path /skydns
        endpoint http://localhost:2379
        upstream 8.8.8.8:53 8.8.4.4:53
    }
    prometheus
    cache 160 skydns.local
    loadbalance
    proxy . 8.8.8.8:53 8.8.4.4:53
}
~~~

Or a setup where we use `/etc/resolv.conf` as the basis for the proxy and the upstream
when resolving external pointing CNAMEs.

~~~
.:53 {
    etcd skydns.local {
        path /skydns
        upstream /etc/resolv.conf
    }
    cache 160 skydns.local
    proxy . /etc/resolv.conf
}
~~~


### Reverse zones

Reverse zones are supported. You need to make CoreDNS aware of the fact that you are also
authoritative for the reverse. For instance if you want to add the reverse for 10.0.0.0/24, you'll
need to add the zone `0.0.10.in-addr.arpa` to the list of zones. (The fun starts with IPv6 reverse zones
in the ip6.arpa domain.) Showing a snippet of a Corefile:

~~~
    etcd skydns.local 0.0.10.in-addr.arpa {
        stubzones
    ...
~~~

Next you'll need to populate the zone with reverse records, here we add a reverse for
10.0.0.127 pointing to reverse.skydns.local.

~~~
% curl -XPUT http://127.0.0.1:4001/v2/keys/skydns/arpa/in-addr/10/0/0/127 \
    -d value='{"host":"reverse.skydns.local."}'
~~~

Querying with dig:

~~~
% dig @localhost -x 10.0.0.127 +short
reverse.atoom.net.
~~~

Or with *debug* queries enabled:

~~~
% dig @localhost -p 1053 o-o.debug.127.0.0.10.in-addr.arpa. PTR

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;o-o.debug.127.0.0.10.in-addr.arpa. IN  PTR

;; ANSWER SECTION:
127.0.0.10.in-addr.arpa. 300    IN      PTR     reverse.atoom.net.

;; ADDITIONAL SECTION:
127.0.0.10.in-addr.arpa. 300    CH      TXT     "reverse.atoom.net.:0(10,0,,false)[0,]"
~~~

## Debug queries

When debug queries are enabled CoreDNS will return errors and etcd records encountered during the resolution
process in the response. The general form looks like this:

    skydns.test.skydns.dom.a.	0	CH	TXT	"127.0.0.1:0(10,0,,false)[0,]"

This shows the complete key as the owername, the rdata of the TXT record has:
`host:port(priority,weight,txt content,mail)[targetstrip,group]`.

Errors when communicating with an upstream will be returned as: `host:0(0,0,error message,false)[0,]`.

An example:

    www.example.org.	0	CH	TXT	"www.example.org.:0(0,0, IN A: unreachable backend,false)[0,]"

Signalling that an A record for www.example.org. was sought, but it failed with that error.

Any errors seen doing parsing will show up like this:

    . 0 CH TXT "/skydns/local/skydns/r/a: invalid character '.' after object key:value pair"

which shows `a.r.skydns.local.` has a json encoding problem.
