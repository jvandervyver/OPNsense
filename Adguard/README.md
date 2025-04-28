# What is this?

**NOTE:** Throughout `opnsense.mydomain.com` means the FQDN for your OPNsense install. ie. you might have `opsense.subdomain.mydomain.com` or etc.

A guide to show how to setup Adguard on OPNsense with the following features:
- DNS (port 53) `opnsense.mydomain.com port 53`
- DNS over TLS (port 853) `opnsense.mydomain.com port 853`
- DNS over HTTPS (port 443), `https://opnsense.mydomain.com/dns-query`
- UI accessible on `http(s)://opnsense.mydomain.com/adguard`
- unbound as upstream

## Install Adguard

Install Adguard as recommended in [How to setup AdGuard Home DNS on OPNsense with Unbound](https://windgate.net/setup-adguard-home-opnsense-adblocker/)

**Recommended alternations:**
- Use 127.0.0.1:<port> as your upstream for unbound, using the explicit firewall IP when it is running on the same host is questionable at best
- Keep unbound as a recursive resolver.  DNS leak test will show your IP, but trusting Cloudflare DNS might be false economy.
- Consider using another port for unbound, ie `51162` (randomly select ephemeral port), 5353 is a bad choice

## Post guide Adguard configuration

In the Adguard UI, navigate to `Encryption settings`
- Check **Enable Encryption**
- Configure **Server Name** to the name of your OPNsense install, ie. `opnsense.mydomain.com`
- Uncheck **Redirect to HTTPS automatically**
- Leave **ALL** ports as is **EXCEPT** HTTPS port, clear that field
  - **Yes clear it** we are providing HTTPS via OPNsense reverse proxy
- **Certificates**
  - "Set a certificate file path" = `/usr/local/etc/lighttpd_webgui/cert.pem`
  - "Set a private key file" = `/usr/local/etc/lighttpd_webgui/key.pem`

## Post guide changes

1. `ssh` to your OPNsense install and become root
2. `curl --output '/usr/local/etc/lighttpd_webgui/conf.d/adguard_home.conf' 'https://raw.githubusercontent.com/jvandervyver/OPNsense/refs/heads/main/Adguard/adguard_home.conf'`
3. Restart OPNsense webGUI `/usr/local/etc/rc.restart_webgui`
4. Stop Adguard home `/usr/local/etc/rc.d/adguardhome stop`
5. Edit `/usr/local/AdGuardHome/AdGuardHome.yaml` with your favorite editor, ie. `vi /usr/local/AdGuardHome/AdGuardHome.yaml`
6. Update the following sections

Set the address for http to `127.0.0.1` to limit access to localhost only for HTTP endpoint

```
http:
  pprof:
    port: 6060
    enabled: false
  address: 127.0.0.1:3000
```

Ensure trusted proxies has, or at least `127.0.0.1/32`
```
  trusted_proxies:
    - 127.0.0.0/8
```

Without this, you won't see the real-ip of resolve requests

Ensure bind_hosts for DNS allows remote connections, we want DNS queries on port 53, 853, etc. to be fielded by Adguard directly.

```
dns:
  bind_hosts:
    - 0.0.0.0
```

**tls** section

```
tls:
  enabled: true
  server_name: opnsense.mydomain.com
  force_https: false
  port_https: 0
  allow_unencrypted_doh: true
  certificate_path: /usr/local/etc/lighttpd_webgui/cert.pem
  private_key_path: /usr/local/etc/lighttpd_webgui/key.pem
```

Notice `allow_unencrypted_doh: true`, this is copy/paste from the Adguard FAQ, we are going to reverse proxy this.
Also notice `force_https: false` and `port_https: 0`.  lighttpd, OPNsense's reverse proxy doesn't support HTTPS backend, and since we are binding to 127.0.0.1 it is completely pointless and wasteful anyway.

7. Restart Adguard home `/usr/local/etc/rc.d/adguardhome start`
8. On any computer on the network test https://opnsense.mydomain.com/adguard
9. On any computer on the network test DNS over HTTPS: `curl -v example.com --doh-url https://opnsense.mydomain.com/dns-query`
