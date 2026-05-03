# pds-pro-ar — usage

PDS-Pro is the SGC web admin UX that wraps `goat pds admin` with operator
OAuth. One PDS-Pro instance fronts multiple PDS instances. The Go source
lives at `P3X-118/PDS-Pro`; this role is its Ansible deployment.

## On-disk layout

```
/sgc/sgc-pds-pro/
├── etc/
│   ├── config.yaml         # rendered from defaults + group_vars
│   └── secrets/            # OAuth client secrets, per-PDS admin passwords (mode 0600)
└── state/
    ├── session-secret      # cookie-store HMAC secret (derived from sgc_pgsk)
    └── audit.db            # SQLite audit log
```

The container mounts `etc/config.yaml` read-only to `/etc/pds-pro/config.yaml`,
`secrets/` read-only to `/etc/pds-pro/secrets/`, and `state/` writable to
`/etc/pds-pro/state/`.

## Required vars

```yaml
pds_pro_enabled: true
pds_pro_hostname: "admin.pds.example.com"

# At least one OAuth provider must be enabled.
pds_pro_oauth_okta_enabled: true
pds_pro_oauth_okta_org_url: "https://example.okta.com"
pds_pro_oauth_okta_client_id: "0oaXXXXXXXXXXXXXX"
pds_pro_oauth_okta_client_secret: "{{ vault_pds_pro_okta_client_secret }}"

pds_pro_allowlist:
  - email_domain: "example.com"
    roles: ["super-admin"]

pds_pro_instances:
  - name: "alpha"
    pds_host: "https://pds-alpha.example.com"
    # Shared-salt example (default tier). For non-shared, replace
    # `pds.admin` with `pds.admin.alpha` on BOTH sides — pds-ar's
    # pds_admin_password_salt and this admin_password.
    admin_password: >-
      {{ '%s' | format(sgc_pgsk) | password_hash('sha512', 'pds.admin', rounds=655555) | to_uuid }}
```

## Notes on secrets

- **OAuth client secrets** are externally minted by each provider — they
  cannot be derived from `sgc_pgsk`. Store them in an Ansible vault file
  and reference them from group_vars.
- **PDS admin passwords** must match the value the corresponding `pds-ar`
  role derives for the same instance. Two supported tiers:
  - *Shared* (default, lower cost): one salt `pds.admin` for the entire
    fleet. The single-PDS-per-host case usually wants this.
  - *Non-shared* (higher cost): per-instance salt `pds.admin.<instance>`.
    Required when you want a compromised admin credential to blast-radius
    only one instance, or when running multiple PDS instances under a
    single `sgc_pgsk`. Set `pds_admin_password_salt: "pds.admin.<inst>"`
    on the `pds-ar` deployment AND use the same salt in this role's
    `pds_pro_instances[].admin_password` derivation.
- **Session secret** is derived from `sgc_pgsk` so it stays stable across
  restarts and reinstalls without being checked in. Rotating it logs all
  operators out.

## Multi-provider example

See `defaults/main.yml` for all `pds_pro_oauth_<provider>_*` variables.
Twitter (X) uses goth's twitterv2 provider, which is OAuth 1.0a — no
configurable scopes.
