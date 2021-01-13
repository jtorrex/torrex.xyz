---
title: "OpenmediaVault: OpenVPN Plugin fails to connect by outdated CRL"
date: "2019-09-30"
tags:
- openmediavault
- openvpn
- crl
- rsa
categories:
- blog
- selfhosted
author: jtorrex
---

# Description:

After upgrade OpenmediaVault, the OpenVPN plugin fails and doesn't allow incoming connections.

# Problems after upgrade

As an habitual practice, I try to keep all my systems updated regularly. But as well knowed, sometimes upgrades breaks things, and after break thinks you only have one option, research and repair.

After upgrade my OpenmediaVault instance on a dedicated server, I noticed that my laptop OpenVPN client doesn't connect to the destination.

Both installed versions are:

* OMV: 4.1.26-1 Arrakis
* OpenVPN: 2.4.0

## Diggin on the logs

On the logs /var/log/openvpn.log I saw entries like:

```
VERIFY ERROR: depth=0, error=CRL has expired: CN=******
OpenSSL: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
TLS_ERROR: BIO read tls_read_plaintext error
TLS Error: TLS object -> incoming plaintext read error
TLS Error: TLS handshake failed
SIGUSR1[soft,tls-error] received, client-instance restarting
```

The first line reveals that the CRL (Certificatioin Revocation List) has expired.

To check it manually, we can run:

```
root@openmediavault:# openssl crl -in /etc/openvpn/pki/crl.pem -text
```

And it reveals the obvious problem:

```
Certificate Revocation List (CRL):
        Version 2 (0x1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: /CN=ChangeMe
        Last Update: Apr 18 21:47:18 2019 GMT
        Next Update: Sep 20 18:05:57 2019 GMT
```

## Looking for renew

After search about how to renew the CRL certificate, the solution it's pretty easy and transparent. We only need to renew the cert using EasyRsa (the location of the EasyRsa will vary depending of the plarform). In my case, it's located under /opt.

First, we need to move to the OpenVPN directory:

```
root@openmediavault:# cd /etc/openvpn/
```

When we are inside, we could run:

```
root@openmediavault:/etc/openvpn# /opt/EasyRSA-3.0.3/easyrsa gen-crl
Using configuration from /opt/EasyRSA-3.0.3/openssl-1.0.cnf

An updated CRL has been created.
CRL file: /etc/openvpn/pki/crl.pem
```

After this step, the crl.pem was renewed, and we can check the next caducity date as checked before:

```
root@openmediavault:# openssl crl -in /etc/openvpn/pki/crl.pem -text
```

Now, the result reveals:

```
        Next Update: Mar 28 21:47:18 2020 GMT
```

Afther renewing the CRL, we only need to restart the OpenVPN service:

```
root@openmediavault:# systemctl restart openvpn
```

## Tunneling again

After restart the service, we should check if it's working properly and we should try to connect again to the OpenVPN server:

Checking the service...

```
root@openmediavault:/etc/openvpn# systemctl status openvpn
● openvpn.service - OpenVPN service
   Loaded: loaded (/lib/systemd/system/openvpn.service; enabled; vendor preset: enabled)
   Active: active (exited) since Mon 2019-09-30 23:50:28 CEST; 4min 43s ago
  Process: 44080 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
 Main PID: 44080 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 9830)
   Memory: 0B
      CPU: 0
   CGroup: /system.slice/openvpn.service

Sep 30 23:50:28 openmediavault systemd[1]: Starting OpenVPN service...
Sep 30 23:50:28 openmediavault systemd[1]: Started OpenVPN service.
```

When I try to connect again, the connection it's succesfull and I can use again my personal VPN against my dedicated server.

On the server logs now we can see the chain like:

```
Mon Sep 30 23:58:46 2019 VERIFY OK: depth=1, CN=ChangeMe
Mon Sep 30 23:58:46 2019 VERIFY OK: depth=0, CN=********
Mon Sep 30 23:58:46 2019 peer info: IV_VER=2.4.7
Mon Sep 30 23:58:46 2019 peer info: IV_PLAT=linux
Mon Sep 30 23:58:46 2019 peer info: IV_PROTO=2
Mon Sep 30 23:58:46 2019 peer info: IV_NCP=2
Mon Sep 30 23:58:46 2019 peer info: IV_LZ4=1
Mon Sep 30 23:58:46 2019 peer info: IV_LZ4v2=1
Mon Sep 30 23:58:46 2019 peer info: IV_LZO=1
Mon Sep 30 23:58:46 2019 peer info: IV_COMP_STUB=1
Mon Sep 30 23:58:46 2019 peer info: IV_COMP_STUBv2=1
Mon Sep 30 23:58:46 2019 peer info: IV_TCPNL=1
Mon Sep 30 23:58:48 2019 Control Channel: TLSv1.2, cipher TLSv1/SSLv3 ECDHE-RSA-AES256-GCM-SHA384, 2048 bit RSA
Mon Sep 30 23:58:48 2019 [*******] Peer Connection Initiated with [AF_INET] **************
Mon Sep 30 23:58:48 2019 [*******] MULTI_sva: pool returned IPv4=10.8.0.2, IPv6=(Not enabled)
Mon Sep 30 23:58:52 2019 [*******] Data Channel Encrypt: Cipher 'AES-256-GCM' initialized with 256 bit key
Mon Sep 30 23:58:52 2019 [*******] Data Channel Decrypt: Cipher 'AES-256-GCM' initialized with 256 bit key
```

After this point, we can consider that the OpenVPN service was restablished and the CRL renewed for a few months more!

Enjoy free VPN!
