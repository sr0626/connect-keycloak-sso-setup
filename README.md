# Amazon Connect SAML SSO — Direct Keycloak → AWS Federation (POC Runbook)

A reproducible runbook to stand up SAML SSO into Amazon Connect that **deep-links a
user straight into the agent workspace**, using a local **Keycloak** IdP federating
**directly** to AWS. **No Lambda, no API Gateway, no custom code.**

## What you get

Open one URL → log in at Keycloak → land directly in the Amazon Connect **agent
workspace** (`/agent-app-v2/`), as a federated AWS user. The "which page to land on"
is a single config field in Keycloak (the SAML RelayState).

---

## Configuration — fill these in

Every command and value below uses these placeholders. Set your own values; the right
column shows the kind of value expected.

| Placeholder | Meaning | Example |
| ----------- | ------- | ------- |
| `<AWS_ACCOUNT_ID>` | AWS account ID | `123456789012` |
| `<AWS_REGION>` | AWS region of the Connect instance | `us-west-2` |
| `<CONNECT_INSTANCE_ID>` | Amazon Connect instance ID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `<CONNECT_DOMAIN>` | Connect access-URL domain | `<alias>.my.connect.aws` |
| `<IDP_HOST>` | Keycloak hostname (pinned in `/etc/hosts`) | `keycloak-sso.local` |
| `<KC_REALM>` | Keycloak realm name | `connect-poc` |
| `<SAML_PROVIDER>` | IAM SAML identity provider name | `KeycloakConnect` |
| `<FED_ROLE>` | IAM federation role name | `KeycloakConnectRole` |
| `<AGENT_EMAIL>` | test agent user (= Connect username) | `testagent@example.com` |
| `<ADMIN_EMAIL>` | test admin user (= Connect username) | `testadmin@example.com` |
| `<TEST_PASSWORD>` | password for the test users | `ChangeMe123!` |

For the CLI steps, export them once so the commands are copy-paste runnable:

```bash
export AWS_ACCOUNT_ID=123456789012
export AWS_REGION=us-west-2
export CONNECT_INSTANCE_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
export IDP_HOST=keycloak-sso.local
export KC_REALM=connect-poc
export SAML_PROVIDER=KeycloakConnect
export FED_ROLE=KeycloakConnectRole
export AGENT_EMAIL=testagent@example.com
export ADMIN_EMAIL=testadmin@example.com
export TEST_PASSWORD='ChangeMe123!'
```

---

## Assumptions / Prerequisites

These are taken as **already done** before this runbook (they live in other projects /
the AWS console):

1. **Amazon Connect instance with SAML identity management.**
   - `IdentityManagementType = SAML`
     (verify: `aws connect describe-instance --instance-id $CONNECT_INSTANCE_ID --region $AWS_REGION`)
   - With SAML, users cannot log in with a local password — every login must come
     through this federation flow.
2. **Two Connect users exist**, username = the test email, each with a matching
   security profile:
   | Connect username (= email) | Security profile |
   | -------------------------- | ---------------- |
   | `<AGENT_EMAIL>`            | `Agent`          |
   | `<ADMIN_EMAIL>`           | `Admin`          |
   The Connect **username must equal the email** Keycloak sends (the `RoleSessionName`).
3. **Tooling on the local machine:** Docker, AWS CLI (configured, default profile with
   IAM + Connect permissions), and admin rights to edit `/etc/hosts`.

---

## Architecture

```
Browser → http://<IDP_HOST>:8080/realms/<KC_REALM>/protocol/saml/clients/connect
  │  Keycloak: "which client has IDP-init name 'connect'?" → client  urn:amazon:webservices
  ▼
Keycloak issues a SIGNED SAML assertion:
   • Audience       = urn:amazon:webservices
   • Role           = <FED_ROLE> , <SAML_PROVIDER>  (the IAM SAML provider)
   • RoleSessionName= <AGENT_EMAIL>
   • RelayState     = …/connect/federate/<CONNECT_INSTANCE_ID>?destination=/agent-app-v2/   ← deep-link
   │  POSTed to  https://signin.aws.amazon.com/saml
  ▼
AWS sign-in:  sts:AssumeRoleWithSAML  → temp session as <FED_ROLE>
   │  then redirects the browser to the RelayState URL
  ▼
…/connect/federate/<CONNECT_INSTANCE_ID>?destination=/agent-app-v2/
   │  AWS Connect: connect:GetFederationToken → logs the user into the instance,
   │  honors ?destination=
  ▼
Lands in  https://<CONNECT_DOMAIN>/agent-app-v2/   (agent workspace)
```

