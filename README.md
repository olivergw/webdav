# WebDAV

WebDAV data is stored in `/opt/data/webdav`. Apache provides the WebDAV service;
`webdav-nginx` is a small Nginx proxy with the same query and user-agent filters
previously enabled in Nginx Proxy Manager. The filters are pattern-based, can
block legitimate requests and are not a complete firewall. TLS and routing are
handled by Traefik; no host ports are published by this stack.

## Staged deployment

The first stage deliberately keeps Apache attached to `traefik_proxy` and
Traefik pointed directly to Apache. The additional private `backend` network
lets Nginx reach Apache, so the front end can be checked before changing the
live route. Recreating WebDAV causes a brief interruption; its bind-mounted
data is unchanged. Credentials remain in the ignored host `.env` file.

With the Traefik network already present, validate and recreate Apache:

```sh
docker compose config -q
docker compose up -d --no-deps webdav
docker compose run --rm --no-deps nginx nginx -t
docker compose up -d --no-deps nginx
docker compose ps
```

Test the front end privately before switching Traefik: an unauthenticated
request must return 401, an exploit-filter test query should return 403, and
authenticated PROPFIND, download and a disposable upload/delete should work.
Also verify Infuse playback, Range requests and large file transfers. Do not
place passwords on command lines or in logs. Nginx disables request buffering
and uses a one-hour upstream send/read timeout to support long transfers; test
against the actual workload. Traefik owns the public HSTS policy.

Only after those checks should `vps-traefik` route `media.nwcn.co.uk` to
`http://webdav-nginx:80`. Verify the same operations through the public HTTPS
hostname. Until that switch, remove or stop the new Nginx service to roll back;
Apache remains reachable through its existing direct Traefik route. After the
switch, restoring the previous Traefik route to `http://webdav:80` rolls back
without modifying `/opt/data/webdav`. Do not remove Apache's `traefik_proxy`
attachment until after the route change has been verified and the rollback plan
has been updated.
