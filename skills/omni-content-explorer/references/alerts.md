# Alerts

An alert is a schedule with a condition on one tile of an existing dashboard: it runs on its cron and delivers only when the condition holds. Any user who can schedule the dashboard can create one; it needs no admin rights. For plain scheduled deliveries, recipients and email-only users, see **`omni-admin`** → *Schedules*. To build or change the tile an alert watches, see **`omni-content-builder`**.

Add `conditionType` and `conditionQueryMapKey` to the `schedules create` body (each requires the other):

| `conditionType` | Delivers when the watched tile… |
|---|---|
| `RESULTS_PRESENT` | returns rows |
| `RESULTS_MISSING` | returns no rows |
| `RESULTS_CHANGED` | returns different results from the last run |
| `RESULTS_UNCHANGED` | returns the same results as the last run |

`conditionQueryMapKey` is the watched tile's key in the dashboard's `queryPresentations.data` (`omni documents v2-get`), and `filterConfig` sets the dashboard filter values for this alert, so each alert can watch its own subset. Confirm field names with `omni schedules create --schema` before writing the body.

```bash
omni schedules create --body '{
  "identifier": "dashboard-identifier",
  "name": "Alert - projects reaching 80% of budget",
  "schedule": "0 8 ? * MON-FRI *",
  "timezone": "Europe/London",
  "destinationType": "email",
  "format": "link_only",
  "conditionType": "RESULTS_CHANGED",
  "conditionQueryMapKey": "7",
  "recipients": [{ "email": "owner@company.com" }]
}'
```

- **Watchlist tile for `RESULTS_CHANGED`.** Build the watched tile from categorical columns only (item, owner, band) and filter it to rows at the threshold. It then changes only when a row enters, leaves, or changes band. A tile that includes a numeric column changes on every run, so the alert fires daily.
- **An alert sends messages to people.** Confirm the recipients with the user before creating one.

## Choosing a tile worth alerting on

- **Keep the list short.** An alert that lists 50 or more rows every run gets ignored. Filter the tile to the rows someone should act on (a threshold, a recent window such as "crossed 100% in the last 14 days", or a minimum size), and aim for about 10 rows or fewer on a normal day.
- **Pick the condition per tile.** `RESULTS_CHANGED` suits a watchlist: you hear when a row enters or leaves. `RESULTS_PRESENT` suits an exception list that should normally be empty, for example a clean-up queue.
- **Match the cron to when the data is complete.** A weekly "missing timesheets" list run on Monday morning flags everyone who has not submitted yet. Run it after the submission deadline, and filter out leave.
- **Say which condition to use on the tile.** Put it in the subtitle ("Alert: Results changed, daily"), so a viewer setting the alert in the UI picks the right one.
- **Put alertable lists on their own page,** with optional filters (for example "only my jobs") next to them. The alert keeps the filter values set when it is created (`filterConfig`).
