# containerisierung-docker-kubernetes

Training zu Containerisierung mit Docker und Kubernetes, mit Übungen und Exercises.
Entstanden als Merge aus `training-kubernetes-einfuehrung` (Basis) plus den
Docker-spezifischen Inhalten aus `training-kubernetes-docker`.

## Secrets-Handling

- Secrets werden mit SOPS + Age verschlüsselt
- Plain `.env` niemals committen — liegt in `.gitignore`
- Verschlüsselte Secrets: `.env.enc` (mit SOPS)
- Age-Key: `~/.age/key.txt`

### Entschlüsseln auf neuem Rechner

```bash
sops --decrypt --input-type dotenv --output-type dotenv .env.enc > .env
```

## Workshop-Struktur

- Exercises in `exercises/`
- Alle Übungen sind nummeriert und eigenständig
