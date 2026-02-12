# Fly.io Maintenance Guide (jake-uptime-kuma)

This guide is a practical SOP for maintaining Uptime Kuma on Fly.io with minimal manual work.

## 1) One-time setup

### 1.1 Required local tools

- `flyctl`
- `git`

### 1.2 Ensure persistent volume exists

```bash
fly volumes list --app jake-uptime-kuma
```

Expected volume name: `uptime_data` (region: `nrt`), mounted to `/app/data`.

If missing:

```bash
fly volumes create uptime_data --app jake-uptime-kuma --region nrt --size 1
```

### 1.3 Enable GitHub auto deploy

This repository includes `.github/workflows/fly-deploy.yml`.

Add repository secret:

- Name: `FLY_API_TOKEN`
- Value: output of:

```bash
fly tokens create deploy -x 999999h
```

## 2) Daily operations

### 2.1 Check app health

```bash
fly status --app jake-uptime-kuma
fly logs --app jake-uptime-kuma --no-tail
```

### 2.2 Restart app

```bash
fly machine restart $(fly machine list --app jake-uptime-kuma --json | jq -r '.[0].id') --app jake-uptime-kuma
```

If `jq` is unavailable, restart from Fly dashboard (Machine -> Restart).

### 2.3 Verify version/image

```bash
fly image show --app jake-uptime-kuma
```

## 3) Update and release workflow (recommended)

1. Edit `fly.toml` or app code in a branch.
2. Open PR and review.
3. Merge to `master`.
4. GitHub Actions deploys automatically to Fly.io.

For manual deploy:

```bash
fly deploy --app jake-uptime-kuma
```

## 4) Rollback

### 4.1 Fast rollback from previous release

```bash
fly releases --app jake-uptime-kuma
fly releases rollback <VERSION> --app jake-uptime-kuma
```

### 4.2 Validate after rollback

```bash
fly status --app jake-uptime-kuma
fly logs --app jake-uptime-kuma --no-tail
```

## 5) Data backup recommendations

Uptime Kuma stores runtime data in `/app/data` (SQLite by default). Keep regular backups.

Low-effort approach:

- In Uptime Kuma web UI, export backup before major changes.
- Keep your Fly volume healthy and avoid deleting it.

More robust approach:

- Schedule periodic backup of `/app/data` using your own script/runner.
- Store encrypted backup artifacts outside Fly (e.g. object storage).

## 6) Common incidents

### 6.1 App not starting

Check:

- `fly logs --app jake-uptime-kuma`
- Volume exists and mounted as `uptime_data -> /app/data`
- `internal_port = 3001` in `fly.toml`

### 6.2 Slow or OOM behavior

You currently use:

```toml
[vm]
  memory = "1gb"
```

Scale up if needed:

```bash
fly scale memory 2048 --app jake-uptime-kuma
```

### 6.3 Deploy succeeded but site unavailable

Check:

- `fly status --app jake-uptime-kuma`
- Machine health checks in dashboard
- App URL and DNS settings

## 7) Minimum monthly checklist

- Review app logs for errors.
- Review monitor notification failures in Uptime Kuma UI.
- Confirm deploy pipeline still passes.
- Verify rollback command and release list are available.
