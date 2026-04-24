# ATELIER PRA / PCA — Kubernetes, Flask, SQLite

> **Auteure :** BACHIRI Imane  
> **Établissement :** EPSI  
> **Date :** Avril 2026

---

## 🧠 L'idée en 30 secondes

Cet atelier met en œuvre un mini-PRA sur Kubernetes en déployant une application Flask avec une base SQLite stockée sur un volume persistant (PVC `pra-data`) et des sauvegardes automatiques réalisées chaque minute vers un second volume (PVC `pra-backup`) via un CronJob. L'image applicative est construite avec **Packer** et le déploiement orchestré avec **Ansible**.

---

## 📐 Architecture cible

```
┌─────────────────────────────────────────────────┐
│                Cluster K3d (pra)                │
│                                                 │
│  ┌──────────────┐      ┌──────────────────────┐ │
│  │  Pod Flask   │─────▶│  PVC pra-data        │ │
│  │  (Gunicorn)  │      │  /data/app.db        │ │
│  └──────────────┘      └──────────────────────┘ │
│         │                        │              │
│         │              CronJob (1/min)           │
│         │                        ▼              │
│         │              ┌──────────────────────┐ │
│         └─────────────▶│  PVC pra-backup      │ │
│                        │  /backup/app-*.db    │ │
│                        └──────────────────────┘ │
└─────────────────────────────────────────────────┘
```

---

## Séquence 5 : Exercices

### Exercice 1 — Composants dont la perte entraîne une perte de données

La perte du **pod seul** n'entraîne aucune perte de données : Kubernetes recrée automatiquement le pod, et les données restent dans le PVC `pra-data`.

La perte du **PVC `pra-data` seul** entraîne une perte partielle : on peut restaurer depuis `pra-backup`, mais on perd les données entre le dernier backup et le moment du sinistre (= RPO).

La perte du **PVC `pra-backup` seul** supprime toutes les sauvegardes. Si `pra-data` est aussi perdu par la suite, aucune restauration n'est possible.

La perte **simultanée des deux PVC** entraîne une perte totale et définitive : aucune donnée ne peut être récupérée.

👉 **Conclusion :** Le seul composant dont la perte rend impossible toute restauration est `pra-backup` (si `pra-data` est aussi perdu).

---

### Exercice 2 — Pourquoi les données n'ont pas été perdues lors de la suppression du PVC pra-data ?

Lors du scénario 2, nous avons supprimé le PVC `pra-data` (la base de données de production). Les données n'ont pas été perdues définitivement car :

1. **Le CronJob effectuait une sauvegarde toutes les minutes** depuis `pra-data` vers `pra-backup`.
2. **Le PVC `pra-backup` n'a pas été touché** lors du sinistre : il contenait donc une copie récente de la BDD.
3. **La procédure de restauration** (`50-job-restore.yaml`) a copié le dernier backup de `pra-backup` vers le nouveau `pra-data` vide.

👉 C'est exactement le principe d'un PRA : **séparer physiquement les données de production des sauvegardes**.

---

### Exercice 3 — RTO et RPO

| Indicateur | Définition | Valeur dans cet atelier |
|------------|-----------|------------------------|
| **RPO** (Recovery Point Objective) | Perte de données maximale acceptable | ~1 minute — le CronJob tourne toutes les minutes, donc on perd au maximum les données de la dernière minute avant le sinistre |
| **RTO** (Recovery Time Objective) | Durée maximale acceptable pour restaurer le service | ~5 à 10 minutes — temps pour : scaler à 0, recréer le PVC, relancer le pod, appliquer le job de restauration, relancer le port-forward |

---

### Exercice 4 — Pourquoi cette solution ne peut pas être utilisée en production ?

**1. Stockage local non répliqué**
Les PVC `pra-data` et `pra-backup` sont sur le même cluster K3d, sur la même machine physique. Si le nœud tombe, les deux volumes sont perdus simultanément. Un vrai PRA exige une réplication géographique (autre datacenter, autre région cloud).

**2. Pas de chiffrement des sauvegardes**
Les backups sont stockés en clair sur le volume. En production, les données sensibles doivent être chiffrées au repos (AES-256) et en transit (TLS).

**3. Pas de supervision / alerting**
Aucun mécanisme ne détecte si le CronJob échoue silencieusement. En production, il faut monitorer les sauvegardes et alerter en cas d'échec (Prometheus + Alertmanager).

**4. SQLite non adapté à la production**
SQLite est une base de données fichier mono-utilisateur, non adaptée à une charge concurrente. En production on utilise PostgreSQL, MySQL, etc. avec leurs propres mécanismes de sauvegarde.

**5. Pas de test de restauration automatisé**
Un PRA non testé régulièrement est un PRA qui ne fonctionne pas. Ici, personne ne vérifie que le backup est réellement restaurable.

