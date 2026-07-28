# Learning Notes — GitLab + Traefik + Cloudflare Tunnel (2026-07-28)

## Summary
Today we debugged `502 Bad Gateway` and `404` behavior for `gitlab.vr-worlds.eu` behind:

- Cloudflare Tunnel (`cloudflared`)
- Traefik (reverse proxy)
- GitLab CE container (Omnibus)

Final root cause was Traefik forwarding to the wrong internal GitLab port.

---

## Final Working Outcome

GitLab works when Traefik forwards to:

- `traefik.http.services.gitlab.loadbalancer.server.port=80`

Instead of:

- `...server.port=8080` ❌

---

## Key Lessons Learned

## 1) “Container healthy” does not always mean “app fully reachable”
- `docker ps` showed healthy, but app endpoints still needed verification.
- Always test real HTTP behavior, not only health status.

Useful checks:
```bash
docker exec -it gitlab curl -I http://localhost/
docker exec -it gitlab curl -I http://localhost/users/sign_in
```

---

## 2) Validate every hop in the chain
Request path is:

**Client → Cloudflare Edge → cloudflared → Traefik → GitLab**

Test each layer independently:
- GitLab local response
- Traefik router/service status
- Public Cloudflare response

---

## 3) Traefik dashboard/API is critical for truth
These commands quickly showed routing state:

```bash
curl -s http://127.0.0.1:8080/api/http/routers | jq
curl -s http://127.0.0.1:8080/api/http/services | jq
curl -s http://127.0.0.1:8080/api/http/middlewares | jq
```

---

## 4) Wrong upstream port can look like app/proxy/header issues
We investigated headers, redirects, and GitLab config deeply, but final blocker was upstream port mismatch.

For this GitLab deployment, Traefik must target port **80**.

---

## 5) Cloudflared logs can contain historical noise
`docker logs cloudflared` included old failures and config versions.
Always focus on **latest logs after making a new request**.

---

## 6) Clean resets help remove stale GitLab state
When behavior looked inconsistent (e.g. old redirects), resetting persisted GitLab dirs helped ensure fresh config application.

---

## Configuration Notes (Working Pattern)

### Traefik labels (important parts)
```yaml
- traefik.enable=true
- traefik.docker.network=proxy
- traefik.http.routers.gitlab.rule=Host(`gitlab.vr-worlds.eu`)
- traefik.http.routers.gitlab.entrypoints=web
- traefik.http.routers.gitlab.middlewares=gitlab-https-headers@docker
- traefik.http.services.gitlab.loadbalancer.server.port=80
- traefik.http.middlewares.gitlab-https-headers.headers.customrequestheaders.X-Forwarded-Proto=https
- traefik.http.middlewares.gitlab-https-headers.headers.customrequestheaders.X-Forwarded-Ssl=on
```

### GitLab Omnibus (relevant)
- `external_url` should match public HTTPS URL
- GitLab behind reverse proxy should receive forwarded HTTPS headers

---

## Quick Troubleshooting Checklist (Future)

1. Check GitLab processes:
```bash
docker exec -it gitlab gitlab-ctl status
```

2. Check direct GitLab local HTTP:
```bash
docker exec -it gitlab curl -I http://localhost/
```

3. Check Traefik router + service:
```bash
curl -s http://127.0.0.1:8080/api/http/routers | jq '.[] | select(.name=="gitlab@docker")'
curl -s http://127.0.0.1:8080/api/http/services | jq '.[] | select(.name=="gitlab@docker")'
```

4. Check host-header routing through Traefik:
```bash
curl -I -H "Host: gitlab.vr-worlds.eu" http://127.0.0.1:80
```

5. Check public response:
```bash
curl -I https://gitlab.vr-worlds.eu
```

6. If Cloudflare unstable, inspect recent cloudflared logs:
```bash
docker logs --tail 100 cloudflared
```

---

## Final Takeaway
The decisive fix was simple but easy to miss:

> Correct Traefik upstream port for GitLab was **80**, not 8080.

Everything else depended on that being right first.