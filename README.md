# CalDAV to Mastodon

This Cloudflare Worker automatically posts upcoming events (in a public CalDAV calendar) to a Mastodon account. It runs daily and checks for meetings scheduled for a configurable number of days ahead.

## Features

- automatically checks for upcoming events daily
- posts formatted announcements to Mastodon
- serverless function via Cloudflare Workers
- basic web interface manual support for testing

## Setup

### 0. Prerequisites

- Cloudflare account
- Mastodon account and access token with `write:statuses` permission
- A public CalDAV calendar feed that supports the [sabre/dav ICSExportPlugin](https://sabre.io/dav/ics-export-plugin/). This is a specific requirement for this worker to function correctly.

### 1. Deploy Cloudflare Worker

#### Deploy to Cloudflare

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/andesco/cloudflare-worker-caldav-to-mastodon)

#### Cloudflare Dashboard

<nobr>Workers & Pages</nobr> ⇢ Create an application ⇢ [Clone a repository](https://dash.cloudflare.com/?to=/:account/workers-and-pages/create/deploy-to-workers): \
   `http://github.com/andesco/cloudflare-worker-caldav-to-mastodon`

#### Wrangler CLI

Update `wrangler.toml` to set your environment variables:

```toml wrangler.toml
[vars]
CALENDAR_EXPORT_URL = "{URL}"
MASTODON_INSTANCE_URL = "{URL}"
DAYS_AHEAD = "1"
```

```bash
cd cloudflare-worker-caldav-to-mastodon
wrangler login
wrangler deploy
```

### 2. Enable Cloudflare Access

To protect the web interface, set up Cloudflare Access in your dashboard:

1. [Cloudflare Dashboard](https://dash.cloudflare.com) ⇢ Zero Trust ⇢ Access ⇢ Applications
2. Add an application for your Worker and its hostnames.
3. Configure authentication rules (email, domain, etc.)

> [!WARNING]
> Set up Cloudflare Access before saving your Mastodon token to `MASTODON_ACCESS_TOKEN`. Deploy the worker first, verify that the web interface can read your public calendar, secure access, and then add your token.

> [!NOTE]
> If your worker is protected by Cloudflare Access, use `cloudflared` CLI to authenticate:
> ```bash
> cloudflared access curl https://{worker}.{subdomain}.workers.dev/post/day -X POST
> ```

### 3. Get Mastodon Access Token

1. Mastodon instance ⇢ Settings ⇢ Development ⇢ New Application
2. Permissions:  `write:statuses`
3. Save
4. Copy `{your access token}`

### 4. Add Mastodon Access Token

   #### Cloudflare Dashboard

   [Workers & Pages](https://dash.cloudflare.com/?to=/:account/workers-and-pages/) ⇢ `{worker}` ⇢ Settings: <nobr>Variables and Secrets: Add:</nobr>\
      Type: `Secret`\
      Variable name: `MASTODON_ACCESS_TOKEN`\
      Value: `{your access token}`
   
   #### Wrangler CLI
      
   ```bash
   wrangler secret put MASTODON_ACCESS_TOKEN`
   ```

### 5. Modify Schedule
 
   #### Cloudflare Dashboard
   
   [Workers & Pages](https://dash.cloudflare.com/?to=/:account/workers-and-pages/) ⇢ `{worker}` ⇢ Settings: <nobr>Trigger Events: Edit</nobr>
    
   #### Wrangler CLI
   
   The default schedule is set to run daily at 5:30 PM UTC. Modify the cron schedule in `wrangler.toml` and redeploy.
   
   ```toml wrangler.toml
   [triggers]
   crons = ["30 17 * * *"]
   ```
   
   ```bash
   wrangler secret put MASTODON_ACCESS_TOKEN`
   ```
   
## Environment Variables & Secret

| Variable | Description | Example |
|----------|-------------|---------|
| `CALENDAR_EXPORT_URL` | CalDAV calendar subscription URL | `https://social.coop/calendar/?export` |
| `MASTODON_INSTANCE_URL` | Mastodon instance URL | `https://social.coop` |
| `DAYS_AHEAD` | days ahead to post events | `0` &nbsp; `1` &nbsp; `0,1,14` |
| `MASTODON_ACCESS_TOKEN` | access token from Mastodon | `your-private-access-token` |

> [!NOTE]
> `DAYS_AHEAD` <br> `0` posts all events occuring today <br> `1` posts all events occuring tomorrow (default) <br> `0,1,14` posts all events occuring today, tomorrow, and in 14 days


## Usage

### Automatic Operation

Once deployed and configured, the Worker will:
- run daily at your scheduled time (defaulting to 17:30 UTC);
- check for events occuring in `{DAYS_AHEAD} days; and
- post formatted announcements to Mastodon for each event.

### Manual Posting

Visit your Worker in a browser for a simple web interfac:
```
https://{worker}.{subdomain}.workers.dev
```

## API

The worker provides a simple API for fetching events in jCal (JSON) and posting to Mastodon.

#### GET /api/events

```bash
curl "https://{worker}.{subdomain}.workers.dev/api/events?days={days}"
```
-   fetch events within specified `{days}`

#### POST /trigger

```bash
curl -X POST https://{worker}.{subdomain}.workers.dev/post/day
```
- checks for events occurring in `{DAYS_AHEAD}` days and posts to Mastodon
- runs automatically via cron schedule

```bash
curl -X POST https://{worker}.{subdomain}.workers.dev/post/next
```
- posts the closest event within the next 2 weeks


## Customization

#### Changing Post Format

Modify the `postToMastodon()` function to customize:
- post text and emojis
- visibility settings (`public`, `unlisted`, `private`)
- character limits and truncation

#### Filtering Events

Add filters in `checkAndPostDayEvents()` to only post certain events:

```javascript
const tomorrowEvents = events.filter(event => {
  const eventDate = new Date(event.start);
  const isValidDate = eventDate >= tomorrow && eventDate <= tomorrowEnd;
  const isRelevantEvent = event.summary.includes('Public') || event.summary.includes('Community');
  return isValidDate && isRelevantEvent;
});
```

#### Time Zone Handling

The Worker uses UTC by default. To handle specific timezones:

```javascript
const tomorrow = new Date();
// Convert to specific timezone
const options = { timeZone: 'America/New_York' };
const localDate = new Date(tomorrow.toLocaleString('en-US', options));
```

## Troubleshooting

## Calendar Data Source

> [!IMPORTANT]
> This worker requires a CalDAV server with the [sabre/dav ICS Export Plugin](https://sabre.io/dav/ics-export-plugin/) enabled.

ICS Export Plugin allows this Worker to efficiently fetch events in jCal format, within a limited date range, by contructing a URL as follows:

`{CALENDAR_EXPORT_URL}&accept=jcal&start={timestamp}&end={timestamp}&expand=1`

## File Structure

```
cloudflare-worker-caldav-to-mastodon/
├── index.js          # main Cloudflare Worker code
├── README.md         # this document
└── wrangler.toml     # Wrangler configuration (optional)
```

## License

This project is [licensed under the MIT License](LICENSE).
