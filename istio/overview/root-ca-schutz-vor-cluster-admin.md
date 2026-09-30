# FAQ: Root-CA-Schutz bei istiod einrichten

## Kann ich das bei istiod einrichten?

Ja, aber nicht mit der Default-Konfiguration – istiod hat von Haus aus eine eingebaute Self-Signed-CA (Citadel), die Root **und** Intermediate selbst hält, als Kubernetes-Secret `istio-ca-secret` im Namespace `istio-system`. Genau das gilt es zu vermeiden. Drei Reifegrade:

### Stufe 1: "Plugged-in" Intermediate-CA (einfachste Verbesserung)

Istio unterstützt nativ, dass istiod nur eine selbst erzeugte **Intermediate-CA** bekommt (Root bleibt extern):

```bash
kubectl create secret generic cacerts -n istio-system \
  --from-file=ca-cert.pem \
  --from-file=ca-key.pem \
  --from-file=root-cert.pem \
  --from-file=cert-chain.pem
```

- ✅ Root-Key liegt nie im Cluster.
- ❌ Der Intermediate-Key liegt weiterhin als normales K8s-Secret im Cluster – ein Node-Root-Admin kommt weiterhin dran. Begrenzt nur den Schaden (Intermediate rotierbar, Root bleibt sauber).

### Stufe 2: `cert-manager` + `istio-csr` (Standard-Empfehlung für Production)

istiod hält gar keinen privaten Schlüssel mehr selbst, sondern reicht CSRs an [`cert-manager-istio-csr`](https://cert-manager.io/docs/usage/istio-csr/) weiter. `cert-manager` ist an einen beliebigen **Issuer** angebunden:

- Vault-Issuer (Vault mit Auto-Unseal via Cloud-KMS)
- AWS Private CA / Google Certificate Authority Service (Cloud-HSM-backed)
- Venafi oder andere Enterprise-PKI

Der Cluster-Admin sieht dann nie einen privaten CA-Key als Kubernetes-Secret – der Signiervorgang passiert komplett extern, nur das fertige, kurzlebige Zertifikat kommt zurück in den Pod.

### Stufe 3: SPIRE statt istiod-eigener CA

Istio kann so konfiguriert werden, dass es SPIRE als Identity-Provider nutzt statt der eingebauten Citadel-CA. SPIRE Server unterstützt **UpstreamAuthority-Plugins** direkt für HSM (PKCS#11), AWS KMS, GCP KMS etc. – der Root-Key verlässt nie das HSM, egal wer Node-Root auf dem SPIRE-Server hat.

### Übersicht

| Stufe | Root-Schutz | Aufwand |
|---|---|---|
| Default istiod | ❌ keiner | 0 |
| Plugged-in Intermediate | ✅ Root, ❌ Intermediate | gering |
| `cert-manager` + `istio-csr` + Vault/Cloud-CA | ✅ vollständig | mittel |
| SPIRE + HSM-UpstreamAuthority | ✅ vollständig, maximal auditierbar | höher |

**Empfehlung für die meisten produktiven Setups:** `cert-manager` + `istio-csr` + Vault (oder Cloud-CA) als Sweet Spot zwischen Aufwand und Sicherheit.
