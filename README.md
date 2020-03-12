# dnsredir

## Name

_dnsredir_ - yet another seems better forward/proxy plugin for CoreDNS.

*dnsredir* plugin works just like the *forward* plugin which re-uses already opened sockets to the upstreams. Currently, it supports `UDP`, `TCP`, and `DNS-over-TLS` and uses in continuous health checking.

Like the *proxy* plugin, it also supports multiple backends, which each upstream also supports multiple TLS server names. Load balancing features including multiple policies, health checks and failovers.

The health check works by sending `. IN NS` to upstream host, somewhat like a ping packet in `ICMP` protocol. Any response that is not a network error(for example, `REFUSED`, `SERVFAIL`, etc.) is taken as a healthy upstream.

When all upstream hosts are down this plugin can opt fallback to randomly selecting a upstream host and sending the requests to it as last resort.

## Syntax

The phrase *redirect* and *forward* can be used interchangeably, unless explicitly stated otherwise.

In its most basic form, a simple DNS redirecter uses the following syntax:

```Corefile
dnsredir FILE... {
	to TO...
}
```

* `FILE...` is the file list which contains base domain to match for the request to be redirected. `.`(i.e. root zone) can be used solely to match all incoming requests as a fallback.

	Currently, two kind of formats are supported:

	1. `DOMAIN`, which the whole line is the domain name.

	2. `server=/DOMAIN/...`, which is the format of `dnsmasq` config file, note that only the `DOMAIN` will be honored, other fields will be simply discarded.

	Text after `#` character will be treated as comment.

	Unparsable lines(including whitespace-only line) are therefore just ignored.

* `to TO...` are the destination endpoints to redirected to.

	The `to` syntax allows you to specify a protocol, a port, etc:

	`[dns://]IP[:PORT]`(or no protocol) for plain DNS.

	`tls://IP[:PORT][@TLS_SERVER_NAME]` for DNS over TLS, if you combine `:` and `@`, `@` must comes last. Be aware of some DoT servers require TLS server name as mandatory option.

An expanded syntax can be utilized to unleash of power of `dnsredir` plugin:

```Corefile
dnsredir FILE... {
	reload DURATION
	[INLINE]
	except IGNORED_NAME...

	spray
	policy random|round_robin|sequential
	health_check DURATION
	max_fails INTEGER

	to TO...
	expire DURATION
	force_tcp
	prefer_udp
	tls CERT KEY CA
	tls_servername NAME
}
```

Some of the knobs take a `DURATION` as argument, *second* will be used as default time unit as if you don't specify it, **zero time duration to disable the feature** unless it's explicitly stated.

* `FILE...` and `to TO...` as above.

* `reload` change the interval between each `FILE...` reload, zero to disable the feature. Default is `2s`, minimal is `1s`.

* `INLINE` are the domain names sealed in `Corefile`, they serve as supplementaries. Note that domain names in `FILE...` will still be read. `INLINE` is forbidden if you specify `.`(i.e. root zone) as `FILE...`.

* `except` is a space-separated list of domains to exclude from redirecting. Requests that match none of these names will be passed through.

* `spray` when all upstreams in `to` are marked as unhealthy, randomly pick one to send the traffic to. (Last resort, as a failsafe.)

* `policy` specifies the policy to use for selecting upstream hosts. The default is `random`.

	* `random` will randomly select a healthy upstream host.

	* `round_robin` will select a healthy upstream host in round robin order.

	* `sequential` will select a healthy upstream host in sequential order.

* `health_check` specifies upstream hosts health checking interval. Default is `2s`, minimal is `500ms`.

* `max_fails` is the maximum number of consecutive health checking failures that are needed before considering an upstream to be down. `0` to disable this feature(which the upstream will never be marked as down). Default is `3`.

* `expire` will expire (cached) connections after this time interval. Default is `15s`, minimal is `1s`.

* `force_tcp` uses `TCP` even if the request comes in over UDP.

* `prefer_udp` try first using `UDP` even when the request comes in over `TCP`. If response is truncated(`TC` flag set in response) then do another attempt over `TCP`. If both `force_tcp` and `prefer_udp` are specified the `force_tcp` takes precedence.

	**XXX**: not yet implemented, this feature might be deprecated in future.

* `tls CERT KEY CA` define the TLS properties for TLS connection. From 0 to 3 arguments can be specified:

	* `tls` - No client authentication is used, and the system CAs are used to verify the server certificate.

	* `tls CA` - No client authentication is used, and the CA file is used to verify the server certificate.

	* `tls CERT KEY` - Client authentication is used with the specified CERT/KEY pair. The server certificate is verified with the system CAs.

	* `tls CERT KEY CA` - Client authentication is used with the specified CERT/KEY pair. The server certificate is verified with the given CA file.

	Note that this TLS config is global for redirecting DNS requests.

* `tls_servername` specifies the global TLS server name in the TLS configuration.

	For example, `cloudflare-dns.com` can be used for `1.1.1.1`(Cloudflare), `quad9.net` can be used for `9.9.9.9`(Quad9).

	Note that this is a global name, it doesn't affect the TLS server names specified in `to TO...`.

## Metrics

TODO

## Examples

Redirect all requests to Cloudflare DNS:

```Corefile
dnsredir . {
	to tls://1.1.1.1 tls://1.0.0.1
	tls_servername one.one.one.one
}
```

Redirect all requests to with different upstreams:

```Corefile
dnsredir . {
	# 1.1.1.1 uses the global TLS server name
	# 8.8.8.8 and 9.9.9.9 uses its own TLS server name
	to tls://1.1.1.1 tls://8.8.8.8@dns.google tls://9.9.9.9@quad9.net
	tls_servername cloudflare-dns.com
}
```

Redirect domains listed in file and fallback to Google DNS:

```Corefile
dnsredir accelerated-domains.china.conf {
	reload 3s
	max_fails 0
	to 114.114.114.114 223.5.5.5 119.29.29.29
	policy round_robin
	prefer_udp

	# INLINE domain
	example.org
	example.gov
}

dnsredir google.china.conf apple.china.conf {
	reload 10s
	to tls://223.5.5.5 dns://101.6.6.6 tls://223.6.6.6
	# TLS upstreams use the global TLS server name
	tls_servername alidns.com
	except adservice.google.com doubleclick.net
}

dnsredir . {
	to tls://8.8.8.8@8888.google tls://2001:4860:4860::64@dns.google
	policy sequential
	spray
}
```

**TODO**: add more examples

### See also

[CoreDNS github repository](https://github.com/coredns/coredns)

[forward plugin](https://github.com/coredns/coredns/tree/master/plugin/forward)

[proxy plugin](https://github.com/coredns/proxy)

