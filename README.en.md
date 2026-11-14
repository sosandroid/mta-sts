[🇫🇷 Français](README.md) · 🇬🇧 **English**

# MTA-STS for a personal domain with Proton Mail

This guide explains, step by step, how to protect the mail that reaches your domain (for example `you@example.com`) hosted on **Proton Mail**, using **MTA-STS**. No particular IT skills are needed: a free GitHub account and access to your domain's DNS settings are enough.

---

## Contents

1. [Before you start](#1-before-you-start)
2. [Create the GitHub repository](#2-create-the-github-repository)
3. [Verify your domain with GitHub](#3-verify-your-domain-with-github)
4. [Publish the site with GitHub Pages](#4-publish-the-site-with-github-pages)
5. [Add the MTA-STS and TLS-RPT DNS records](#5-add-the-mta-sts-and-tls-rpt-dns-records)
6. [Check that everything works](#6-check-that-everything-works)
7. [Switch to enforce mode](#7-switch-to-enforce-mode)
8. [Automatic tracking of Proton's policy (OVH API)](#8-automatic-tracking-of-protons-policy-ovh-api)
9. [Going further: DNSSEC](#9-going-further-dnssec)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Before you start

### What is MTA-STS for?

When someone writes to you, their mail server hands the message over to Proton's servers. By default, this transfer is encrypted **only if everything goes well**. An attacker sitting on the path can make the message travel unencrypted, without anyone noticing.

**MTA-STS** fixes this. You publish a rule (the "policy") that tells every mail server in the world:

> "For my domain, deliver mail **only** to Proton's servers, **only** over an encrypted connection, with a valid certificate. Otherwise, do not deliver."

Proton already applies this rule to its own addresses (`@proton.me`, `@protonmail.com`). However, **Proton does not publish it for custom domains**: it is up to you, and that is what this guide is about.

### ⚠️ Who is this method for?

The **GitHub Pages** hosting described here works well. It is free, and the HTTPS certificate (Let's Encrypt) renews automatically. But it is **meant for individuals**.

**Companies and organisations should host this kind of security policy themselves**, on infrastructure they run, such as a web server or reverse proxy they administer. There are several reasons for this:
- **Control**: the mail security policy does not depend on a personal GitHub account or on a third party's terms of service.
- **Availability and monitoring**: the certificate, the site and DNS are watched by internal tools.
- **Traceability**: every change goes through the company's change-management process.
- **Compliance**: there is no unmanaged dependency in the security chain.

### What you need

| Item | Example |
|---|---|
| A domain whose mail is already on Proton Mail | `example.com` |
| Access to the domain's DNS zone | OVH, Gandi, Cloudflare… control panel |
| A GitHub account (free) | https://github.com/signup |
| An address to receive reports | `tlsrpt@example.com` (Proton alias) |

> 💡 Throughout this guide, **replace `example.com` with your domain** and **`your-account` with your GitHub username**.

### Check that your MX records point to Proton

Your DNS zone should already contain these two records, set up when you configured Proton:

```
@   MX   10 mail.protonmail.ch.
@   MX   20 mailsec.protonmail.ch.
```

---

## 2. Create the GitHub repository

1. Sign in to GitHub, click **+** at the top right, then **New repository**.
2. **Repository name**: `mta-sts`
3. Select **Public**. GitHub Pages is free only for public repositories.
4. Tick **Add a README file**, then click **Create repository**.

You will now create two files. For each one:
click **Add file → Create new file**, type the **exact name** (`/` creates folders), paste the content, then click **Commit changes… → Commit changes**.

### File 1: `.nojekyll`

- **Name**: `.nojekyll` (with the leading dot)
- **Content**: leave empty

> This file is **essential**. Without it, GitHub ignores folders whose name starts with a dot, and your policy could not be found.

### File 2: `.well-known/mta-sts.txt`

- **Name**: `.well-known/mta-sts.txt`
- **Content**:

```
version: STSv1
mode: testing
mx: mail.protonmail.ch
mx: mailsec.protonmail.ch
max_age: 86400
```

> We deliberately start in **`testing`** mode with a short lifetime (`86400` seconds, i.e. 1 day). Sending servers report problems **without blocking** mail. We will switch to strict mode in step 7.

### Optional file: `index.html`

You can add a home page that explains the setup to visitors of `https://mta-sts.example.com/`. The `index.html` file provided with this guide detects your domain on its own: there is nothing to edit.

---

## 3. Verify your domain with GitHub

This step stops anyone else from using your subdomain on GitHub, for example if you delete the repository one day.

1. Click your **profile picture**, then **Settings**. In the left menu, open **Pages**.
2. Click **Add a domain**, enter `example.com`, then click **Add domain**.
3. GitHub shows a **TXT** record to create, of the form `_github-pages-challenge-your-account`, with a value.
4. Create this record in your DNS zone (see the box below).
5. Go back to GitHub and click **Verify**. Verification can take up to a few hours.

> **📘 Adding a record at OVH**
> Control panel → **Web Cloud** → **Domain names** → `example.com` → **DNS zone** tab → **Add an entry**.
> Choose the type (CNAME, TXT…), fill in the **subdomain** and the **target**, then confirm.
> For a TXT record, enter the value **without quotes**: OVH adds them itself.
> Other DNS providers offer an equivalent form.

---

## 4. Publish the site with GitHub Pages

### 4.1 Create the `mta-sts` subdomain

In your DNS zone, add:

| Type | Subdomain | Target |
|---|---|---|
| CNAME | `mta-sts` | `your-account.github.io.` |

### 4.2 Turn on Pages

1. In the `mta-sts` repository, click **Settings**, then **Pages** in the left menu.
2. **Source**: *Deploy from a branch*. **Branch**: `main`, folder `/ (root)`. Click **Save**.
3. **Custom domain**: enter `mta-sts.example.com`, then click **Save**.
   GitHub automatically creates a `CNAME` file in the repository: do not delete it.
4. Wait for the green tick **DNS check successful**. This takes from a few minutes to an hour.
5. Tick **Enforce HTTPS**.

> The **Enforce HTTPS** box stays greyed out until GitHub has obtained the Let's Encrypt certificate. Wait, then reload the page.
> If your zone has a **CAA** record, it must allow `letsencrypt.org`.

### 4.3 Quick check

Open this address in your browser:
**https://mta-sts.example.com/.well-known/mta-sts.txt**

You should see the text of your policy, with a padlock in the address bar.

---

## 5. Add the MTA-STS and TLS-RPT DNS records

In your DNS zone, add these two **TXT** records:

| Type | Subdomain | Value | Purpose |
|---|---|---|---|
| TXT | `_mta-sts` | `v=STSv1; id=20261114T1500` | Announces that a policy exists |
| TXT | `_smtp._tls` | `v=TLSRPTv1; rua=mailto:tlsrpt@example.com` | Address that will receive reports |

**About the `id`**
- It is just a version number: letters and digits, 32 characters at most. Using the date and time is convenient.
- **Every time you change `mta-sts.txt`, change the `id`.** Otherwise, sending servers keep using the old version they have stored.

**About reports (TLS-RPT)**
- In Proton, create `tlsrpt@example.com` as an alias, with a filter that moves its messages to a separate folder.
- You will receive about one report per day from each major sender (Google, Microsoft…), as a `.json.gz` attachment.
- Use your own address, not a third-party service's, so that **you** can see any problems.

---

## 6. Check that everything works

### The simplest way: an online test

Go to **https://internet.nl/test-mail/**, enter `example.com` and run the test. The *STARTTLS and DANE* section and the MTA-STS / TLS-RPT indicators should be green. Wait a few hours after your DNS changes before running the test.

### From the command line (optional)

**macOS / Linux**
```bash
dig +short TXT _mta-sts.example.com
dig +short TXT _smtp._tls.example.com
curl -s https://mta-sts.example.com/.well-known/mta-sts.txt
```

**Windows (PowerShell)**
```powershell
Resolve-DnsName -Type TXT _mta-sts.example.com
Resolve-DnsName -Type TXT _smtp._tls.example.com
Invoke-RestMethod https://mta-sts.example.com/.well-known/mta-sts.txt
```

---

## 7. Switch to enforce mode

After **one to two weeks** in `testing` mode, if the reports you receive show no failures:

1. In the repository, open `.well-known/mta-sts.txt`, click the **pencil** (*Edit*) and replace the content with:

   ```
   version: STSv1
   mode: enforce
   mx: mail.protonmail.ch
   mx: mailsec.protonmail.ch
   max_age: 604800
   ```

   This is exactly the policy Proton publishes for its own domains. `604800` seconds is 7 days.
2. Click **Commit changes**.
3. Wait a minute or two, then check in your browser that the new version is online.
4. **Only then**, in your DNS zone, update the `_mta-sts` TXT record with a new `id`, for example `v=STSv1; id=20261128T1000`.

> **The order matters**: file first, then `id`. The other way round, a server could download the old file and keep it for a week.

---

## 8. Automatic tracking of Proton's policy (OVH API)

Proton very rarely changes its policy. If it does, you must update your copy **and then** change the `id` in your DNS. This step automates both, every week, with a **GitHub Action**: a small program that GitHub runs for you.

> ℹ️ The example uses the **OVH API**. **The same method works with any DNS provider that offers an API** (Cloudflare, Gandi, deSEC, Infomaniak, Scaleway…). Only the last step of the program, "Change the id of the TXT record", needs adapting.
> If your DNS provider has no API, you can still use the action: remove the last step and change the `id` by hand when GitHub alerts you.

### What the action does, every Monday morning

1. It downloads Proton's policy and **checks it**: correct format, only `*.protonmail.ch` servers. If the file looks wrong, it changes nothing and alerts you.
2. If Proton's servers have changed, it updates **only the `mx:` lines** of your file. Your `mode` and `max_age` are kept.
3. It publishes the new file and **waits until it is actually online**.
4. It then changes the `id` of the `_mta-sts` TXT record at OVH.
5. If anything fails, it stops and GitHub notifies you by email.

### 8.1 Find the ID of the `_mta-sts` record at OVH

1. Open the API console: **https://eu.api.ovh.com/console** and sign in with your OVH account.
2. Look for **`GET /domain/zone/{zoneName}/record`**.
3. Fill in `zoneName` = `example.com`, `fieldType` = `TXT`, `subDomain` = `_mta-sts`, then click **Execute**.
4. Note the number returned, for example `[5123456789]`. This is the **record ID**.

### 8.2 Create restricted API keys

1. Open **https://eu.api.ovh.com/createToken/**.
2. Enter a name (`github-mta-sts`) and a description, and choose **Unlimited** validity.
3. Under **Rights**, add **exactly** these three lines, replacing the domain and the ID:

   | Method | Path |
   |---|---|
   | GET | `/domain/zone/example.com/record/5123456789` |
   | PUT | `/domain/zone/example.com/record/5123456789` |
   | POST | `/domain/zone/example.com/refresh` |

4. Confirm. OVH shows three keys: **Application Key**, **Application Secret** and **Consumer Key**. Copy them.

> 🔒 **Do not grant more rights.** With these three lines, the keys can change **only** the `_mta-sts` record. If they were stolen, your MX records and the rest of your zone would stay protected.

### 8.3 Store the keys in GitHub

In the repository, go to **Settings** → **Secrets and variables** → **Actions**.

- **Secrets** tab, click **New repository secret** three times:

  | Name | Value |
  |---|---|
  | `OVH_AK` | Application Key |
  | `OVH_AS` | Application Secret |
  | `OVH_CK` | Consumer Key |

- **Variables** tab, click **New repository variable**:

  | Name | Value |
  |---|---|
  | `OVH_MTASTS_RECORD_ID` | the ID noted in 8.1 |

### 8.4 Create the action

Click **Add file → Create new file**. Name the file `.github/workflows/sync-mta-sts.yml`, paste the content below, then **replace `example.com`** on the `DOMAIN:` line. Finally, click **Commit changes**.

```yaml
name: sync-mta-sts

on:
  schedule:
    - cron: "17 6 * * 1"     # every Monday at 06:17 UTC
  workflow_dispatch:
    inputs:
      force_dns:
        description: "Test the DNS id update even without a change"
        type: boolean
        default: false

permissions:
  contents: write
  pages: write

env:
  DOMAIN: example.com

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Fetch and validate Proton's policy
        run: |
          curl -fsS --retry 3 https://mta-sts.protonmail.com/.well-known/mta-sts.txt | tr -d '\r' > proton.txt
          grep -qx 'version: STSv1' proton.txt || { echo "::error::Invalid Proton policy"; exit 1; }
          grep '^mx:' proton.txt | sort > mx.new
          [ -s mx.new ] || { echo "::error::No MX in Proton's policy"; exit 1; }
          if grep -v -E '^mx: [a-z0-9.-]+\.protonmail\.ch$' mx.new; then
            echo "::error::Unexpected MX in Proton's policy: nothing changed"; exit 1
          fi

      - name: Update MX lines (mode and max_age kept)
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

      - name: Commit and republish
        if: steps.upd.outputs.changed == 'true'
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898319+github-actions[bot]@users.noreply.github.com"
          git commit -am "MTA-STS: MX synced with Proton"
          git push
          gh api -X POST "repos/${GITHUB_REPOSITORY}/pages/builds" || true

      - name: Wait for the new policy to be online
        if: steps.upd.outputs.changed == 'true'
        run: |
          for i in $(seq 1 30); do
            if diff -q <(curl -fsS "https://mta-sts.${DOMAIN}/.well-known/mta-sts.txt?n=$i" | tr -d '\r') \
                       <(tr -d '\r' < .well-known/mta-sts.txt) > /dev/null; then
              echo "New policy online"; exit 0
            fi
            sleep 30
          done
          echo "::error::Policy not online after 15 min: DNS id NOT changed"; exit 1

      # ↓ OVH-specific step: adapt it for another DNS provider
      - name: Change the id of the _mta-sts TXT record at OVH
        if: steps.upd.outputs.changed == 'true' || inputs.force_dns
        env:
          OVH_AK: ${{ secrets.OVH_AK }}
          OVH_AS: ${{ secrets.OVH_AS }}
          OVH_CK: ${{ secrets.OVH_CK }}
          RECORD_ID: ${{ vars.OVH_MTASTS_RECORD_ID }}
        run: |
          ovh() {  # usage: ovh METHOD /path [JSON body]
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
            || { echo "::error::Record ${RECORD_ID} is not the _mta-sts TXT record"; exit 1; }

          new="v=STSv1; id=$(date -u +%Y%m%d%H%M%S)"
          ovh PUT  "/domain/zone/${DOMAIN}/record/${RECORD_ID}" "$(jq -cn --arg t "$new" '{target: $t}')"
          ovh POST "/domain/zone/${DOMAIN}/refresh"
          echo "::notice::_mta-sts.${DOMAIN} → ${new}"
```

### 8.5 Test the action

1. In the repository, open the **Actions** tab, then **sync-mta-sts**, and click **Run workflow**.
2. Tick **force_dns**, then click **Run workflow**.
3. After about a minute, a **green tick** should appear.
4. Check that the `id` has changed, on https://internet.nl or with `dig +short TXT _mta-sts.example.com`.

> Changing the `id` without changing the file is harmless: servers simply download the same policy again.

### 8.6 Get notified of failures

Click your **profile picture** → **Settings** → **Notifications**. Under **System → Actions**, tick **Email** and **Only notify for failed workflows**.

> ⏰ **Good to know**: on a public repository, GitHub **pauses scheduled actions after 60 days without activity**. It warns you by email first. Just click **Enable workflow** in the *Actions* tab.

---

## 9. Going further: DNSSEC

Proton publishes **TLSA** (DANE) records for its mail servers. **You do not need to create any TLSA record** in your zone. All you need to do is **turn on DNSSEC** for your domain:

- **At OVH**: Web Cloud → Domain names → `example.com` → **DNSSEC** tab → Enable.
- **Elsewhere**: look for the DNSSEC option in your domain settings.

Servers that use DANE (Postfix, many European providers) will then also check Proton's certificates. MTA-STS (Gmail, Microsoft) and DANE complement each other.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `https://mta-sts.example.com/.well-known/mta-sts.txt` returns a 404 error | `.nojekyll` file missing | Create it (step 2) |
| *Enforce HTTPS* stays greyed out | DNS not propagated yet, or blocking CAA | Wait; check the CNAME and CAA (step 4) |
| Certificate error in the browser | Certificate not issued yet | Remove and re-add the *Custom domain* in Settings → Pages |
| internet.nl does not see MTA-STS | `_mta-sts` TXT missing or mistyped | Check the value `v=STSv1; id=…` (step 5) |
| The action fails at the OVH step (`403`) | Wrong key rights | Recreate the keys with the exact 3 lines (step 8.2) |
| The action fails at the OVH step (`Invalid signature`) | Key pasted incorrectly into Secrets | Re-paste `OVH_AS` and `OVH_CK` (step 8.3) |
| The action no longer runs | Paused after 60 days of inactivity | *Actions* tab → **Enable workflow** |

---

## References

- [RFC 8461 — MTA-STS](https://datatracker.ietf.org/doc/html/rfc8461) · [RFC 8460 — TLS-RPT](https://datatracker.ietf.org/doc/html/rfc8460)
- [privsec.dev — MTA-STS with a custom domain on Proton Mail](https://privsec.dev/posts/knowledge/setting-up-mta-sts-with-a-custom-domain-on-proton-mail/)
- [GitHub Docs — Securing GitHub Pages with HTTPS](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
- [OVHcloud — First steps with the API](https://docs.ovhcloud.com/en/guides/manage-and-operate/api/first-steps)
- [internet.nl — mail test](https://internet.nl/test-mail/)

[🇫🇷 Français](README.md) · 🇬🇧 **English**
