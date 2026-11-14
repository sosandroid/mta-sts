# MTA-STS pour domaine perso avec ProtonMail

Créez un dépôt public dédié, par exemple mta-sts. Un dépôt public est nécessaire pour Pages sur l’offre gratuite, et chaque site Pages n’accepte qu’un seul domaine personnalisé. Le dépôt contient :

````
.nojekyll                      ← fichier vide, INDISPENSABLE
.well-known/mta-sts.txt
CNAME                          ← contient : mta-sts.exemple.fr
````

Sans .nojekyll, Jekyll ignore les dossiers qui commencent par un point, et /.well-known/ répondrait 404.

2. fichier mta-sts.txt
.well-known/mta-sts.txt, copie de la politique de Proton. D’après privsec.dev, Proton publie version: STSv1 / mode: enforce / mx: mail.protonmail.ch / mx: mailsec.protonmail.ch / max_age: 604800 : 
privsec

````
version: STSv1
mode: enforce
mx: mail.protonmail.ch
mx: mailsec.protonmail.ch
max_age: 604800
````
3. DNS de exemple.fr

````dns
mta-sts       CNAME  <votre-compte>.github.io.
_mta-sts      TXT    "v=STSv1; id=20261113T1800"
_smtp._tls    TXT    "v=TLSRPTv1; rua=mailto:tlsrpt@exemple.fr"
````
Si vous avez un enregistrement CAA, il doit autoriser letsencrypt.org. Sinon, le certificat ne sera pas délivré.

4. Dans GitHub, allez dans Settings → Pages et choisissez Source : Deploy from a branch (main, dossier /). Indiquez le domaine mta-sts.exemple.fr, attendez la coche verte, puis cochez Enforce HTTPS. Cette case reste grisée tant que la vérification DNS de GitHub n’a pas réussi et que le certificat Let’s Encrypt n’est pas délivré. 
is-pro

5. Vérifiez le domaine dans les paramètres de votre compte GitHub (Settings → Pages → Add a verified domain). Cela ajoute un TXT _github-pages-challenge-… et empêche quelqu’un d’autre de récupérer votre sous-domaine si le dépôt est supprimé.

6. Contrôle

````bash
curl -sI https://mta-sts.exemple.fr/.well-known/mta-sts.txt   # 200, text/plain
echo | openssl s_client -connect mta-sts.exemple.fr:443 -servername mta-sts.exemple.fr 2>/dev/null | openssl x509 -noout -issuer
# issuer: C=US, O=Let's Encrypt, ...
````
Rester aligné sur Proton (facultatif)

Inutile de synchroniser automatiquement : à chaque changement du fichier, il faudrait aussi changer l’id dans votre DNS. Le plus sûr est une alerte. Une GitHub Action hebdomadaire compare votre copie avec celle de Proton. Si elle échoue, GitHub vous envoie un e-mail.

````yaml
# .github/workflows/check-proton.yml
name: check-proton-policy
on:
  schedule: [{ cron: "17 6 * * 1" }]
  workflow_dispatch:
jobs:
  diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          curl -fsS https://mta-sts.protonmail.com/.well-known/mta-sts.txt -o proton.txt
          diff <(grep '^mx:' proton.txt | sort) <(grep '^mx:' .well-known/mta-sts.txt | sort)
````
Le test ne compare que les lignes mx:, qui sont les seules à devoir correspondre exactement. Vous restez ainsi libre de choisir votre mode et votre max_age.
