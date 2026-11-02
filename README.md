# Deploying Metabase to Scalingo

1. Configure `git`:

```bash
$ scalingo --app dialog-metabase git-setup
```

2. Deploy the application:
```bash
$ git push scalingo main
```
# Updating Metabase on Scalingo

La version de Metabase n'est **pas figée dans ce dépôt** : elle est déterminée au
moment du build par le [buildpack Metabase Scalingo](https://github.com/Scalingo/metabase-buildpack)
via la variable d'environnement `METABASE_VERSION`.

- Si `METABASE_VERSION` est absente ou vaut `*`/`latest`, le buildpack récupère la
  **dernière version publiée** de Metabase.
- Sinon, il télécharge exactement la version demandée depuis
  `https://downloads.metabase.com/<METABASE_VERSION>/metabase.jar`.

Une simple redéploiement reconstruit l'application et télécharge le jar correspondant.

## Épingler une version précise (recommandé)

Épingler la version garantit des déploiements reproductibles et permet d'appliquer
un correctif de sécurité de façon contrôlée. Le format attendu est le tag Metabase,
par exemple `v0.64.3.4`.

```bash
# Définir la version cible
$ scalingo --app dialog-metabase env-set METABASE_VERSION=v0.64.3.4
```

## Procédure d'upgrade

1. (Optionnel) Épingler la version corrigée :

   ```bash
   $ scalingo --app dialog-metabase env-set METABASE_VERSION=v0.64.3.4
   ```

2. Redéployer pour reconstruire avec le nouveau jar :

   ```bash
   $ cd dialog-metabase
   $ git pull origin main
   $ git push scalingo main
   ```

   Si aucun changement n'est à committer (upgrade piloté uniquement par la variable
   d'environnement), forcez un redéploiement avec un commit vide :

   ```bash
   $ git commit --allow-empty -m "chore: bump Metabase to v0.64.3.4 (security fix)"
   $ git push scalingo main
   ```

3. Vérifier la version effective après démarrage via *Admin → Troubleshooting*
   ou l'endpoint `/api/session/properties` (champ `version`), puis tester les
   dashboards embarqués (https://dialog.beta.gouv.fr/stats) et les appels API.

---

## Protection par oauth2-proxy (ProConnect)

Cette instance est placée derrière [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/)
via le [buildpack betagouv](https://github.com/betagouv/oauth2-proxy-buildpack), conformément aux
[recommandations sécurité beta.gouv.fr](https://doc.incubateur.net/communaute/gerer-son-produit/aide-a-la-mise-en-application-des-standards/securite/securite-proxy-dauthentification).
Les utilisateurs non authentifiés via ProConnect n'atteignent jamais Metabase.

### Fonctionnement

- Le buildpack `oauth2-proxy-buildpack` est ajouté dans `.buildpacks`.
- Le `Procfile` lance `oauth2-proxy` en frontal : il écoute sur `$PORT` (port public Scalingo)
  et relaie les requêtes authentifiées vers Metabase, à qui le script attribue `PORT=8080`.
- Metabase écoute donc en interne sur `http://127.0.0.1:8080` (via `MB_JETTY_PORT`).

### Créer l'application OAuth ProConnect

Créez une application OAuth via l'[espace partenaires ProConnect](https://partenaires.proconnect.gouv.fr/)
avec l'URL de callback :

```
https://dialog-metabase.osc-fr1.scalingo.io/oauth2/callback
```

Vous obtenez un `client_id` et un `client_secret`.

### Variables d'environnement à définir sur Scalingo

À configurer via `scalingo --app dialog-metabase env-set ...` (ne pas committer les secrets) :

```bash
# --- ProConnect ---
OAUTH2_PROXY_PROVIDER=oidc
OAUTH2_PROXY_OIDC_ISSUER_URL=https://auth.agentconnect.gouv.fr/api/v2   # endpoint de prod
OAUTH2_PROXY_SCOPE="openid given_name usual_name email profile"
OAUTH2_PROXY_PROMPT=login

# --- Identifiants de l'app OAuth ProConnect ---
OAUTH2_PROXY_CLIENT_ID=<client_id>
OAUTH2_PROXY_CLIENT_SECRET=<client_secret>
OAUTH2_PROXY_COOKIE_SECRET=<secret_32_caracteres>   # ex: openssl rand -base64 32 | head -c 32

# --- Réseau / cookies ---
OAUTH2_PROXY_UPSTREAMS=http://127.0.0.1:8080        # Metabase en interne
OAUTH2_PROXY_REDIRECT_URL=https://dialog-metabase.osc-fr1.scalingo.io/oauth2/callback
OAUTH2_PROXY_COOKIE_DOMAINS=dialog-metabase.osc-fr1.scalingo.io
OAUTH2_PROXY_COOKIE_SECURE=true
OAUTH2_PROXY_REVERSE_PROXY=true                     # requêtes reçues derrière le routeur Scalingo
OAUTH2_PROXY_SKIP_PROVIDER_BUTTON=true              # redirige directement vers ProConnect

# --- Restriction d'accès par domaines email ---
OAUTH2_PROXY_EMAIL_DOMAINS="beta.gouv.fr,developpement-durable.gouv.fr"
```

### Exclure des routes de l'authentification (embeds Metabase, optionnel)

Si vous utilisez les dashboards embarqués Metabase, excluez les routes publiques :

```bash
OAUTH2_PROXY_SKIP_AUTH_ROUTES="^/app/embed\.js$,^/app/.*,^/embed/.*,^/api/embed/.*,^/api/geojson/*"
```

### Déploiement

```bash
$ git push scalingo master
```
