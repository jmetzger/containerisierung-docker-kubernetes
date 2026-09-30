# Traefik-Helm-Release haengt in "pending-upgrade"

Frage aus dem Training: `helm list` zeigt den Traefik-Release dauerhaft als
`pending-upgrade` an, obwohl der Pod laeuft. Was ist da los und wie wird man das los?

**Live verifiziert** gegen einen echten DOKS-Cluster (Traefik-Chart 41.6.0).

## Symptom

```
helm list -A --all
```

```
NAME     NAMESPACE  REVISION  STATUS           CHART           APP VERSION
traefik  ingress    6         pending-upgrade  traefik-41.6.0  v3.7.13
```

Jeder weitere `helm upgrade`/`helm rollback` schlaegt fehl mit:

```
Error: UPGRADE FAILED: another operation (install/upgrade/rollback) is in progress
```

## Ursache

Ein `helm upgrade`-Lauf wurde mittendrin unterbrochen (Terminal-/SSH-Verbindung weg,
Ctrl+C, CI-Job-Timeout/-Kill), **bevor** Helm den Vorgang als `Upgrade complete`
(Erfolg) abschliessen konnte. Die Revision bleibt dauerhaft bei `Preparing upgrade`
haengen - Helm 3 hat dafuer keinen automatischen Timeout.

```
helm history traefik -n ingress
```

```
REVISION  STATUS           DESCRIPTION
5         superseded       Upgrade complete
6         pending-upgrade  Preparing upgrade
```

**Wichtig:** Das betrifft nur die Helm-Metadaten. Der tatsaechliche
Kubernetes-Rollout kann davon voellig unberuehrt sein - erst pruefen, bevor man von
einem echten Incident ausgeht:

```
kubectl -n ingress get deploy,pods
kubectl -n ingress rollout status deploy/traefik
```

Im Testfall war der Pod die ganze Zeit `1/1 Running`, Rollout `successfully rolled
out` - reines Metadaten-Problem.

## Falscher Fix (kursiert online, wirkt aber nicht)

Oft empfohlen, aber wirkungslos: nur das Label des Release-Secrets patchen.

```
kubectl patch secret sh.helm.release.v1.traefik.v6 -n ingress -p '{"metadata": {"labels": {"status": "failed"}}}'
```

Das Label ist nur Metadaten fuer `kubectl`-Selektoren. Der Status, den Helm fuer
Locking/Pending-Checks tatsaechlich auswertet, steckt gzip+base64-codiert als
Protobuf im Feld `data.release` desselben Secrets. Ein Label-Patch aendert daran
nichts - getestet, `helm upgrade` schlug danach weiterhin mit dem Lock-Fehler fehl.

## Tatsaechlicher Fix: haengendes Release-Secret loeschen

Voraussetzung: Pod/Deployment laeuft bereits sauber (siehe oben geprueft). Dann reicht
es, das Secret der haengenden Revision zu loeschen. Helm faellt danach automatisch auf
die letzte tatsaechlich `deployed` Revision zurueck:

```
kubectl -n ingress get secrets -l "owner=helm,name=traefik"

kubectl -n ingress delete secret sh.helm.release.v1.traefik.v6

helm upgrade traefik traefik/traefik -n ingress --version 41.6.0
```

Ergebnis im Test:

```
Release "traefik" has been upgraded. Happy Helming!
STATUS: deployed
REVISION: 6
```

```
helm list -n ingress
```

```
NAME     NAMESPACE  REVISION  STATUS    CHART           APP VERSION
traefik  ingress    6         deployed  traefik-41.6.0  v3.7.13
```

**Hinweis:** `helm list` (ohne `--all`) zeigt direkt nach dem Loeschen der haengenden
Revision und vor dem erneuten Upgrade kurzzeitig **gar nichts** an - weil auch die
vorherige Revision intern bereits als `superseded` markiert war und keine Revision
mehr den Status `deployed` traegt. Das ist rein kosmetisch und loest sich mit dem
naechsten erfolgreichen Upgrade von selbst.

## Alternative: Rollback statt Secret loeschen

```
helm rollback traefik 5 -n ingress
```

Funktioniert oft **nicht**, solange der `pending-upgrade`-Lock noch aktiv ist - Helm
blockiert auch Rollbacks damit. In dem Fall bleibt nur das Loeschen des
Release-Secrets.

## Zusammenfassung

* `pending-upgrade` = abgebrochener `helm upgrade`, kein automatischer Timeout in
  Helm 3.
* Erst den echten Cluster-Zustand pruefen (`kubectl get deploy,pods`) - oft ist nur
  Helms Metadaten-Zustand betroffen, keine echte Stoerung.
* Label-Patch auf dem Release-Secret **wirkt nicht** - der massgebliche Status steckt
  im codierten `data.release`-Feld.
* Wirksamer Fix: Secret der haengenden Revision loeschen (`sh.helm.release.v1.<release>.v<rev>`),
  danach normal upgraden.
* `helm rollback` kann am selben Lock scheitern wie `helm upgrade`.

## Referenzen

* [Helm Doku: Release-Status](https://helm.sh/docs/intro/using_helm/#helpful-options-for-install-upgrade-rollback)
