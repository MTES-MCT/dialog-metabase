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

To upgrade to the latest version of Metabase, you only need to redeploy it,
this will retrieve the latest version avaible on [the Metabase buildpack](https://github.com/metabase/metabase-buildpack).

## Updating After Deploying Using Scalingo's Command Line Tool

```bash
$ cd metabase-scalingo
$ git pull origin main
$ git push scalingo main
```

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
