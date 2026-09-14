# Vault / OpenBao

Gestión de secretos: motor KV, sellado y arranque de la caja fuerte, y alta disponibilidad.

## Orden de lectura recomendado

| # | Nota | Qué cubre |
|---|---|---|
| 01 | [Motor KV](01-vault-kv.md) | put/get/patch, versionado, rutas `data/` y policies |
| 02 | [OpenBao: barrier, seal e init](02-openbao-seal-e-init.md) | capas de claves, tipos de seal, Shamir, root token |
| 03 | [OpenBao: HA y Raft](03-openbao-ha-y-raft.md) | activo-pasivo, quorum, Kubernetes, snapshots |
