# UnitySVC Whoami Recurrent

Service data for a test recurrent service on the [UnitySVC](https://unitysvc.com) platform.

## Overview

This repo contains the provider, offering, and listing data for the **Whoami Recurrent Test** service. It demonstrates the recurrence scheduling feature — customers enroll with `url` and `port` parameters, optionally configure a recurring schedule, and the platform periodically POSTs those parameters to the upstream whoami echo service.

**Upstream:** `https://whoami.svcpass.com` — returns request details for any incoming request.

**Key features:**

- Recurrent execution with interval or cron scheduling
- Parameters: `url` (string) and `port` (integer) sent as request body
- Body transformer renders enrollment parameters via Jinja2 at enrollment time
- Whoami upstream echoes back the request for verification

## Data Structure

```
data/
└── whoami-test/
    ├── provider.json                     # Provider: Whoami Test
    └── services/
        └── whoami-recurrent/
            ├── offering.json             # Offering: upstream config
            ├── listing-default.json      # Listing: service_options, user access, parameters
            └── docs/
                ├── code_example.sh.j2    # cURL example
                └── code_example.py.j2    # Python example
```

### Service Options (Recurrence)

Configured in `listing-default.json`:

| Option | Value | Description |
|--------|-------|-------------|
| `recurrence_enabled` | `true` | Enables recurrence scheduling for enrollments |
| `recurrence_min_interval_seconds` | `60` | Minimum interval: 1 minute |
| `recurrence_max_interval_seconds` | `86400` | Maximum interval: 1 day |
| `recurrence_allow_cron` | `true` | Cron expressions allowed |

### Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `url` | string | Yes | `https://example.com` | Target URL in the payload |
| `port` | integer | Yes | `443` | Target port in the payload |

### Body Transformer

The listing's `user_access_interfaces` uses a Jinja2 body transformer:

```json
"request_transformer": {
  "set_body": "{{ enrollment.parameters | tojson }}"
}
```

At enrollment time, this renders to the actual parameter values (e.g. `{"url": "https://example.com", "port": 443}`), which are POSTed to the upstream on each recurrence execution.

## Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Commands

```bash
# Validate data files
usvc data validate

# Format data files
usvc data format

# List local services
usvc data list

# Upload to UnitySVC
export SERVICE_BASE_URL="https://backend.unitysvc.com/v1"
export UNITYSVC_API_KEY="<api-key>"
usvc data upload
```

## Architecture

```
Recurrence Scheduler (Celery Beat)
    │
    ▼
execute_recurrent_service(enrollment_id)
    │
    ▼
UnitySVC Backend  ──────►  whoami.svcpass.com
    (resolve upstream,           │
     apply body transformer,     └──► Returns request echo
     track usage)
```

- **Celery Beat** triggers the task on the configured schedule (interval or cron)
- **Backend** resolves the enrollment's upstream config, renders the body from parameters, and POSTs to the upstream
- **Upstream** (`whoami.svcpass.com`) echoes back the request details
