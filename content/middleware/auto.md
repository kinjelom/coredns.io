+++
title = "auto"
description = "*auto* enables serving zone data from an RFC 1035-style master file which is automatically picked up from disk. "
weight = 1
tags = [  "middleware" , "auto" ]
categories = [ "middleware" ]
date = "2017-07-26T08:45:58+01:00"
+++

The *auto* middleware is used for an "old-style" DNS server. It serves from a preloaded file that exists
on disk. If the zone file contains signatures (i.e. is signed, i.e. DNSSEC) correct DNSSEC answers
are returned. Only NSEC is supported! If you use this setup *you* are responsible for resigning the
zonefile. New zones or changed zone are automatically picked up from disk.

## Syntax

~~~
auto [ZONES...] {
    directory DIR [REGEXP ORIGIN_TEMPLATE [TIMEOUT]]
    no_reload
    upstream ADDRESS...
}
~~~

**ZONES** zones it should be authoritative for. If empty, the zones from the configuration block
are used.

* `directory` loads zones from the speficied **DIR**. If a file name matches **REGEXP** it will be
  used to extract the origin. **ORIGIN_TEMPLATE** will be used as a template for the origin. Strings
  like `{<number>}` are replaced with the respective matches in the file name, i.e. `{1}` is the
  first match, `{2}` is the second, etc.. The default is: `db\.(.*)  {1}` e.g. from a file with the
  name `db.example.com`, the extracted origin will be `example.com`. **TIMEOUT** specifies how often
  CoreDNS should scan the directory, the default is every 60 seconds. This value is in seconds.
  The minimum value is 1 second.
* `no_reload` by default CoreDNS will reload a zone from disk whenever it detects a change to the
  file. This option disables that behavior.
* `upstream` defines upstream resolvers to be used resolve external names found (think CNAMEs)
  pointing to external names.

All directives from the *file* middleware are supported. Note that *auto* will load all zones found,
even though the directive might only receive queries for a specific zone. I.e:

~~~
auto example.org {
    directory /etc/coredns/zones
}
~~~
Will happily pick up a zone for `example.COM`, except it will never be queried, because the *auto*
directive only is authoritative for `example.ORG`.

## Examples

Load `org` domains from `/etc/coredns/zones/org` and allow transfers to the internet, but send
notifies to 10.240.1.1

~~~
auto org {
    directory /etc/coredns/zones/org
    transfer to *
    transfer to 10.240.1.1
}
~~~

Load `org` domains from `/etc/coredns/zones/org` and looks for file names as `www.db.example.org`,
where `example.org` is the origin. Scan every 45 seconds.

~~~
auto org {
    directory /etc/coredns/zones/org www\.db\.(.*) {1} 45
}
~~~
