# OIDC Debugger

## 1. Ziel

Einrichten von [OIDC Debugger](`https://oidcdebugger.com/`).
Diese Seite kann zur einfachen überprüfung von OIDC-Anfragen benutzt werden.

## 2. Einrichten

### 2.1 Keycloak

1. Erstellen eines Clients, hier `podc-debugger`
2. Client Authentification: `On`
3. Authentification Flow: `Standart Flow` `Implicit Flow` `Direct Access Grants`
4. Valid Redirect URLs: `https://oidcdebugger.com/debug`
5. Web Origins: `https://oidcdebugger.com`
6. Client Secret: kopieren, wird für den Token-exchange benötigt

### 2.2 OIDC Debugger

1. Authorize URI: `https://sso.pngrtz.com/realms/RBS-Ulm/protocol/openid-connect/auth`
2. Redirect URI: `https://oidcdebugger.com/debug`
3. Client ID: `oidc-debugger`
4. Scope: `openid profile email offline_access`

## 3. Benutzung

Es gibt verschiedene Möglichkeiten, eine OIDC-Anfrage zu stellen, die je nach auswahl eine unterschiedliche antwort liefern.

1. **code**

Hier erhält man als Antwort einen hash-code, der durch die Benutzung des folgenden Curl-befehls zum Erhalten eines gültigen Access tokens führt.
```bash
❯ curl -X POST https://sso.pngrtz.com/realms/RBS-Ulm/protocol/openid-connect/token \
      -H "Content-Type: application/x-www-form-urlencoded" \
      -d "grant_type=authorization_code" \
      -d "code=7854623e-2fa5-44bf-b2ac-146a2816b577.594dea93-7ed5-419a-96b6-75f6931ed3b4.d2a80373-991d-403c-953d-4bfa002b984c" \
      -d "client_id=oidc-debugger" \
      -d "client_secret=zpXckxCtqfeV58I..." \
      -d "redirect_uri=https://oidcdebugger.com/debug"
```

Danach kann man mit Hilfe eines weitern Befehls die Antwort decodieren.

```bash
❯ echo "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICI2TDRpMHBHbjUzdjdkOWJwTFFjRzdvOVZpZ29wTThYUTNoLWQwdDMzcV9jIn0.
eyJleHAiOjE3Njc5NTY3MDAsImlhdCI6MTc2Nzk1NjQwMCwiYXV0aF90aW1lIjoxNzY3OTU0NzkyLCJqdGkiOiI5N2E2ZmFhOC0xNzgxLTRjNTctYjY5OS1k
NGZiYWNjOTI2ODEiLCJpc3MiOiJodHRwczovL3Nzby5wbmdydHouY29tL3JlYWxtcy9SQlMtVWxtIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjI1MWY5ZDFm
LWNkNTgtNDQwOC05NzY4LWNiYWQzMTY5NGQ0YiIsInR5cCI6IkJlYXJlciIsImF6cCI6Im9pZGMtZGVidWdnZXIiLCJzaWQiOiI1OTRkZWE5My03ZWQ1LTQx
OWEtOTZiNi03NWY2OTMxZWQzYjQiLCJhY3IiOiIwIiwiYWxsb3dlZC1vcmlnaW5zIjpbImh0dHBzOi8vb2lkY2RlYnVnZ2VyLmNvbSJdLCJyZWFsbV9hY2Nl
c3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiIsImRlZmF1bHQtcm9sZXMtcmJzLXVsbSJdfSwicmVzb3VyY2VfYWNj
ZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3Bl
Ijoib3BlbmlkIG9mZmxpbmVfYWNjZXNzIHByb2ZpbGUgZW1haWwiLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsIm5hbWUiOiJsa3MgbGtzIiwicHJlZmVycmVk
X3VzZXJuYW1lIjoiNjk3NDU5NDAwNTc4ODI2MjUzIiwiZ2l2ZW5fbmFtZSI6ImxrcyIsImZhbWlseV9uYW1lIjoibGtzIiwiZW1haWwiOiJ0ZXN0QGdtYWls
LmNvbSJ9.qbVNTn8SgeYRYAYjAoXAU67JNyWM_ZQRg5SUKCpX0fiwdwpZmYsxGoPOMyEKl6o2w_V_jB02r6F6H5DTrBEfAwdQzOkgZvEX6MjTTt9DzFwQaCQ
lN09UTYl-pnwNkCBStsOYVHM2Y1dXbIzAB3nWtxczjcF7nGo_hw11GC49flhg-GyRdSoXYO14L_EZ94rAp69JPsaNon9SaGoa-5REg1jKlQ3XilzFLdM5nTP
qChrLcwD1kVnD_mryCtGkzOiHEMst17AjdwU8LQOkozjMBvSoJK5bHQON4zNSSMyrRI2r3UaH20RxVkeLK5UHbZH9x-yCAWQrJ0VHFh5QMdP8aw" 
| cut -d '.' -f2 | base64 --decode
```

2. **token/id_token**

Hier erhält man als Antwort sowohl ein Access token, als auch das bereits decodierte Access token. Das dies möglich ist, muss in Keycloak der Implizite Flow für den Client eingestellt sein.

## 3. Beispile einer gültigen Antwort

Dies ist ein Beispiel eines decodierten Access tokens.

```json
{
   "exp": 1768157658,
   "iat": 1768156758,
   "auth_time": 1768155842,
   "jti": "064486df-ae6c-41fe-9263-5030a3cbd876",
   "iss": "https://sso.pngrtz.com/realms/RBS-Ulm",
   "aud": "account",
   "sub": "99e616ce-8036-4ca8-925b-b6c425a47179",
   "typ": "Bearer",
   "azp": "oidc-debugger",
   "sid": "1b552e35-73fc-4ec9-b69b-6c4c892734d3",
   "acr": "0",
   "allowed-origins": [
      "https://oidcdebugger.com"
   ],
   "realm_access": {
      "roles": [
         "offline_access",
         "uma_authorization",
         "default-roles-rbs-ulm"
      ]
   },
   "resource_access": {
      "account": {
         "roles": [
            "manage-account",
            "manage-account-links",
            "view-profile"
         ]
      }
   },
   "scope": "openid offline_access profile email",
   "email_verified": false,
   "name": "123 123",
   "preferred_username": "derlux01",
   "given_name": "123",
   "family_name": "123",
   "email": "derlux01@gmx.de"
}
```
