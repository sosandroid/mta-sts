🇫🇷 **Français** · [🇬🇧 English](README.en.md)

# MTA-STS pour un domaine personnel avec Proton Mail

Ce guide explique, étape par étape, comment protéger le courrier qui arrive sur votre domaine (par exemple `vous@exemple.fr`) hébergé chez **Proton Mail**, grâce à **MTA-STS**. Il ne demande aucune compétence informatique particulière : un compte GitHub gratuit et l'accès à la gestion DNS de votre domaine suffisent.

---

## Sommaire

1. [Avant de commencer](#1-avant-de-commencer)
2. [Créer le dépôt GitHub](#2-créer-le-dépôt-github)
3. [Vérifier votre domaine auprès de GitHub](#3-vérifier-votre-domaine-auprès-de-github)
4. [Publier le site avec GitHub Pages](#4-publier-le-site-avec-github-pages)
5. [Ajouter les enregistrements DNS MTA-STS et TLS-RPT](#5-ajouter-les-enregistrements-dns-mta-sts-et-tls-rpt)
6. [Vérifier que tout fonctionne](#6-vérifier-que-tout-fonctionne)
7. [Passer en mode enforce](#7-passer-en-mode-enforce)
8. [Suivi automatique de la politique Proton (API OVH)](#8-suivi-automatique-de-la-politique-proton-api-ovh)
9. [Pour aller plus loin : DNSSEC](#9-pour-aller-plus-loin--dnssec)
10. [En cas de problème](#10-en-cas-de-problème)

---

## 1. Avant de commencer

### À quoi sert MTA-STS ?

Quand quelqu'un vous écrit, son serveur de messagerie transmet le message aux serveurs de Proton. Par défaut, ce transfert est chiffré **seulement si tout se passe bien**. Une personne malveillante placée sur le trajet peut faire en sorte que le message passe en clair, sans que personne ne s'en aperçoive.

**MTA-STS** corrige ce problème. Vous publiez une règle (la « politique ») qui dit aux serveurs du monde entier :

> « Pour mon domaine, livrez le courrier **uniquement** aux serveurs de Proton, **uniquement** de façon chiffrée, avec un certificat valide. Sinon, ne livrez pas. »

Proton applique déjà cette règle pour ses propres adresses (`@proton.me`, `@protonmail.com`). En revanche, **Proton ne la publie pas pour les domaines personnels** : c'est à vous de le faire, et c'est l'objet de ce guide.

### ⚠️ Pour qui est cette méthode ?

L'hébergement sur **GitHub Pages** proposé ici fonctionne bien. Il est gratuit et le certificat HTTPS (Let's Encrypt) est renouvelé automatiquement. Mais il est **à réserver aux particuliers**.

**Les entreprises et les organisations devraient héberger elles-mêmes ce type de règle de sécurité**, sur leur propre infrastructure, par exemple un serveur web ou un reverse proxy qu'elles administrent. Plusieurs raisons à cela :
- **Maîtrise** : la politique de sécurité de la messagerie ne dépend pas d'un compte GitHub personnel ni des conditions d'un service tiers.
- **Disponibilité et supervision** : le certificat, le site et le DNS sont surveillés par les outils internes.
- **Traçabilité** : toute modification passe par les procédures de changement de l'entreprise.
- **Conformité** : il n'y a pas de dépendance non maîtrisée dans la chaîne de sécurité.

### Ce qu'il vous faut

| Élément | Exemple |
|---|---|
| Un domaine dont le courrier est déjà chez Proton Mail | `exemple.fr` |
| L'accès à la zone DNS du domaine | espace client OVH, Gandi, Cloudflare… |
| Un compte GitHub (gratuit) | https://github.com/signup |
| Une adresse pour recevoir les rapports | `tlsrpt@exemple.fr` (alias Proton) |

> 💡 Dans tout le guide, **remplacez `exemple.fr` par votre domaine** et **`votre-compte` par votre identifiant GitHub**.

### Vérifier que vos MX pointent bien vers Proton

Dans votre zone DNS, vous devez déjà avoir ces deux enregistrements, mis en place lors de la configuration de Proton :

```
@   MX   10 mail.protonmail.ch.
@   MX   20 mailsec.protonmail.ch.
```

---

## 2. Créer le dépôt GitHub

1. Connectez-vous à GitHub, cliquez sur **+** en haut à droite, puis sur **New repository**.
2. **Repository name** : `mta-sts`
3. Cochez **Public**. GitHub Pages est gratuit uniquement pour les dépôts publics.
4. Cochez **Add a README file**, puis cliquez sur **Create repository**.

Vous allez maintenant créer deux fichiers. Pour chacun :
cliquez sur **Add file → Create new file**, tapez le **nom exact** (les `/` créent les dossiers), collez le contenu, puis cliquez sur **Commit changes… → Commit changes**.

### Fichier 1 : `.nojekyll`

- **Nom** : `.nojekyll` (avec le point au début)
- **Contenu** : laissez vide

> Ce fichier est **indispensable**. Sans lui, GitHub ignore les dossiers dont le nom commence par un point, et votre politique serait introuvable.

### Fichier 2 : `.well-known/mta-sts.txt`

- **Nom** : `.well-known/mta-sts.txt`
- **Contenu** :

```
version: STSv1
mode: testing
mx: mail.protonmail.ch
mx: mailsec.protonmail.ch
max_age: 86400
```

> On commence volontairement en mode **`testing`**, avec une durée courte (`86400` secondes, soit 1 jour). Les serveurs expéditeurs signalent les problèmes **sans bloquer** le courrier. On passera en mode strict à l'étape 7.

### Fichier facultatif : `index.html`

Vous pouvez ajouter une page d'accueil qui explique la démarche aux visiteurs de `https://mta-sts.exemple.fr/`. Le fichier `index.html` fourni avec ce guide détecte votre domaine tout seul : il n'y a rien à modifier.

---

## 3. Vérifier votre domaine auprès de GitHub

Cette étape empêche quelqu'un d'autre d'utiliser votre sous-domaine sur GitHub, par exemple si vous supprimez un jour le dépôt.

1. Cliquez sur votre **photo de profil**, puis sur **Settings**. Dans le menu de gauche, ouvrez **Pages**.
2. Cliquez sur **Add a domain**, saisissez `exemple.fr`, puis cliquez sur **Add domain**.
3. GitHub affiche un enregistrement **TXT** à créer, de la forme `_github-pages-challenge-votre-compte`, avec une valeur.
4. Créez cet enregistrement dans votre zone DNS (voir l'encadré ci-dessous).
5. Revenez sur GitHub et cliquez sur **Verify**. La vérification peut prendre jusqu'à quelques heures.

> **📘 Ajouter un enregistrement chez OVH**
> Espace client → **Web Cloud** → **Noms de domaine** → `exemple.fr` → onglet **Zone DNS** → **Ajouter une entrée**.
> Choisissez le type (CNAME, TXT…), remplissez le **sous-domaine** et la **cible**, puis validez.
> Pour un TXT, saisissez la valeur **sans guillemets** : OVH les ajoute lui-même.
> Les autres hébergeurs DNS proposent un formulaire équivalent.

---

## 4. Publier le site avec GitHub Pages

### 4.1 Créer le sous-domaine `mta-sts`

Dans votre zone DNS, ajoutez :

| Type | Sous-domaine | Cible |
|---|---|---|
| CNAME | `mta-sts` | `votre-compte.github.io.` |

### 4.2 Activer Pages

1. Dans le dépôt `mta-sts`, cliquez sur **Settings**, puis sur **Pages** dans le menu de gauche.
2. **Source** : *Deploy from a branch*. **Branch** : `main`, dossier `/ (root)`. Cliquez sur **Save**.
3. **Custom domain** : saisissez `mta-sts.exemple.fr`, puis cliquez sur **Save**.
   GitHub crée automatiquement un fichier `CNAME` dans le dépôt : ne le supprimez pas.
4. Attendez la coche verte **DNS check successful**. Cela prend de quelques minutes à une heure.
5. Cochez **Enforce HTTPS**.

> La case **Enforce HTTPS** reste grisée tant que GitHub n'a pas obtenu le certificat Let's Encrypt. Patientez, puis rechargez la page.
> Si votre zone contient un enregistrement **CAA**, il doit autoriser `letsencrypt.org`.

### 4.3 Contrôle rapide

Ouvrez cette adresse dans votre navigateur :
**https://mta-sts.exemple.fr/.well-known/mta-sts.txt**

Vous devez voir le texte de votre politique, avec un cadenas dans la barre d'adresse.

---

## 5. Ajouter les enregistrements DNS MTA-STS et TLS-RPT

Dans votre zone DNS, ajoutez ces deux enregistrements **TXT** :

| Type | Sous-domaine | Valeur | Rôle |
|---|---|---|---|
| TXT | `_mta-sts` | `v=STSv1; id=20261114T1500` | Annonce qu'une politique existe |
| TXT | `_smtp._tls` | `v=TLSRPTv1; rua=mailto:tlsrpt@exemple.fr` | Adresse qui recevra les rapports |

**À propos de l'`id`**
- C'est un simple numéro de version : lettres et chiffres, 32 caractères au maximum. Utiliser la date et l'heure est pratique.
- **À chaque modification du fichier `mta-sts.txt`, changez l'`id`.** Sinon, les serveurs expéditeurs continuent d'utiliser l'ancienne version, qu'ils gardent en mémoire.

**À propos des rapports (TLS-RPT)**
- Dans Proton, créez l'adresse `tlsrpt@exemple.fr` en alias, avec un filtre qui range ses messages dans un dossier à part.
- Vous recevrez environ un rapport par jour et par grand expéditeur (Google, Microsoft…), sous forme de pièce jointe `.json.gz`.
- Utilisez votre propre adresse, pas celle d'un service tiers, pour **voir** vous-même les éventuels problèmes.

---

## 6. Vérifier que tout fonctionne

### Le plus simple : un test en ligne

Rendez-vous sur **https://internet.nl/test-mail/**, saisissez `exemple.fr`, puis lancez le test. La partie *STARTTLS et DANE* et les indicateurs MTA-STS / TLS-RPT doivent être au vert. Attendez quelques heures après vos modifications DNS avant de lancer le test.

### En ligne de commande (facultatif)

**macOS / Linux**
```bash
dig +short TXT _mta-sts.exemple.fr
dig +short TXT _smtp._tls.exemple.fr
curl -s https://mta-sts.exemple.fr/.well-known/mta-sts.txt
```

**Windows (PowerShell)**
```powershell
Resolve-DnsName -Type TXT _mta-sts.exemple.fr
Resolve-DnsName -Type TXT _smtp._tls.exemple.fr
Invoke-RestMethod https://mta-sts.exemple.fr/.well-known/mta-sts.txt
```

---

## 7. Passer en mode enforce

Après **une à deux semaines** en mode `testing`, si les rapports reçus ne signalent pas d'échec :

1. Dans le dépôt, ouvrez `.well-known/mta-sts.txt`, cliquez sur le **crayon** (*Edit*) et remplacez le contenu par :

   ```
   version: STSv1
   mode: enforce
   mx: mail.protonmail.ch
   mx: mailsec.protonmail.ch
   max_age: 604800
   ```

   C'est exactement la politique que Proton publie pour ses propres domaines. `604800` secondes correspondent à 7 jours.
2. Cliquez sur **Commit changes**.
3. Attendez une ou deux minutes, puis vérifiez dans le navigateur que la nouvelle version est en ligne.
4. **Ensuite seulement**, dans votre zone DNS, modifiez le TXT `_mta-sts` avec un nouvel `id`, par exemple `v=STSv1; id=20261128T1000`.

> **L'ordre est important** : d'abord le fichier, ensuite l'`id`. Dans l'autre sens, un serveur pourrait télécharger l'ancien fichier et le garder une semaine.

---

## 8. Suivi automatique de la politique Proton (API OVH)

Proton modifie très rarement sa politique. Si cela arrive, il faut mettre à jour votre copie **puis** changer l'`id` dans votre DNS. Cette étape automatise les deux, chaque semaine, grâce à une **GitHub Action**, c'est-à-dire un petit programme que GitHub exécute pour vous.

> ℹ️ L'exemple utilise l'**API d'OVH**. **La même méthode fonctionne avec tout hébergeur DNS qui propose une API** (Cloudflare, Gandi, deSEC, Infomaniak, Scaleway…). Seule la dernière étape du programme, « Changer l'id du TXT », est à adapter.
> Si votre hébergeur DNS n'a pas d'API, vous pouvez quand même utiliser l'action : supprimez la dernière étape et changez l'`id` à la main lorsque GitHub vous le signale.

### Ce que fait l'action, chaque lundi matin

1. Elle télécharge la politique de Proton et **la contrôle** : format correct, uniquement des serveurs `*.protonmail.ch`. Si le fichier semble anormal, elle ne modifie rien et vous alerte.
2. Si les serveurs de Proton ont changé, elle met à jour **uniquement les lignes `mx:`** de votre fichier. Votre `mode` et votre `max_age` sont conservés.
3. Elle publie le nouveau fichier et **attend qu'il soit réellement en ligne**.
4. Elle change ensuite l'`id` du TXT `_mta-sts` chez OVH.
5. En cas d'erreur, elle s'arrête et GitHub vous prévient par e-mail.

### 8.1 Trouver l'identifiant de l'enregistrement `_mta-sts` chez OVH

1. Ouvrez la console de l'API : **https://eu.api.ovh.com/console** et connectez-vous avec votre compte OVH.
2. Cherchez **`GET /domain/zone/{zoneName}/record`**.
3. Renseignez `zoneName` = `exemple.fr`, `fieldType` = `TXT`, `subDomain` = `_mta-sts`, puis cliquez sur **Execute**.
4. Notez le nombre renvoyé, par exemple `[5123456789]`. C'est l'**identifiant de l'enregistrement**.

### 8.2 Créer des clés d'API limitées

1. Ouvrez **https://eu.api.ovh.com/createToken/**.
2. Renseignez un nom (`github-mta-sts`), une description, et choisissez une validité **Unlimited**.
3. Dans **Rights**, ajoutez **exactement** ces trois lignes, en remplaçant le domaine et l'identifiant :

   | Méthode | Chemin |
   |---|---|
   | GET | `/domain/zone/exemple.fr/record/5123456789` |
   | PUT | `/domain/zone/exemple.fr/record/5123456789` |
   | POST | `/domain/zone/exemple.fr/refresh` |

4. Validez. OVH affiche trois clés : **Application Key**, **Application Secret** et **Consumer Key**. Copiez-les.

> 🔒 **Ne donnez pas plus de droits.** Avec ces trois lignes, les clés ne peuvent modifier **que** l'enregistrement `_mta-sts`. Si elles étaient volées, vos MX et le reste de votre zone resteraient protégés.

### 8.3 Enregistrer les clés dans GitHub

Dans le dépôt, allez dans **Settings** → **Secrets and variables** → **Actions**.

- Onglet **Secrets**, cliquez trois fois sur **New repository secret** :

  | Nom | Valeur |
  |---|---|
  | `OVH_AK` | Application Key |
  | `OVH_AS` | Application Secret |
  | `OVH_CK` | Consumer Key |

- Onglet **Variables**, cliquez sur **New repository variable** :

  | Nom | Valeur |
  |---|---|
  | `OVH_MTASTS_RECORD_ID` | l'identifiant noté en 8.1 |

### 8.4 Créer l'action

Cliquez sur **Add file → Create new file**. Nommez le fichier `.github/workflows/sync-mta-sts.yml`, collez le contenu ci-dessous, puis **remplacez `exemple.fr`** à la ligne `DOMAIN:`. Enfin, cliquez sur **Commit changes**.

```yaml
name: sync-mta-sts

on:
  schedule:
    - cron: "17 6 * * 1"     # chaque lundi à 06:17 UTC
  workflow_dispatch:
    inputs:
      force_dns:
        description: "Tester la mise à jour de l'id DNS même sans changement"
        type: boolean
        default: false

permissions:
  contents: write
  pages: write

env:
  DOMAIN: exemple.fr

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Récupérer et valider la politique Proton
        run: |
          curl -fsS --retry 3 https://mta-sts.protonmail.com/.well-known/mta-sts.txt | tr -d '\r' > proton.txt
          grep -qx 'version: STSv1' proton.txt || { echo "::error::Politique Proton invalide"; exit 1; }
          grep '^mx:' proton.txt | sort > mx.new
          [ -s mx.new ] || { echo "::error::Aucun MX dans la politique Proton"; exit 1; }
          if grep -v -E '^mx: [a-z0-9.-]+\.protonmail\.ch$' mx.new; then
            echo "::error::MX inattendu dans la politique Proton : aucune modification"; exit 1
          fi

      - name: Mettre à jour les MX (mode et max_age conservés)
        id: upd
        run: |
          f=.well-known/mta-sts.txt
          if diff <(grep '^mx:' "$f" | tr -d '\r' | sort) mx.new; then
            echo "changed=false" >> "$GITHUB_OUTPUT"
          else
            { echo "version: STSv1"; grep '^mode:' "$f"; cat mx.new; grep '^max_age:' "$f"; } > new.txt
            mv new.txt "$f"
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Committer et republier
        if: steps.upd.outputs.changed == 'true'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898319+github-actions[bot]@users.noreply.github.com"
          git commit -am "MTA-STS : MX synchronisés avec Proton"
          git push
          gh api -X POST "repos/${GITHUB_REPOSITORY}/pages/builds" || true

      - name: Attendre que la nouvelle politique soit en ligne
        if: steps.upd.outputs.changed == 'true'
        run: |
          for i in $(seq 1 30); do
            if diff -q <(curl -fsS "https://mta-sts.${DOMAIN}/.well-known/mta-sts.txt?n=$i" | tr -d '\r') \
                       <(tr -d '\r' < .well-known/mta-sts.txt) > /dev/null; then
              echo "Nouvelle politique en ligne"; exit 0
            fi
            sleep 30
          done
          echo "::error::Politique pas en ligne après 15 min : id DNS NON modifié"; exit 1

      # ↓ Étape propre à OVH : à adapter pour un autre hébergeur DNS
      - name: Changer l'id du TXT _mta-sts chez OVH
        if: steps.upd.outputs.changed == 'true' || inputs.force_dns
        env:
          OVH_AK: ${{ secrets.OVH_AK }}
          OVH_AS: ${{ secrets.OVH_AS }}
          OVH_CK: ${{ secrets.OVH_CK }}
          RECORD_ID: ${{ vars.OVH_MTASTS_RECORD_ID }}
        run: |
          ovh() {  # usage : ovh MÉTHODE /chemin [corps JSON]
            local m="$1" url="https://eu.api.ovh.com/1.0$2" body="${3:-}" ts sig args=()
            ts=$(curl -fsS https://eu.api.ovh.com/1.0/auth/time)
            sig='$1$'$(printf '%s' "${OVH_AS}+${OVH_CK}+${m}+${url}+${body}+${ts}" | sha1sum | cut -d' ' -f1)
            [ -n "$body" ] && args=(--data "$body")
            curl -fsS -X "$m" "$url" -H "Content-Type: application/json" \
              -H "X-Ovh-Application: ${OVH_AK}" -H "X-Ovh-Consumer: ${OVH_CK}" \
              -H "X-Ovh-Timestamp: ${ts}" -H "X-Ovh-Signature: ${sig}" "${args[@]}"
          }

          ovh GET "/domain/zone/${DOMAIN}/record/${RECORD_ID}" > rec.json
          jq -e '.fieldType == "TXT" and .subDomain == "_mta-sts"' rec.json > /dev/null \
            || { echo "::error::L'enregistrement ${RECORD_ID} n'est pas le TXT _mta-sts"; exit 1; }

          new="v=STSv1; id=$(date -u +%Y%m%d%H%M%S)"
          ovh PUT  "/domain/zone/${DOMAIN}/record/${RECORD_ID}" "$(jq -cn --arg t "$new" '{target: $t}')"
          ovh POST "/domain/zone/${DOMAIN}/refresh"
          echo "::notice::_mta-sts.${DOMAIN} → ${new}"
```

### 8.5 Tester l'action

1. Dans le dépôt, ouvrez l'onglet **Actions**, puis **sync-mta-sts**, et cliquez sur **Run workflow**.
2. Cochez **force_dns**, puis cliquez sur **Run workflow**.
3. Au bout d'une minute environ, une **coche verte** doit apparaître.
4. Vérifiez que l'`id` a changé, sur https://internet.nl ou avec `dig +short TXT _mta-sts.exemple.fr`.

> Changer l'`id` sans modifier le fichier ne pose aucun problème : les serveurs téléchargent simplement la même politique une nouvelle fois.

### 8.6 Être prévenu en cas d'erreur

Cliquez sur votre **photo de profil** → **Settings** → **Notifications**. Dans la section **System → Actions**, cochez **Email** et **Only notify for failed workflows**.

> ⏰ **Bon à savoir** : sur un dépôt public, GitHub **met en pause les actions planifiées après 60 jours sans activité**. Il vous prévient par e-mail avant. Il suffit alors de cliquer sur **Enable workflow** dans l'onglet *Actions*.

---

## 9. Pour aller plus loin : DNSSEC

Proton publie des enregistrements **TLSA** (DANE) sur ses serveurs de messagerie. **Vous n'avez aucun enregistrement TLSA à créer** dans votre zone. Il suffit d'**activer DNSSEC** sur votre domaine :

- **Chez OVH** : Web Cloud → Noms de domaine → `exemple.fr` → onglet **DNSSEC** → Activer.
- **Ailleurs** : cherchez l'option DNSSEC dans la gestion du domaine.

Les serveurs qui savent utiliser DANE (Postfix, de nombreux fournisseurs européens) vérifieront alors aussi les certificats de Proton. MTA-STS (Gmail, Microsoft) et DANE se complètent.

---

## 10. En cas de problème

| Symptôme | Cause probable | Solution |
|---|---|---|
| `https://mta-sts.exemple.fr/.well-known/mta-sts.txt` affiche une erreur 404 | Fichier `.nojekyll` absent | Créez-le (étape 2) |
| *Enforce HTTPS* reste grisé | DNS pas encore propagé, ou CAA bloquant | Attendez, vérifiez le CNAME et le CAA (étape 4) |
| Erreur de certificat dans le navigateur | Certificat pas encore délivré | Retirez puis remettez le *Custom domain* dans Settings → Pages |
| internet.nl ne voit pas MTA-STS | TXT `_mta-sts` absent ou mal saisi | Vérifiez la valeur `v=STSv1; id=…` (étape 5) |
| L'action échoue à l'étape OVH (`403`) | Droits des clés incorrects | Recréez les clés avec les 3 lignes exactes (étape 8.2) |
| L'action échoue à l'étape OVH (`Invalid signature`) | Clé mal copiée dans les Secrets | Recopiez `OVH_AS` et `OVH_CK` (étape 8.3) |
| L'action ne s'exécute plus | Pause après 60 jours d'inactivité | Onglet *Actions* → **Enable workflow** |

---

## Références

- [RFC 8461 — MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) · [RFC 8460 — TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460)
- [privsec.dev — MTA-STS avec un domaine personnalisé chez Proton](https://privsec.dev/posts/knowledge/setting-up-mta-sts-with-a-custom-domain-on-proton-mail/)
- [GitHub Docs — HTTPS pour GitHub Pages](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
- [OVHcloud — Premiers pas avec les API](https://docs.ovhcloud.com/fr/guides/manage-and-operate/api/first-steps)
- [internet.nl — test de messagerie](https://internet.nl/test-mail/)

🇫🇷 **Français** · [🇬🇧 English](README.en.md)
