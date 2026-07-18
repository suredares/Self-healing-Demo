# self-healing-demo

A minimal self-healing pipeline: **Prometheus → Alertmanager → webhook → automatic restart**.

When nginx dies, Prometheus notices, Alertmanager fires a webhook, and a ~50-line
Python responder restarts the container. No paid tools, no operators, runs on a laptop.

Full write-up: [My nginx died on a Saturday, so I taught it to fix itself]
https://dev.to/surebabunarayanan/my-nginx-died-on-a-saturday-so-i-taught-it-to-fix-itself-4bo1

## Stack

| Component | Job |
|---|---|
| nginx | the service we protect |
| nginx-prometheus-exporter | turns nginx status into metrics |
| Prometheus | scrapes every 5s, evaluates the `NginxDown` rule |
| Alertmanager | routes the firing alert to a webhook |
| responder (Flask) | receives the webhook and restarts nginx, with a circuit breaker |

## Run it

```bash
docker compose up -d --build
```

Check everything is up:

- nginx: http://localhost:8080
- Prometheus targets: http://localhost:9090/targets (both targets UP)
- Alertmanager: http://localhost:9093

## Break it

```bash
docker stop nginx
```

Then watch:

1. http://localhost:9090/alerts - `NginxDown` goes *pending*, then *firing* (~30–40s)
2. `docker logs -f responder` - you'll see the webhook arrive and the restart happen
3. `docker ps` - nginx is back, uptime a few seconds

Whole loop takes roughly 45–60 seconds with the settings in this repo.

## Safety notes

This is a demo. Before doing anything like this in production you'd want, at minimum:
rate limiting (a basic version is in `responder/app.py`), deduplication, escalation to
a human after N failed restarts, and an audit trail. Blind auto-restart against a
service that's crashing for a real reason (bad config, full disk) just melts the CPU
politely. The circuit breaker in the responder stops after 3 restarts in 10 minutes.

## License

MIT - use it, break it, learn from it.
