# WebDAV

WebDAV data is stored in `/opt/data/webdav`. Apache provides the WebDAV service;
`webdav-nginx` is a small Nginx proxy with the same query and user-agent filters
previously enabled in Nginx Proxy Manager. The filters are pattern-based, can
block legitimate requests and are not a complete firewall. TLS and routing are
handled by Traefik; no host ports are published by this stack.

## Deployment

Traefik reaches `webdav-nginx` over `traefik_proxy`. Only Nginx and Apache join
the internal `backend` network; Apache has no direct proxy-network attachment.
Ensure the shared Traefik network exists before starting this stack. Keep
credentials in the ignored host `.env` file.

```sh
docker compose config -q
docker compose up -d --no-deps webdav
docker compose run --rm --no-deps nginx nginx -t
docker compose up -d --no-deps --force-recreate nginx
docker compose ps
```

Nginx resolves the `webdav` upstream at startup. Recreate or reload Nginx after
recreating Apache, so it resolves the current container address. Check the
public authentication challenge, authenticated PROPFIND, Infuse playback and
Range requests. Nginx disables request buffering and uses a one-hour upstream
send/read timeout for long transfers; test large transfers with your workload.
The HTTPS route and HSTS are owned by Traefik.

During migration, an upload with the `infuse` account returned 403 through both
the direct Apache route and Nginx. The cause and intended write permissions have
not been established. The filters are pattern-based, can block legitimate
requests and are not a complete firewall.

## Rollback

For a quick temporary rollback while both containers are running, attach Apache
to the shared network with `docker network connect traefik_proxy webdav`, then
change the Traefik route back to `http://webdav:80`. A future Compose recreation
will remove that manual attachment. For a persistent rollback, restore the
previous WebDAV Compose version and the previous Traefik route, then recreate
and verify the services. Neither procedure requires changing `/opt/data/webdav`.
