---
title: Reverse proxies
---

# Running Snikket behind a reverse proxy

The default Snikket setup assumes that there is no other HTTP/HTTPS server
running. If you already have another web server running for example, you will
need to instruct it to forward Snikket traffic to Snikket.

!!! note

    A quick note about non-HTTP services. Snikket includes a number of non-HTTP
    services which cannot be routed through a HTTP reverse proxy. This includes
    XMPP, STUN and TURN. The documentation here applies to redirecting the HTTP
    and HTTPS ports (80 and 443) through a reverse proxy only.

# Certificates

It is important to get certificates correct when deploying Snikket behind a reverse
proxy. Snikket needs to obtain certificates from Let's Encrypt in order to secure
the non-HTTP services it provides. Be careful that your reverse proxy does not
requests from Let's Encrypt that are intended for the Snikket service.

# Configuration

## Snikket

First we need to tell Snikket to use alternative ports, so that it doesn't conflict
with the primary web server/proxy that will be forwarding the traffic. This can be
done by adding the following lines to /etc/snikket/snikket.conf:

```
SNIKKET_TWEAK_HTTP_PORT=5080
SNIKKET_TWEAK_HTTPS_PORT=5443
```

You can choose any alternative ports that you would prefer, but the rest of this
documentation will assume you use the ports given in this example.

In the next step, you need to configure the web server to forward traffic to
Snikket on these ports. Follow the section below according to which web server
you are using.

## Web servers

Each web server is different, so here we provide some example configuration snippets
for the most common servers. Feel free to contribute any that you would like to see
included!

### Nginx

```
server {
  # Accept HTTP connections
  listen 80;
  listen [::]:80;

  # Accept HTTPS connections
  listen [::]:443 ssl ipv6only=on;
  listen 443 ssl;
  ssl_certificate /path/to/certificate.pem;
  ssl_certificate_key /path/to/key.pem;

  server_name chat.example.com;
  server_name groups.example.com;
  server_name share.example.com;

  location / {
      proxy_pass http://localhost:5080/;
      proxy_set_header  Host            $host;
      proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

### sslh

sslh is a little different to the other servers listed here, as it is not a web server. However it is able
to route encrypted traffic (such as HTTPS and even some kinds of XMPP traffic) to different places.

The snippet below lists the rules required to forward all of Snikket's traffic to Snikket. Don't forget that
Snikket will also need port 80 forwarded to 5080 somehow (otherwise it won't be able to obtain certificates).

Unlike the other solutions here, this approach also allows you to run encrypted XMPP through the HTTPS port.
To take full advantage of this feature, you will need to add additional DNS records. See [advanced DNS](dns.md)
for more information.

This configuration requires sslh 1.18 or higher.

```
listen:
(
    { host: "0.0.0.0"; port: "443"; },
);

protocols:
(
     ## Snikket rules
     # Send encrypted XMPP traffic directly to Snikket (this must be above the HTTPS rules)
     { name: "tls";     host: "127.0.0.1"; port: "5223"; alpn_protocols: [ "xmpp-client" ]; },
     # Send HTTPS traffic to Snikket's HTTPS port
     { name: "tls";     host: "127.0.0.1"; port: "5443"; sni_hostnames:  [ "chat.example.com", "groups.chat.example.com", "share.chat.example.com" ] },
     # Send unencrypted XMPP traffic to Snikket (will use STARTTLS)
     { name: "xmpp";    host: "127.0.0.1"; port: "5222"; },

     ## Other rules
     # Add rules here to forward any other hosts/protocols to non-Snikket destinations
);

```