---

## Setup

### Step 1 — Hostname (`<IDP_HOST>` instead of `localhost`)

The SAML **issuer** Keycloak puts in the assertion is derived from the host you browse
to. AWS only trusts the issuer registered in its SAML provider
(`http://<IDP_HOST>:8080/realms/<KC_REALM>`). If you browse via `localhost`, the issuer
becomes `http://localhost:...` and **AWS rejects the assertion**. So we pin a stable
hostname.

Add this line to `/etc/hosts` (needs sudo):

```bash
echo "127.0.0.1   $IDP_HOST" | sudo tee -a /etc/hosts
# verify:
dscacheutil -q host -a name "$IDP_HOST"     # → ip_address: 127.0.0.1
```

> macOS quirk: `.local` names go through multicast DNS first, so the **first** lookup
> per session pauses ~5s before falling back to `/etc/hosts`. Harmless — the flow still
> works, it's just a one-time delay on each redirect.

### Step 2 — Launch Keycloak (the IdP)

```bash
docker run -d --name keycloak-poc \
  -p 8080:8080 \
  -v keycloak_data:/opt/keycloak/data \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```

- `start-dev` uses the embedded H2 database stored in the `keycloak_data` volume.
- **Always use `:latest`** (or ≥ the version that created the volume). If you reuse a
  volume written by a newer Keycloak, an older image fails to open it with a misleading
  `Wrong user name or password [28000-230]` / `Failed to obtain JDBC connection`. The
  data is fine — it's a version-downgrade error.
- Admin console: `http://localhost:8080` (login `admin` / `admin`).

