# CalDAV to Mastodon

This Cloudflare Worker automatically posts upcoming events (in a public CalDAV calendar) to a Mastodon account. It runs daily and checks for meetings scheduled for a configurable number of days ahead.

## Features

- automatically checks for upcoming meetings daily (configurable days ahead)
- posts formatted announcements to Mastodon
- serverless function via Cloudflare Workers
- basic web interface manual support for testing

## Setup

### 0. Prerequisites

- Cloudflare account
- Mastodon account and access token with `write:statuses` permission
- A public CalDAV calendar feed that supports the [sabre/dav ICSExportPlugin](https://sabre.io/dav/ics-export-plugin/). This is a specific requirement for this worker to function correctly.

### 1. Get Mastodon API Credentials

1. Mastodon instance ⇢ Settings ⇢ Development ⇢ New Application
2. Permissions:  `write:statuses`
3. Save
4. Copy **Access Token** (keep this secure - don't add it to your worker yet)

### 2. Deploy Cloudflare Worker

#### Deploy to Cloudflare

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/andesco/cloudflare-worker-caldav-to-mastodon)

#### Cloudflare Dashboard

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com) ⇢ Workers & Pages
2. Click "Create Application" ⇢ "Create Worker"
3. Name your worker (e.g., `calendar-mastodon-bot`)
4. Replace the default code with the contents of `index.js`
5. Save and Deploy
6. Set the environtment variables (see below): \
Settings ⇢ Variables

#### Wrangler CLI

```bash
npm install -g wrangler
cd cloudflare-worker-caldav-to-mastodon
wrangler login
wrangler deploy
wrangler secret put MASTODON_ACCESS_TOKEN
```

Add public environment variables to your `wrangler.toml` file:

```toml
[vars]
CALENDAR_EXPORT_URL = "{URL}" # ends in ?export
MASTODON_INSTANCE_URL = "{URL}"
DAYS_AHEAD = "1" # post events occuring __ days ahead (0-15, where 0=today and 1=tomorrow, default: 1)
```

### 3. Security Setup (IMPORTANT)

**⚠️ Set up Cloudflare Access BEFORE adding your Mastodon token:**

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com) → Security → Access → Applications
2. Add application for your Worker domain
3. Configure authentication rules (email, domain, etc.)

**Why this matters:** Without protection, anyone who finds your worker URL can trigger posts to your Mastodon account. Deploy the worker first (without `MASTODON_ACCESS_TOKEN`), test it safely, then secure it before adding your token.

### 4. Configure Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `CALENDAR_EXPORT_URL` | CalDAV calendar subscription URL | `https://social.coop/calendar/?export` |
| `MASTODON_INSTANCE_URL` | Mastodon instance URL | `https://social.coop` |
| `DAYS_AHEAD` | Days ahead to post events for (0-15, where 0=today, 1=tomorrow) | `1` |
| `MASTODON_ACCESS_TOKEN` | access token from Mastodon (add ONLY after securing worker) | `your-private-access-token` |

### 5. Set Up Daily Schedule

The default cron schedule is already configured in `wrangler.toml` to run daily at 5:30 PM UTC:

```toml
[triggers]
crons = ["30 17 * * *"]
```
Modify the cron schedule in `wrangler.toml` before deploying or use the Cloudflare Dashboard.



## Usage

### Automatic Operation

Once deployed and configured, the Worker will:
- run daily at your scheduled time (defaulting to 17:30 UTC);
- check for events happening N days ahead (configurable via `DAYS_AHEAD`, default: 1 day/tomorrow, 0=today); and
- post formatted announcements to Mastodon for each event.

### Manual Testing

**Web Interface**:

Visit your Worker URL in a browser for a simple web interfac:
```
https://{worker}.{subdomain}.workers.dev
```

## API

The worker provides a simple API for fetching events in jCal (JSON) and posting to Mastodon.

#### GET /api/events

```bash
curl "https://{worker}.{subdomain}.workers.dev/api/events?days=14"
```
-   `days`: the number of days within to fetch events (optional, number: 1–24, default: 14)

### POST /trigger

```bash
curl -X POST https://{worker}.{subdomain}.workers.dev/post/day
```
- checks for events occurring N days ahead (based on `DAYS_AHEAD` setting) and posts to Mastodon
- runs automatically via cron schedule

```bash
curl -X POST https://{worker}.{subdomain}.workers.dev/post/tomorrow
```
- legacy alias for `/post/day` (for backward compatibility)

```bash
curl -X POST https://{worker}.{subdomain}.workers.dev/post/next
```
- posts the closest event within the next 14 days

## Cloudflare Access

**Cloudflare Access Protection**:

To protect the web interface, set up Cloudflare Access in your dashboard:
1. Go to Security ⇢ Access ⇢ Applications
2. Add application for your Worker domain
3. Configure authentication rules (email, domain, etc.)

**Accessing Protected Endpoints via CLI**:

If your worker is protected by Cloudflare Access, use `cloudflared` CLI to  authenticate:

```bash
cloudflared access curl https://{worker}.{subdomain}.workers.dev/post/day -X POST
```


## Customization

### Changing Post Format

Modify the `postToMastodon()` function to customize:
- post text and emojis
- visibility settings (`public`, `unlisted`, `private`)
- character limits and truncation

### Filtering Events

Add filters in `checkAndPostDayEvents()` to only post certain events:

```javascript
const tomorrowEvents = events.filter(event => {
  const eventDate = new Date(event.start);
  const isValidDate = eventDate >= tomorrow && eventDate <= tomorrowEnd;
  const isRelevantEvent = event.summary.includes('Public') || event.summary.includes('Community');
  return isValidDate && isRelevantEvent;
});
```

### Time Zone Handling

The Worker uses UTC by default. To handle specific timezones:

```javascript
const tomorrow = new Date();
// Convert to specific timezone
const options = { timeZone: 'America/New_York' };
const localDate = new Date(tomorrow.toLocaleString('en-US', options));
```

## Troubleshooting

## Calendar Data Source

**Important:** This worker is specifically designed to work with a CalDAV server with the [sabre/dav ICSExportPlugin](https://sabre.io/dav/ics-export-plugin/) enabled. This plugin allows the worker to efficiently fetch a specific date range of events in the jCal format, using a URL like this:

```
https://{CALENDAR_EXPORT_URL}&accept=jcal&start={timestamp}&end={timestamp}&expand=1
```

If your calendar URL does not support this plugin, the worker will not be able to fetch events.


## File Structure

```
cloudflare-worker-caldav-to-mastodon/
├── index.js          # Main Worker code
├── README.md         # This documentation
└── wrangler.toml     # Wrangler configuration (optional)
```

## License

This project is [licensed under the MIT License](LICENSE).
