# Security

## Credentials overview

| Credential | What it is | Where it lives |
|---|---|---|
| `SERVICENOW_USERNAME` | ServiceNow user for REST API calls (GitHub → SN) | GitHub repository secret |
| `SERVICENOW_PASSWORD` | Password for the above user | GitHub repository secret |
| `SERVICENOW_URL` | Full API endpoint URL | GitHub repository secret |
| `SERVICENOW_UI_URL` | ServiceNow base URL | GitHub repository secret |
| GitHub PAT | Token for ServiceNow → GitHub dispatches | ServiceNow system property `github.dispatch.config` |

---

## GitHub repository secrets

GitHub Actions uses these to call ServiceNow. They are never exposed in logs.

**Repository Settings → Secrets and variables → Actions → New repository secret**

Add all four secrets listed in the table above. Anyone with write access to the repo can use them in workflows but cannot read the raw values.

---

## GitHub Personal Access Token (Fine-grained)

ServiceNow uses this token to send `repository_dispatch` events to GitHub.

### Generate the token

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**
2. Click **Generate new token**
3. Set **Token name** and an expiry date
4. Under **Repository access** → select **Only select repositories** → pick this repo
5. Under **Permissions → Repository permissions**, set:
   - **Contents** → `Read and write`
   - **Metadata** → `Read-only` (required, added automatically)
6. Click **Generate token** and copy it immediately — it won't be shown again

### Store the token in ServiceNow

Paste the token as the `token` value in the `github.dispatch.config` system property:

```json
{
  "Your Account Display Name": {
    "token": "github_pat_xxxxx",
    "owner": "your-org",
    "repo":  "your-repo"
  }
}
```

See [INSTALLATION.md](INSTALLATION.md) Step 2 for full instructions on this property.

### Rotating the token

When the token expires or is revoked:
1. Generate a new token following the steps above
2. Open `github.dispatch.config` in ServiceNow system properties
3. Replace the `token` value and tick **Ignore cache** before saving
4. No flow or workflow changes are needed

---

## Least-privilege notes

- The PAT only needs **Contents: Read and write** — this is the minimum required to send `repository_dispatch` events
- The ServiceNow user (`SERVICENOW_USERNAME`) should have only the REST API role — no admin access needed
- Scope the PAT to **Only select repositories**, not all repositories
