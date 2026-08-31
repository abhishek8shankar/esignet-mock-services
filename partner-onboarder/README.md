# Partner Onboarder

## Overview
Onboards a Mock Relying Party as an OIDC partner by exchanging certificates with eSignet, using the [partner-onboarder Helm chart](https://github.com/mosip/mosip-onboarding). Runs as a Helm-managed Kubernetes Job.

## Prerequisites
* Access to a cluster with `esignet` deployed (default namespace: `esignet`, override with `$MOCK_RP_NS`).
* A `keycloak` namespace containing:
    * ConfigMap `keycloak-env-vars`
    * Secret `keycloak`
    * Secret `keycloak-client-secrets`

  These are copied into the target namespace automatically during install.
* Either:
    * S3 bucket credentials (host, region, bucket, access key, secret key), **or**
    * An NFS server + path

  for storing the onboarding HTML reports.
* `values.yaml` configured under `onboarding.propertiesOverride.mock-rp-oidc` — at minimum set:
    * `LOGO_URI`, `REDIRECT_URIS` for your relying party
    * `URL`, `AUTHMANAGER_URL`, `PMS_URL`, `KEYCLOAK_URL`, `EXTERNAL_URL` for your environment
    * `KEYCLOAK_ADMIN_USERNAME` / `KEYCLOAK_ADMIN_PASSWORD`
    * `KEYCLOAK_CLIENT_SECRET`
    * `mosip_pms_client_secret`
    * `MOSIP_ID` — set to `"true"` only if the MOSIP ID plugin is enabled on your eSignet deployment.

## Install
Run:

    ./install.sh [kubeconfig]

`kubeconfig` is optional — if provided, it's exported as `KUBECONFIG` before running.

The script will prompt you interactively for:
1. **Public domain & valid SSL?** — answer `n` if you don't have one; this sets `ENABLE_INSECURE=true` in the onboarding configmap.
2. **Update existing live deployment?** — answer `y` to patch the mock relying party's private-key secret, restart `$MOCK_RELYING_PARTY_SERVICE_NAME`, and update `$MOCK_RELYING_PARTY_UI_NAME`'s `CLIENT_ID`. Leave blank/`N` for a one-off or test onboard that shouldn't touch anything already running.
3. **Is `values.yaml` configured correctly?** — confirms you've completed the Prerequisites step above.
4. **S3 or NFS for reports?** — provide S3 credentials, or fall back to an NFS mount if you decline S3.
5. **Is eSignet deployed with default plugins? / MOSIP ID plugin?** — determines the `mosipid` Helm value.

The script then:
* Creates the `esignet` namespace if it doesn't exist.
* Disables Istio injection on that namespace.
* Copies the required Keycloak configmap/secrets from the `keycloak` namespace.
* Installs the `esignet-mock-rp-onboarder` Helm release using `values.yaml` plus the values collected above, and waits for the Job to complete.
* Restarts `$MOCK_RELYING_PARTY_SERVICE_NAME` if you opted to sync the live deployment.
* Deletes the temporary `mock-rp-oidc` properties configmap created for the run.

### Environment variable overrides
| Variable | Default |
|---|---|
| `MOCK_RP_NS` | `esignet` |
| `MOCK_RELYING_PARTY_SERVICE_NAME` | `mock-relying-party-service` |
| `MOCK_RELYING_PARTY_UI_NAME` | `mock-relying-party-ui` |

## Troubleshooting
* Once the onboarder Job completes, a detailed HTML report is generated and stored in the configured S3 bucket / NFS directory. Check it to confirm the partner onboarded successfully.

### Commonly found issues

1. **KER-ATH-401: Authentication Failed**
   Resolution: Provide the correct secret key for `mosip-deployment-client`.

2. **KER-KMS-021: The PARTNER Certificate validity is less than required minimum validity**
   Resolution: Check with your admin about adding a grace period in configuration.

3. **Upload of certificate will not be allowed to update other domain certificate**
   Resolution: Expected when uploading the `ida-cred` certificate a second time — it should only run once, and this error can be ignored if the cert is already present.