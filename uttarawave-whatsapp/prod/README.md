# uttarawave-whatsapp

[GOWA](https://github.com/aldinokemal/go-whatsapp-web-multidevice) — a REST API in front of a
WhatsApp number, used to send customer notifications.

> GOWA is **unofficial** (built on the reverse-engineered WhatsApp Web protocol). WhatsApp can
> ban the linked number. Use a dedicated SIM, never the main business number, and avoid bulk sends.

Runs as a single-replica `Recreate` Deployment with a ClusterIP Service on port 3000, exposed at
https://wa.uttarawave.com through a Traefik ingress (protected only by `APP_BASIC_AUTH`).
The WhatsApp session is stored in SQLite on the `uttarawave-whatsapp-storages` PVC, which is kept
even if the chart is uninstalled.

## 1. Create the secret

The chart reads its credentials from a Secret named `uttarawave-whatsapp-env`, created by hand
(it is not in git).

| Key | Required | Purpose |
|---|---|---|
| `APP_BASIC_AUTH` | yes | `user:password` for the API and dashboard. GOWA refuses to start without it. The backend uses the same pair to call GOWA. |
| `WHATSAPP_WEBHOOK_SECRET` | yes | Key GOWA signs incoming-message webhooks with (`X-Hub-Signature-256`). The backend uses the same key to verify that a webhook really came from GOWA. Without it GOWA invents a new random key on every restart and the backend could never verify. |

Paste into bash or zsh. It prompts for the username and password (the password is not echoed)
and generates the webhook secret:

```sh
printf 'GOWA username: '; read -r GOWA_USER
printf 'GOWA password: '; stty -echo; read -r GOWA_PASS; stty echo; echo
kubectl -n uttarawave create secret generic uttarawave-whatsapp-env \
  --from-literal=APP_BASIC_AUTH="${GOWA_USER}:${GOWA_PASS}" \
  --from-literal=WHATSAPP_WEBHOOK_SECRET="$(openssl rand -hex 32)"
unset GOWA_USER GOWA_PASS
```

Read the values back (the backend's config needs them):

```sh
S=$(kubectl -n uttarawave get secret uttarawave-whatsapp-env -o jsonpath='{.data.APP_BASIC_AUTH}' | base64 -d)
echo "username: ${S%%:*}"
echo "password: ${S#*:}"
echo "webhook secret: $(kubectl -n uttarawave get secret uttarawave-whatsapp-env -o jsonpath='{.data.WHATSAPP_WEBHOOK_SECRET}' | base64 -d)"
unset S
```

## 2. Deploy

The secret must exist first, otherwise the pod stays in `CreateContainerConfigError`.

```sh
helm template uttarawave-whatsapp . -f values.yaml -f values_tag.yaml   # preview
```

The PVC uses the cluster's default StorageClass. Set `persistence.storageClassName` in
`values.yaml` to pick another.

## 3. Link the WhatsApp number (one time)

Open <https://wa.uttarawave.com>, log in with the `APP_BASIC_AUTH` credentials, click **Login**, and
scan the QR code from the phone: WhatsApp → Linked devices → Link a device.

Check:

```sh
curl -u '<username>:<password>' https://wa.uttarawave.com/app/status
```

Open WhatsApp on that phone at least every couple of weeks; WhatsApp unlinks devices whose main
phone stays inactive too long, and the QR scan must then be repeated.

## Calling it from the backend

In-cluster URL: `http://uttarawave-whatsapp.uttarawave:3000`

```sh
curl -u '<username>:<password>' -X POST http://uttarawave-whatsapp.uttarawave:3000/send/message \
  -H 'Content-Type: application/json' \
  -d '{"phone":"8801XXXXXXXXX@s.whatsapp.net","message":"Hello"}'
```

## Upgrading

Bump `image.tag` in `values_tag.yaml` to a release from
<https://github.com/aldinokemal/go-whatsapp-web-multidevice/releases>. The session on the PVC
survives upgrades.
