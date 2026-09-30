# Traefik Dashboard freischalten

Traefik hat ein eingebautes Web-Dashboard, das Router, Services, Middlewares und
Zertifikate live anzeigt. Im offiziellen Helm-Chart (`traefik/traefik`, Version 41.6.0)
ist `api.dashboard: true` bereits der **Standard** - das Dashboard existiert also
intern immer. Es ist nur standardmaessig **nicht erreichbar**, weil keine Route darauf
zeigt.

**Wichtig:** Das Dashboard hat von Haus aus kein Login. Wer draufkommt, sieht die
komplette Routing-Konfiguration des Clusters. Nie ohne Schutz oeffentlich exponieren.

## Hintergrund: welcher Port, welcher Modus

```
kubectl -n ingress get pod -l app.kubernetes.io/name=traefik -o jsonpath='{.items[0].spec.containers[0].ports}'
```

```
[{"containerPort":9100,"name":"metrics"},{"containerPort":8080,"name":"traefik"},{"containerPort":8000,"name":"web"},{"containerPort":8443,"name":"websecure"}]
```

Der interne Port fuer Dashboard/API heisst `traefik` (Container-Port `8080`) - **nicht**
`9000`, wie in aelteren Traefik-Versionen/Bloegen oft zu lesen ist. Dieser Port ist im
Kubernetes-Service standardmaessig nicht exponiert (`kubectl -n ingress get svc traefik`
zeigt nur `web` und `websecure`).

Zwei Modi steuern die Erreichbarkeit:

| Setting | Wirkung |
|---|---|
| `api.insecure: false` (Standard) | Dashboard-Router existiert, ist aber an **keinem** Entrypoint gebunden - Zugriff nur ueber eine explizit angelegte `IngressRoute` |
| `api.insecure: true` | Dashboard wird zusaetzlich **unauthenticated** direkt am `traefik`-Entrypoint (8080) exponiert - nur fuer schnelles lokales Debugging, nie in Produktion |

## Variante 1: Port-Forward direkt auf den Pod (Training/Debugging)

Schnellster Weg, ohne irgendetwas an der Konfiguration zu aendern - **funktioniert nur
mit `api.insecure: true`**, siehe Tabelle oben:

```
helm upgrade traefik traefik/traefik -n ingress --version 41.6.0 --reuse-values --set api.insecure=true

kubectl -n ingress port-forward $(kubectl -n ingress get pod -l app.kubernetes.io/name=traefik -o name) 8080:8080
```

Dashboard dann unter `http://localhost:8080/dashboard/` (abschliessender Slash ist
Pflicht). Getestet: liefert `HTTP 200` und echte Live-Daten (`/api/overview`).

Danach unbedingt wieder deaktivieren:

```
helm upgrade traefik traefik/traefik -n ingress --version 41.6.0 --reuse-values --set api.insecure=false
```

**Achtung:** `helm upgrade` ohne `--reset-values` behaelt frueher per `--set` gesetzte
Werte automatisch bei (kein automatisches Zuruecksetzen auf Chart-Defaults) - deshalb
hier `--set api.insecure=false` explizit gegensetzen, nicht einfach ohne Flags
upgraden.

## Variante 2: IngressRoute mit BasicAuth (dauerhaft, sicher)

Fuer dauerhaften Zugriff ueber eine normale Domain, abgesichert per BasicAuth-Middleware.
Live getestet mit `curl` gegen den Ingress-LoadBalancer:

```
htpasswd -nb admin 'starkesPasswort' > authfile.txt

kubectl -n ingress create secret generic dashboard-basic-auth --from-file=users=authfile.txt
```

```
nano middleware.yml
```

```
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: dashboard-auth
  namespace: ingress
spec:
  basicAuth:
    secret: dashboard-basic-auth
```

```
nano ingressroute.yml
```

```
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: ingress
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`dashboard-<dein-name>.appv2.do.t3isp.de`) && PathPrefix(`/dashboard`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: dashboard-auth
    - match: Host(`dashboard-<dein-name>.appv2.do.t3isp.de`) && PathPrefix(`/api`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
      middlewares:
        - name: dashboard-auth
```

```
kubectl apply -f middleware.yml -f ingressroute.yml
```

Test:

```
curl -i http://dashboard-<dein-name>.appv2.do.t3isp.de/dashboard/
# ohne Auth -> 401

curl -u admin:starkesPasswort -i http://dashboard-<dein-name>.appv2.do.t3isp.de/dashboard/
# mit korrektem Passwort -> 200
```

Erwartetes Verhalten (live verifiziert):

| Request | Ergebnis |
|---|---|
| ohne Auth | `401` |
| falsches Passwort | `401` |
| korrektes Passwort | `200`, `/api/overview` liefert echte Router/Service-Zahlen |

**Hinweis `/api`-Route:** Der Pfad `/dashboard` liefert nur die statische UI. Die UI
laedt ihre Daten von `/api/...` nach - deshalb braucht es **beide** Routen
(`/dashboard` und `/api`), sonst bleibt das Dashboard leer/fehlerhaft.

## Aufraeumen (Testressourcen)

```
kubectl -n ingress delete ingressroute traefik-dashboard
kubectl -n ingress delete middleware dashboard-auth
kubectl -n ingress delete secret dashboard-basic-auth
```

## Zusammenfassung

* `api.dashboard: true` ist im Traefik-Helm-Chart bereits **Standard** - das Dashboard
  muss nicht "aktiviert" werden, sondern nur erreichbar gemacht werden.
* Interner Port ist `8080` (Name `traefik`), nicht `9000`.
* `api.insecure: true` exponiert das Dashboard unauthenticated - nur fuer kurzes,
  lokales Debugging per Port-Forward, danach sofort wieder deaktivieren.
* Fuer dauerhaften/produktiven Zugriff: eigene `IngressRoute` auf den internen Service
  `api@internal`, abgesichert mit einer BasicAuth-`Middleware` - beide Pfade
  `/dashboard` und `/api` freigeben.
* `helm upgrade` ohne `--reset-values` behaelt zuvor per `--set` gesetzte Werte bei -
  zum gezielten Zuruecksetzen den Gegenwert explizit setzen oder `--reset-values`
  verwenden.

## Referenzen

* [Traefik Doku: Dashboard](https://doc.traefik.io/traefik/operations/dashboard/)
* [Traefik Doku: API](https://doc.traefik.io/traefik/operations/api/)