**6. RTO et RPO trop élevés pour certains secteurs**
1 minute de RPO et 10 minutes de RTO peuvent être inacceptables dans les secteurs bancaire, santé, ou e-commerce.

---

### Exercice 5 — Architecture plus robuste

```
┌─────────────────────────────────────────────────────────────┐
│                    ZONE A (Datacenter 1)                    │
│  ┌─────────────┐    ┌──────────────────────────────────┐   │
│  │ Kubernetes  │    │  BDD managée (ex: RDS Multi-AZ)  │   │
│  │  (pods app) │───▶│  avec réplication synchrone      │   │
│  └─────────────┘    └──────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │ réplication géographique
┌─────────────────────────────▼───────────────────────────────┐
│                    ZONE B (Datacenter 2)                    │
│         Réplica BDD (standby)                               │
│         Backups chiffrés (S3 / GCS / Azure Blob)            │
└─────────────────────────────────────────────────────────────┘
```

**Améliorations clés :**
- BDD managée avec réplication (ex: AWS RDS Multi-AZ) → failover automatique en < 60 secondes
- Sauvegardes vers stockage objet externe (S3, GCS) → survivent à la perte totale du cluster
- Chiffrement des backups au repos (AES-256) et en transit (TLS)
- Monitoring : alertes si le backup échoue (Prometheus + Alertmanager)
- Tests de restauration automatisés hebdomadaires
- RPO < 5 secondes grâce à la réplication synchrone
- RTO < 2 minutes grâce au failover automatique

---

## Séquence 6 : Ateliers

### Atelier 1 — Route /status

Ajout d'une route `GET /status` dans `app/app.py` qui retourne en JSON :
- `count` : nombre d'événements en base
- `last_backup_file` : nom du dernier fichier de backup dans `/backup`
- `backup_age_seconds` : âge du dernier backup en secondes

**Code ajouté :**
```python
@app.get("/status")
def status():
    init_db()
    conn = get_conn()
    cur = conn.execute("SELECT COUNT(*) FROM events")
    n = cur.fetchone()[0]
    conn.close()
    backup_dir = "/backup"
    last_backup_file = None
    backup_age_seconds = None
    if os.path.isdir(backup_dir):
        files = sorted(os.listdir(backup_dir))
        if files:
            last_backup_file = files[-1]
            full_path = os.path.join(backup_dir, last_backup_file)
            age = datetime.utcnow().timestamp() - os.path.getmtime(full_path)
            backup_age_seconds = int(age)
    return jsonify(
        count=n,
        last_backup_file=last_backup_file,
        backup_age_seconds=backup_age_seconds
    )
```

**Capture d'écran de la route /status fonctionnelle :**

> 📸 *[INSÉRER ICI LE SCREENSHOT DE /status AVEC last_backup_file ET backup_age_seconds REMPLIS]*

---

### Atelier 2 — Choisir son point de restauration

#### Runbook — Restauration vers un point spécifique

**Objectif :** Restaurer la base de données SQLite depuis un backup précis plutôt que systématiquement le dernier.

**Fichier créé :** `pra/51-job-restore-point.yaml`

**Étape 1 — Lister les points de restauration disponibles**
```bash
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='{"spec":{"containers":[{"name":"debug","image":"alpine","command":["sh"],"stdin":true,"tty":true,"volumeMounts":[{"name":"backup","mountPath":"/backup"}]}],"volumes":[{"name":"backup","persistentVolumeClaim":{"claimName":"pra-backup"}}]}}'
# Dans le pod :
ls -lh /backup
exit
```

**Étape 2 — Scaler le pod Flask à 0**
```bash
kubectl -n pra scale deployment flask --replicas=0
```

**Étape 3 — Choisir le fichier de backup et modifier le job**
```bash
# Remplacer app-XXXXXXXXXX.db par le fichier choisi
sed -i 's/value: ""/value: "app-XXXXXXXXXX.db"/' pra/51-job-restore-point.yaml
```

**Étape 4 — Appliquer le job de restauration**
```bash
kubectl apply -f pra/51-job-restore-point.yaml
kubectl -n pra logs job/sqlite-restore-point
```

**Étape 5 — Relancer le pod Flask**
```bash
kubectl -n pra scale deployment flask --replicas=1
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```

**Étape 6 — Vérifier la restauration**
```bash
# Dans le navigateur :
# https://.../consultation
# https://.../count
```

**Capture d'écran de la restauration réussie :**

> 📸 <img width="895" height="274" alt="image" src="https://github.com/user-attachments/assets/dfefba5c-ca77-46d5-9004-bfdefce12e14" />
<img width="1885" height="342" alt="image" src="https://github.com/user-attachments/assets/75e2fc88-351d-4050-a358-73acb2a31408" />


