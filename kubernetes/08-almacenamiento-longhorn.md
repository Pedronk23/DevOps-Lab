# Kubernetes — Almacenamiento: PV, PVC y Longhorn

## PV, PVC y Pod

| Objeto | Qué es |
|---|---|
| **PV** (PersistentVolume) | trozo de almacenamiento real, creado por el admin o automáticamente |
| **PVC** (PersistentVolumeClaim) | petición de un PV por parte de un workload |
| **Pod** | monta el PVC como directorio |

### Analogía

| Objeto | Analogía |
|---|---|
| PV | habitación de hotel (existe físicamente) |
| PVC | la reserva (pides tamaño y tipo) |
| Pod | el huésped (usa la habitación) |

## Longhorn

Es **almacenamiento de bloque distribuido para Kubernetes**. Coge el disco físico de los
nodos y lo reparte en volúmenes independientes, uno por cada aplicación que pide disco.

**Qué hace**: convierte los discos locales de cada nodo en un pool de almacenamiento
replicado. Crea volúmenes que los pods consumen vía PVC como si fuera un disco normal.

**longhorn-manager**: corre en cada nodo y se comunican entre sí vía gossip. El manager del
nodo donde vive el pod que "posee" (*owning*) un volumen es el que activa el engine.

### Cómo funciona un volumen

```
Pod → /dev/longhorn/vol → longhorn-engine (frontend iSCSI / NVMe-oF)
                                │
                      distribuye write a N réplicas
                    ┌───────────┼───────────┐
                    ▼           ▼           ▼
                 nodo-1      nodo-2      nodo-3
```

Cada réplica = directorio en disco local con archivos de datos + head. El engine sincroniza
las escrituras a todas las réplicas antes de confirmar al pod (consistencia fuerte).

### Conceptos importantes

| Concepto | Qué es |
|---|---|
| **Replica count** | cuántas copias. 3 = tolera 2 nodos caídos |
| **Volume attachment** | el volumen es ReadWriteOnce por defecto (un pod a la vez) |
| **Snapshot** | copia point-in-time de la réplica. Incremental, basada en diff de bloques |
| **Backup** | snapshot exportado a S3/NFS externo. Permite DR real |
| **Recurring jobs** | snapshots y backups programados (estilo CronJob) |
| **Node selector / disk tags** | controlas en qué nodos o discos van las réplicas. Útil si solo tienes SSD en algunos nodos |

## Flujo completo: PVC → pod

1. PVC creada con `storageClass: longhorn`.
2. `longhorn-manager` lo detecta → crea el Volume CR.
3. Schedula N réplicas en distintos nodos.
4. Activa `longhorn-engine` en el nodo del pod.
5. El engine expone el dispositivo de bloque al kubelet.
6. El kubelet lo monta en el pod.

> Si un nodo cae → el engine detecta la réplica muerta → **rebuild automático** en otro nodo
> si hay espacio.

## Flujo con Longhorn (dynamic provisioning)

1. **PVC creada** → con StorageClass `longhorn`, Longhorn crea el PV automáticamente.
2. **PVC bound** al PV.
3. **Pod arranca** → Longhorn monta el volumen en el nodo → el pod ve `/data`.

> Sin StorageClass tendrías que crear el PV a mano. Con Longhorn es automático.