If the `keycloak_data` volume already contains the `<KC_REALM>` realm, **you're done —
skip to [Test](#test)**. To build the realm from scratch, continue.

#### Day-to-day: start / stop the container

Once created, reuse the same container with `start`/`stop` (don't re-run `docker run`).
Realm data persists in the `keycloak_data` volume either way.

```bash
docker stop keycloak-poc                       # stop (frees port 8080)
docker start keycloak-poc                       # bring it back up (~10–20s to boot)
docker ps -a --filter name=keycloak-poc         # status
docker logs -f keycloak-poc                      # follow logs (confirm boot)
```

> Only if the container was **removed** (`docker rm`) do you need the full `docker run …`
> above again to recreate it.

### Step 3 — Realm + users (only if starting from an empty Keycloak)

1. Admin console → create realm **`<KC_REALM>`**.
2. Create two users; set **Email** (used as `RoleSessionName`) and a password
   (`<AGENT_EMAIL>` / `<ADMIN_EMAIL>`, both with `<TEST_PASSWORD>`).

   Fast path via admin API (set/reset a known password):
   ```bash
   TOKEN=$(curl -s -d client_id=admin-cli -d username=admin -d password=admin \
     -d grant_type=password \
     http://localhost:8080/realms/master/protocol/openid-connect/token | \
     python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')
   for U in "$AGENT_EMAIL" "$ADMIN_EMAIL"; do
     UID=$(curl -s -H "Authorization: Bearer $TOKEN" \
       "http://localhost:8080/admin/realms/$KC_REALM/users?username=$U&exact=true" | \
       python3 -c 'import sys,json;print(json.load(sys.stdin)[0]["id"])')
     curl -s -X PUT -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
       "http://localhost:8080/admin/realms/$KC_REALM/users/$UID/reset-password" \
       -d "{\"type\":\"password\",\"value\":\"$TEST_PASSWORD\",\"temporary\":false}"
   done
   ```

### Step 4 — The SAML client (`urn:amazon:webservices`)

Create **one** SAML client. The Client ID **must** be `urn:amazon:webservices` (that is
the audience AWS requires).

Admin console → Clients → Create client → **SAML**:

| Setting | Value |
| ------- | ----- |
| Client ID | `urn:amazon:webservices` |
| Name | `AWS Connect direct federation` |
| Valid redirect URIs | `https://signin.aws.amazon.com/saml` |
| Master SAML Processing URL | `https://signin.aws.amazon.com/saml` |
| Name ID format | `email` |
| Force Name ID Format | **On** |
| Force POST Binding | **On** |
| Sign Documents | **On** |
| Sign Assertions | **On** |
| Client Signature Required | **Off** |
| **IDP-Initiated SSO URL name** | `connect` |
| **IDP Initiated SSO Relay State** | `https://<AWS_REGION>.console.aws.amazon.com/connect/federate/<CONNECT_INSTANCE_ID>?destination=/agent-app-v2/` |

> **Two fields do the magic:**
> - *IDP-Initiated SSO URL name* = `connect` is the **routing key** — it's the last path
>   segment of the launch URL. Keep it **unique**; if two clients share it, the launch
>   URL resolves to the wrong one (this bit us).
> - *IDP Initiated SSO Relay State* — the `?destination=/agent-app-v2/` on the end is the
>   entire "go to the agent workspace" instruction. Change that path to change where
>   users land. (RelayState is static per launch link, so per-user/per-profile routing is
>   **not** possible in this model — one destination for everyone.)

> **NameID:** with *Name ID format = email* + *Force Name ID Format = On*, the assertion's
> Subject `<NameID>` is the user's email (no separate NameID mapper needed). That's the SAML
> Subject identity — distinct from the `RoleSessionName` attribute below, which is what AWS
> uses as the session name and which must match the **Connect username**. In this POC both
> happen to be the email.

Then add **three protocol mappers** (Clients → `urn:amazon:webservices` → Client scopes →
the `…-dedicated` scope → Add mapper → By configuration):

| Mapper name | Type | Key config |
| ----------- | ---- | ---------- |
| `Role` | **Hardcoded attribute** | Attribute name `https://aws.amazon.com/SAML/Attributes/Role`, NameFormat `Basic`, value `arn:aws:iam::<AWS_ACCOUNT_ID>:role/<FED_ROLE>,arn:aws:iam::<AWS_ACCOUNT_ID>:saml-provider/<SAML_PROVIDER>` |
| `RoleSessionName` | **User Property** | Property `email`, SAML attribute `https://aws.amazon.com/SAML/Attributes/RoleSessionName`, NameFormat `Basic` |
| `email` | **User Property** | Property `email`, SAML attribute `email`, NameFormat `Basic` |

> Gotcha: Keycloak **merges** client attributes on API update — to *clear* an attribute
> you must PUT it as `""`, not omit it.

### Step 5 — Register Keycloak with AWS (IAM SAML provider + role)

Only needed once per AWS account. To create from scratch:

```bash
# 5a. Export the realm's SAML metadata (contains the signing cert AWS will trust)
curl -s "http://$IDP_HOST:8080/realms/$KC_REALM/protocol/saml/descriptor" \
  -o /tmp/keycloak-metadata.xml

# 5b. Create the IAM SAML identity provider
aws iam create-saml-provider \
  --name "$SAML_PROVIDER" \
  --saml-metadata-document file:///tmp/keycloak-metadata.xml

# 5c. Create the federation role (trust = AssumeRoleWithSAML from that provider)
cat > /tmp/trust.json <<JSON
{ "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::$AWS_ACCOUNT_ID:saml-provider/$SAML_PROVIDER" },
    "Action": "sts:AssumeRoleWithSAML",
    "Condition": { "StringEquals": { "SAML:aud": "https://signin.aws.amazon.com/saml" } }
  }] }
JSON
aws iam create-role --role-name "$FED_ROLE" \
  --assume-role-policy-document file:///tmp/trust.json

# 5d. Permission to federate into the Connect instance
cat > /tmp/connect-policy.json <<JSON
{ "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "connect:GetFederationToken",
    "Resource": "arn:aws:connect:$AWS_REGION:$AWS_ACCOUNT_ID:instance/$CONNECT_INSTANCE_ID/user/*"
  }] }
JSON
aws iam put-role-policy --role-name "$FED_ROLE" \
  --policy-name ConnectFederatePolicy \
  --policy-document file:///tmp/connect-policy.json
aws iam attach-role-policy --role-name "$FED_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/AmazonConnect_FullAccess
```

> The signing cert in the SAML provider (5a/5b) **must match** the realm's current signing
> cert. If you ever regenerate the realm keys, re-run 5a and
> `aws iam update-saml-provider`. Verify they match:
> ```bash
> aws iam get-saml-provider --saml-provider-arn arn:aws:iam::$AWS_ACCOUNT_ID:saml-provider/$SAML_PROVIDER \
>   --query SAMLMetadataDocument --output text | grep -o 'X509Certificate>[^<]*' | head -1
> curl -s "http://$IDP_HOST:8080/realms/$KC_REALM/protocol/saml/descriptor" \
>   | grep -o 'X509Certificate>[^<]*' | head -1
> ```

---

## Test

1. **Use a fresh incognito/private window** (avoids an existing AWS console session
   short-circuiting the RelayState redirect).
2. Open — note the **`<IDP_HOST>`** hostname, not `localhost`:
   ```
   http://<IDP_HOST>:8080/realms/<KC_REALM>/protocol/saml/clients/connect
   ```
3. Log in as `<AGENT_EMAIL>` / `<TEST_PASSWORD>`.
4. **Expected:** you land directly in `https://<CONNECT_DOMAIN>/agent-app-v2/`
   (the agent workspace) — not the AWS console.

If you land on the **AWS console home** instead, the RelayState didn't reach AWS — check
the client's *IDP Initiated SSO Relay State* and that the `connect` URL name is unique.

---

## Verify the SAML flow (SAML-tracer)

When something is off, the fastest way to see *exactly* what Keycloak sent and what AWS
received is **SAML-tracer**, a browser extension that decodes the base64 SAML messages
inline.

### Install

- **Chrome / Edge / Brave:** Chrome Web Store → search **"SAML-tracer"** → Add to
  Chrome. (`https://chromewebstore.google.com/` — extension by Aaron St. Clair / the
  classic SAML-tracer port.)
- **Firefox:** addons.mozilla.org → search **"SAML-tracer"** → Add to Firefox
  (`https://addons.mozilla.org/firefox/addon/saml-tracer/`).

### Capture the flow

1. Open the extension (toolbar icon) — it opens a capture window. Leave it open.
2. In the **same** browser, do the [Test](#test) login (use a fresh private window so
   you capture a clean flow).
3. In SAML-tracer, requests that carry a SAML message are flagged with an orange **`SAML`**
   marker. Click one, then the **SAML** tab to see the decoded XML.

You should see (in order):
- a **POST to `https://signin.aws.amazon.com/saml`** carrying a `SAMLResponse` (this is
  the assertion Keycloak minted) — **this is the important one**, and
- a **302 from `signin.aws.amazon.com`** redirecting to
  `…/connect/federate/<CONNECT_INSTANCE_ID>?destination=/agent-app-v2/` (the RelayState
  being honored).

### What a good assertion looks like — checklist

In the `SAMLResponse` POSTed to `signin.aws.amazon.com/saml`, confirm each of these:

| Element in the XML | Must be |
| ------------------ | ------- |
| `<Response Destination=...>` | `https://signin.aws.amazon.com/saml` |
| `<StatusCode Value=...>` | `urn:oasis:names:tc:SAML:2.0:status:Success` |
| `<Issuer>` | `http://<IDP_HOST>:8080/realms/<KC_REALM>` (the issuer AWS trusts) |
| `<ds:Signature>` present | assertion/response **is signed** (AWS requires it) |
| `<Subject><NameID Format="…emailAddress">` | `<AGENT_EMAIL>` |
| `<SubjectConfirmationData Recipient=...>` | `https://signin.aws.amazon.com/saml` |
| `<Conditions><AudienceRestriction><Audience>` | `urn:amazon:webservices` |
| `<Conditions NotBefore/NotOnOrAfter>` | a current time window (clock not skewed) |
| Attribute `https://aws.amazon.com/SAML/Attributes/Role` | `arn:aws:iam::<AWS_ACCOUNT_ID>:role/<FED_ROLE>,arn:aws:iam::<AWS_ACCOUNT_ID>:saml-provider/<SAML_PROVIDER>` |
| Attribute `https://aws.amazon.com/SAML/Attributes/RoleSessionName` | `<AGENT_EMAIL>` (must equal a **Connect username**) |
| Form field **`RelayState`** (next to `SAMLResponse`) | `https://<AWS_REGION>.console.aws.amazon.com/connect/federate/<CONNECT_INSTANCE_ID>?destination=/agent-app-v2/` |

If all of the above are present and correct, the integration is healthy — AWS will
`AssumeRoleWithSAML` and the RelayState will carry the browser into the agent workspace.

### Red flags (and what they mean)

| Symptom in SAML-tracer | Diagnosis |
| ---------------------- | --------- |
| `RelayState` field empty / missing | Client's *IDP Initiated SSO Relay State* not set, or the `connect` URL name resolved to a different client → lands on AWS console home. |
| `<Audience>` is anything other than `urn:amazon:webservices` | Client ID is wrong → AWS rejects (or you hit the wrong client). |
| `<Issuer>` shows `http://localhost...` | You browsed via `localhost` → issuer won't match the AWS SAML provider; use `<IDP_HOST>`. |
| `Destination` / `Recipient` is **not** `signin.aws.amazon.com/saml` | The assertion is being sent somewhere else (e.g. an old Lambda ACS) → AWS will 400. |
| No `<ds:Signature>` | Sign Assertions/Documents are off → AWS rejects as unsigned. |
| `StatusCode` ≠ `…:Success` | Authentication failed at Keycloak before AWS was even reached. |

### Keycloak server-side logs

For the IdP side, follow the container logs while you log in:

```bash
docker logs -f keycloak-poc
```

You'll see the login/authentication events. For verbose SAML wire-level logging, start
Keycloak with debug on the SAML logger (re-run `docker run …` from Step 2 with
`start-dev --log-level=INFO,org.keycloak.saml:debug,org.keycloak.protocol.saml:debug`).
Most issues, though, are visible directly in the decoded assertion above — SAML-tracer is
usually faster than the logs.

---

## Why no Lambda

An earlier design ran a Lambda "launcher": it received the SAML assertion at its own ACS,
looked up the user's Connect profile, then **re-POSTed the same assertion** to
`https://signin.aws.amazon.com/saml`. AWS rejected it with a **400** — a SAML assertion's
`Recipient`/`Audience` are cryptographically bound to the ACS it was minted for (the
Lambda), not AWS's endpoint, so AWS treats the re-post as invalid.

Federating **directly** (Keycloak → `signin.aws.amazon.com/saml`) avoids the problem
entirely. The trade-off: the Lambda could route per security-profile; direct federation
uses a static RelayState (one destination per launch link).

---

## Teardown / Destroy

```bash
# 1. Stop & remove the Keycloak container (realm data SURVIVES in the volume)
docker rm -f keycloak-poc

# 2. (Optional) delete the realm data permanently
docker volume rm keycloak_data        # ⚠️ irreversible — wipes the <KC_REALM> realm

# 3. Remove the hostname mapping
sudo sed -i '' "/$IDP_HOST/d" /etc/hosts     # macOS

# 4. (Optional) remove the AWS federation resources
aws iam delete-role-policy   --role-name "$FED_ROLE" --policy-name ConnectFederatePolicy
aws iam detach-role-policy   --role-name "$FED_ROLE" --policy-arn arn:aws:iam::aws:policy/AmazonConnect_FullAccess
aws iam delete-role          --role-name "$FED_ROLE"
aws iam delete-saml-provider --saml-provider-arn arn:aws:iam::$AWS_ACCOUNT_ID:saml-provider/$SAML_PROVIDER
```

> The Amazon Connect **instance** and its users are **not** created by this runbook
> (they're an assumption / a separate project), so teardown here does not touch them.

---

## Troubleshooting

| Symptom | Cause / fix |
| ------- | ----------- |
| Keycloak container exits, `Wrong user name or password [28000-230]` | Image older than the `keycloak_data` volume. Use `quay.io/keycloak/keycloak:latest`. |
| Lands on AWS **console home**, not Connect | RelayState empty/not applied. Check the client's *IDP Initiated SSO Relay State*, and that the `connect` URL name is unique to this client. |
| AWS error: invalid/untrusted assertion | Browsed via `localhost` (issuer mismatch) — use `<IDP_HOST>`; or the SAML-provider cert no longer matches the realm cert (re-run Step 5a + `update-saml-provider`). |
| Connect: "user not found" after federation | `RoleSessionName` (the email) doesn't match a Connect **username**, or that Connect user has no security profile. |
| `<IDP_HOST>` slow / times out first hit | macOS `.local` mDNS delay (~5s). Harmless. |
