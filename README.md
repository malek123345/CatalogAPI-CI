# TP n°1 - Culture DevOps & Conteneurisation avec Docker

Projet : **ProductCatalog.Api** (ASP.NET Core 10 Minimal API + SQL Server 2022), orchestré avec Docker Compose.

## Lancer le projet

```bash
cp .env.example .env      # Windows : copy .env.example .env
docker compose up -d --build
docker compose ps
curl -i http://localhost:5000/health
```

Swagger : http://localhost:5000/swagger

## 1. Mesure et Observation

Relevé effectué avec `docker stats --no-stream` :

| Conteneur   | CPU %  | Mémoire         | Réseau I/O (rx/tx) | Block I/O        | PIDs |
|-------------|--------|-----------------|--------------------|------------------|------|
| catalog-api | 0.40 % | 52.07 MiB       | 10.9 kB / 7.43 kB  | 134 MB / 0 B     | 30   |
| catalog-db  | 1.91 % | 1001 MiB        | 8.55 kB / 9.97 kB  | 1.21 GB / 116 MB | 242  |

**Comparaison.** SQL Server consomme environ 20 fois plus de mémoire (1001 MiB contre 52 MiB), 8 fois plus de processus/threads (242 contre 30), et beaucoup plus d'I/O disque (1.21 GB lus contre 134 MB). C'est normal : c'est un moteur de base de données complet (buffer pool, tempdb, système de logs, agents internes) qui préalloue de la mémoire pour son cache, alors que l'API est un service léger et sans état qui ne fait que traduire des requêtes HTTP en requêtes SQL. Le trafic réseau reste faible pour les deux (quelques kB) car seuls quelques appels de test ont été faits.

**Pourquoi la mesure est-elle importante ?** Le « M » de CALMS : on ne peut pas améliorer ce qu'on ne mesure pas. La mesure permet de dimensionner les ressources (par exemple fixer une limite mémoire sur SQL Server), de détecter les régressions après un déploiement, de prouver l'effet d'un changement et d'alimenter la boucle de feedback entre Dev et Ops.

**Trois métriques d'une chaîne DevOps :**
1. **Fréquence de déploiement / lead time** : rapidité avec laquelle un commit arrive en production.
2. **Taux d'échec des changements** : pourcentage de déploiements causant un incident ou un rollback.
3. **MTTR** (Mean Time To Recovery) : temps moyen pour rétablir le service après un incident.

(Métriques techniques complémentaires : latence p95, taux d'erreurs 5xx, utilisation CPU/RAM.)

**Ping de port vs healthcheck applicatif.** Un port ouvert prouve seulement qu'un processus écoute, pas que l'application fonctionne. Pendant son démarrage, SQL Server peut accepter des connexions TCP sans être prêt à traiter des requêtes. Le healthcheck applicatif (`SELECT 1` via sqlcmd, endpoint `/health`) vérifie que le service répond réellement. Grâce à `depends_on: condition: service_healthy`, l'API ne démarre qu'une fois la base réellement prête, ce qui évite des crashs au démarrage. Nous l'avons constaté : lors d'un premier essai, `catalog-db` est resté `unhealthy` (mot de passe `sa` différent de celui d'un ancien volume), et l'API n'a pas été lancée tant que la base n'était pas saine.

## 2. Automatisation et Lean

Le multi-stage build compile l'application dans une image SDK jetable : aucun SDK .NET n'est requis sur le poste du développeur ni sur l'agent CI, ce qui supprime le « ça marche sur ma machine » et rend le build reproductible. L'image finale ne contient que le runtime ASP.NET et les binaires publiés, donc moins d'espace disque, moins de données à pousser/tirer vers le registry et moins de bande passante consommée. Le cache de couches est aussi exploité : `dotnet restore` n'est rejoué que si le `.csproj` change, ce qui accélère les builds CI (le build Compose suivant a duré moins d'une seconde grâce au cache).

## 3. Sécurité

La directive `USER app` exécute l'application avec un utilisateur non privilégié plutôt que root. En cas d'exploitation d'une faille (exécution de code à distance), l'attaquant n'a pas les droits root dans le conteneur : il ne peut pas modifier le système, installer des outils ou facilement tenter une évasion vers l'hôte. C'est l'application du principe du moindre privilège. S'y ajoutent la surface d'attaque réduite de l'image finale (pas de SDK ni de compilateur) et l'absence de secrets dans Git (`.env` ignoré, seul `.env.example` est versionné).

## 4. Analyse d'images

| Image                              | Disk usage | Content size |
|------------------------------------|-----------:|-------------:|
| `mcr.microsoft.com/dotnet/sdk:10.0`  | 1.3 GB     | 351 MB       |
| `productcatalog-api:1.0` (finale)  | 455 MB     | 126 MB       |

L'image finale est environ **3 fois plus petite** (taille disque) que l'image SDK (≈ 2,8 fois en taille compressée). L'écart s'explique par l'absence dans l'image finale du SDK .NET : compilateur Roslyn, MSBuild, NuGet, CLI `dotnet`, templates et outils de développement. Elle ne contient que le runtime ASP.NET Core (`aspnet:10.0`) et les DLL publiées. Note : l'énoncé annonce 800 à 900 Mo pour le SDK ; avec .NET 10 on mesure 1.3 GB.

## 5. Observations complémentaires

- **Réseau** : les conteneurs communiquent par nom de service (`db`) via le réseau bridge `catalog-net`.
- **Persistance** : sans volume, les données disparaissent à la suppression du conteneur (couche d'écriture éphémère) ; avec un volume managé monté sur `/var/opt/mssql`, elles survivent (testé avec `mssqldata-catalog`).
- **Secrets** : `.env` (réel, non versionné) et `.env.example` (modèle versionné).
- **Corrections apportées à l'énoncé** : nom du conteneur `catalog-db` (et non `catalogdb`) dans la chaîne de connexion, nom de volume `mssqldata-catalog` cohérent, versions des packages alignées sur .NET 10 (`Microsoft.AspNetCore.OpenApi` retiré, Swashbuckle 10.x).