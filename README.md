# moodle-deploy

Déploiement de [Moodle](https://moodle.org) sur [Scalingo](https://scalingo.com),
avec PostgreSQL, Redis (optionnel) et les fichiers stockés sur
[Scaleway Object Storage](https://www.scaleway.com/fr/object-storage/).

Le code de Moodle n'est pas dans ce dépôt : il est téléchargé au build par
[moodle-buildpack](https://github.com/MathieuGilet/moodle-buildpack), dont le
README détaille le fonctionnement et toutes les variables d'environnement.

| Fichier | Rôle |
|---|---|
| `.buildpacks` | [moodle-buildpack](https://github.com/MathieuGilet/moodle-buildpack) (Moodle), puis le [buildpack PHP Scalingo](https://github.com/Scalingo/php-buildpack) (PHP, Nginx, Composer). L'ordre est important. |
| `Procfile` | Process `web` (Moodle, cron, envoi des fichiers vers S3) et hook `postdeploy` (installation / mise à jour de la base). |

## Déploiement

```bash
scalingo create mon-moodle
scalingo --app mon-moodle addons-add postgresql postgresql-starter-512
scalingo --app mon-moodle addons-add redis redis-starter-256   # optionnel (sessions)

scalingo --app mon-moodle env-set \
  MOODLE_VERSION=v5.3.0 \
  MOODLE_WWWROOT=https://mon-moodle.osc-fr1.scalingo.io \
  MOODLE_ADMIN_PASSWORD='un-mot-de-passe-solide' \
  MOODLE_ADMIN_EMAIL=admin@example.com \
  MOODLE_S3_BUCKET=mon-bucket-moodle \
  MOODLE_S3_REGION=fr-par \
  MOODLE_S3_ACCESS_KEY=SCWXXXXXXXXXXXXXXXXX \
  MOODLE_S3_SECRET_KEY=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

git remote add scalingo <url git de l'application>
git push scalingo main
```

Au premier déploiement, `postdeploy` installe la base de données Moodle ; aux
déploiements suivants, il lance la mise à jour (`admin/cli/upgrade.php`).

Pour changer de version de Moodle, modifiez `MOODLE_VERSION` puis redéployez
(`scalingo --app mon-moodle deploy` ou un nouveau `git push`).

> **Un seul conteneur web.** Les fichiers déposés sont d'abord écrits sur le
> disque du conteneur puis envoyés sur S3 par ce même conteneur : ne scalez pas
> le process `web` au-delà de 1 conteneur.

## Variables d'environnement principales

| Variable | Obligatoire | Description |
|---|---|---|
| `MOODLE_VERSION` | oui | Tag, branche ou SHA Git de Moodle (ex. `v5.3.0`). |
| `MOODLE_WWWROOT` | oui | URL publique de Moodle. |
| `MOODLE_ADMIN_PASSWORD` | à l'installation | Mot de passe du compte administrateur. |
| `MOODLE_S3_BUCKET` | — | Active le stockage des fichiers sur S3. |
| `MOODLE_S3_ACCESS_KEY` / `MOODLE_S3_SECRET_KEY` | avec S3 | Clé API Scaleway. |
| `MOODLE_S3_REGION` | — | Région Scaleway (défaut `fr-par`). |

`SCALINGO_POSTGRESQL_URL` et `SCALINGO_REDIS_URL` sont fournies par les addons.
Liste complète : [README de moodle-buildpack](https://github.com/MathieuGilet/moodle-buildpack#variables-denvironnement).

## Personnalisation

Fichiers optionnels pris en compte par le buildpack :

```
.
├── config.local.php       # réglages $CFG supplémentaires, inclus par config.php
├── patches/               # *.patch appliqués (git apply) sur le code de Moodle
└── moodle/                # copié par-dessus l'arborescence de Moodle
    └── public/
        ├── mod/monplugin/
        └── theme/montheme/
```

## Version du buildpack

`.buildpacks` suit la branche `main` de moodle-buildpack. Pour figer une
version, remplacez `#main` par un tag ou un SHA (ex. `#v1.0.0`).
