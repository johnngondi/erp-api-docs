# FRC Sanctions List API

The **FRC Sanction List** quick action: the Financial Reporting Centre's sanctions list, kept up to date
automatically, plus the screening of everyone in the system against it.

Base path: `/api/v1/app/{company}/property-management/frc-sanctions`

## How the list is kept

The list is national, not per company. It is filled from two full, structured sources, read every night by
`frc:sync` (03:15), or on demand with `POST /sync` (queued):

| Source | What it gives | Resolution |
|---|---|---|
| UN Security Council consolidated list (XML) | Al-Qaida / ISIL, Taliban and Iran entries: the regimes FRC issues notices for | `1267`, `1988`, `1737` |
| FRC Kenya Domestic List (Excel, linked from the FRC TFS notices page) | Domestic designations | `1373` |

- An entry the latest pull no longer has is marked **delisted**, not deleted.
- A source that cannot be read changes nothing, so a failed download never delists anyone.
- The FRC TFS notices page is also watched. A newly published notice PDF is downloaded, read by the AI
  document extraction, and held **for review**. It changes nothing until someone applies it.
- On the very first sync, the notices already on the page are recorded as `baseline`. Their changes are
  already in the two full lists.

After every sync, and after a notice is applied, everyone in the system is **screened** (see
[Screening](#screening)).

## Permissions

| Permission | Allows |
|---|---|
| `view-frc-sanctions` | The list, notices, screening runs and matches, and both Excel downloads |
| `manage-frc-sanctions` | Upload, apply and dismiss notices; pull the sources now; run a screening now |

`manage-frc-sanctions` is listed by `GET me/permissions`, so it can gate the quick action's buttons.

## Matching rules

- A person is flagged when **every word** of one of an entry's names (its listed name, or one of its
  aliases) is in the person's name, in **any order**.
- The person may have extra names (a middle name) and is still flagged.
- An entry with a **one-word** name flags only a person whose whole name is that one word.
- Case, spaces and punctuation (`-`, `.`, `,`, `'`) are ignored. Nothing else is forgiven: there is no typo
  tolerance and no "similar spelling".
- UN aliases graded "Low" quality are not used. The UN itself says they are too broad to screen on.
- Delisted entries never flag.

| Listed | Person in the system | Flagged |
|---|---|---|
| MOUNIR · EL MOTASSADEQ | Mounir El Motassadeq | Yes |
| MOUNIR · EL MOTASSADEQ | El-Motassadeq Mounir Ali | Yes |
| MOUNIR · EL MOTASSADEQ | Mounir Otieno | No |
| FAIZ (one name) | Faiz | Yes |
| FAIZ (one name) | Faiz Mohamed | No |

## The list

`GET /` lists the entries. `GET /{frcSanction}` returns one.

Filters:
- `filter[search]`: every word typed, in any order, found in the name, an alias or the reference ("El Motassadeq Mounir" finds MOUNIR EL MOTASSADEQ). Part of a name still lists candidates.
- Exact: `filter[status]=listed|delisted`, `filter[regime]=1267,1988`, `filter[category]=individual|entity`,
  `filter[source]=un|domestic|notice`, `filter[id]`.
- `filter[has_matches]=true|false`: entries someone in the system currently matches.
- Ranges: `filter[listed_between]=2026-01-01,2026-12-31`, `filter[delisted_between]=...`.

Sorts: `name`, `reference`, `regime`, `status`, `listed_on`, `delisted_on`, `created_at`. The default is
newest listing first.

```json
{
  "data": [
    {
      "id": 41,
      "reference": "QDi.082",
      "category": { "value": "individual", "color": "primary" },
      "regime": "1267",
      "name": "MOUNIR EL MOTASSADEQ",
      "name_parts": ["MOUNIR", "EL MOTASSADEQ"],
      "aliases": ["Mounir el Moutassadeq"],
      "reason": "Arrested on 28 Nov. 2001 and found guilty in Germany ...",
      "listed_on": { "raw": "2002-09-30", "formatted": "30 Sep, 2002" },
      "status": { "value": "listed", "color": "danger" },
      "delisted_on": null,
      "source": { "value": "un", "color": "primary" },
      "notice_id": null,
      "last_synced_at": "2026-10-20T03:15:40.000000Z",
      "current_matches_count": 0
    }
  ],
  "summary": {
    "listed": 610,
    "delisted": 0,
    "last_synced_at": "2026-10-20 03:15:40",
    "pending_notices": 1,
    "sync_running": false,
    "last_audit": {
      "id": 7,
      "trigger": { "value": "sync", "color": "primary" },
      "finished_at": "2026-10-20T03:15:52.000000Z",
      "people_checked": 185,
      "matches": 0,
      "sources": { "un": { "ok": true, "updated": 597 }, "domestic": { "ok": true, "updated": 13 }, "notices": { "ok": true, "new": 0 } }
    }
  },
  "links": {}, "meta": {}
}
```

- `summary.pending_notices`: notices waiting for review, the Pending notices tab's count.
- `summary.sync_running`: a sync is still going.
- `summary.last_audit.sources`: what each source did, when the last run followed a sync. `null` otherwise.

`GET /export` downloads the list as Excel (`frc-sanctions-list-YYYYMMDD.xlsx`), with the same filters.

## Pull the sources now

`POST /sync` queues what the nightly job runs (the sources, then a screening) and answers at once with
`202`:

```json
{ "data": { "message": "Updating the FRC list. Everyone will be screened against it once it is done, in a minute or two.", "queued": true, "sync_running": true } }
```

A sync takes about a minute, mostly the UN file. Only one runs at a time. While one is running (yours, or
the nightly one), `POST /sync` queues nothing and answers `202` with `"queued": false`.

To show the result, poll `GET /` until `summary.sync_running` is `false`. Then read `summary.last_audit`, or
the newest run in `GET /audits`. A run that followed a sync has `trigger` `sync` and carries `sources`:

```json
"sources": {
  "un": { "ok": true, "added": 0, "updated": 597, "relisted": 0, "delisted": 0 },
  "domestic": { "ok": true, "added": 0, "updated": 13, "relisted": 0, "delisted": 0 },
  "notices": { "ok": true, "baseline": 0, "new": 1, "failed": 0 }
}
```

A source that failed comes back as `{ "ok": false, "error": "..." }`. Everything else was still updated,
and everyone was still screened.

## Notices

`GET /notices` lists them, newest first. Filters: `filter[status]`, `filter[regime]`, `filter[id]`,
`filter[created_between]`.

| `status` | Meaning |
|---|---|
| `pending_review` | Read; waiting for someone to apply or dismiss it |
| `applied` | Its changes are on the list |
| `dismissed` | Set aside; the list was not changed |
| `failed` | Could not be read; `error` says why. It can be dismissed |
| `baseline` | Already on the FRC page at the first sync; covered by the full lists |

`kind` is `listing`, `delisting`, `amendment`, or `null` when the notice says none of those.

### Upload a notice

Upload the PDF through the uploads endpoint first, then:

`POST /notices`

```json
{ "upload_id": 512 }
```

The PDF is read by the AI there and then (this takes a few seconds). The response is the notice with its
`entries`, each showing what applying it would do:

```json
{
  "data": {
    "message": "Notice read. Review the entries, then apply it.",
    "notice": {
      "id": 3,
      "title": "Notice No. 18 of 2026",
      "notice_number": "18 of 2026",
      "regime": "1267",
      "kind": "amendment",
      "issued_on": "2026-10-15",
      "status": { "value": "pending_review", "color": "warning" },
      "error": null,
      "entries_count": 2,
      "entries": [
        {
          "action": "delist",
          "reference": "QDi.336",
          "category": "individual",
          "name_parts": ["ANAS", "HASAN", "KHATTAB"],
          "aliases": [],
          "reason": null,
          "name": "ANAS HASAN KHATTAB",
          "sanction_id": 77,
          "effect": "delist"
        },
        {
          "action": "list",
          "reference": "QDi.427",
          "category": "individual",
          "name_parts": ["NOOR", "WALI", "MEHSUD"],
          "aliases": [],
          "reason": "Leader of TTP.",
          "name": "NOOR WALI MEHSUD",
          "sanction_id": null,
          "effect": "add"
        }
      ]
    }
  }
}
```

`effect` is one of:
- `add`: a new entry.
- `update`: changes a listed entry.
- `relist`: lists a delisted entry again.
- `delist`: removes a listed entry.
- `already_delisted`: nothing to do.
- `not_on_list`: a delisting for an entry we never had; nothing to do.

A document that is not a readable sanctions notice comes back with status `failed`, and `error` says why
(e.g. "This is a utility bill.").

`GET /notices/{notice}` returns the same detail, `entries` included. List rows carry only `entries_count`.

### Apply or dismiss

- `POST /notices/{notice}/apply` applies a `pending_review` notice, then screens everyone. The response
  carries `effects` (a count per effect), the notice, and the screening run (`audit`).
- `POST /notices/{notice}/dismiss` sets a `pending_review` or `failed` notice aside without changing the
  list.

Either one on a notice in any other status is a `422` keyed on `status`.

## Screening

Who is screened:
- Everyone in the users table: tenants, landlords, suppliers, staff and directors.
- Every lease applicant, under the name on the application and the name read off their ID.
- Every guarantor.

A run happens after each nightly sync, after a notice is applied, and on demand.

- `POST /audits` runs one now (the **Audit** button). The response is
  `{ "message": "1 person in the system matches the FRC list.", "audit": {...} }`.
- `GET /audits` lists the runs, newest first. `GET /audits/{audit}` returns one.
  - Filters: `filter[trigger]=sync|notice|manual`, `filter[id]`, `filter[created_between]`.

```json
{
  "id": 8,
  "trigger": { "value": "manual", "color": "secondary" },
  "sanctions_checked": 610,
  "people_checked": 185,
  "matches": 1,
  "new_matches": 1,
  "cleared_matches": 0,
  "sources": null,
  "started_at": "2026-10-20T09:00:00.000000Z",
  "finished_at": "2026-10-20T09:00:02.000000Z",
  "runBy": { "id": 12, "name": "Jane Wambui" }
}
```

### Matches (the FRC flag report)

`GET /matches` returns current matches by default. Use `filter[current]=false` to include the cleared ones.

Filters:
- `filter[search]`: the person's name, or the name matched, word by word in any order.
- `filter[subject_type]=user|lease_application|applicant_guarantor`.
- `filter[frc_sanction_id]`, `filter[regime]`, `filter[id]`.
- `filter[flagged_between]`.

```json
{
  "id": 5,
  "subject": { "type": "user", "id": 230, "name": "Mounir El Motassadeq", "roles": ["vendor"] },
  "matched_name": "MOUNIR EL MOTASSADEQ",
  "company_id": null,
  "sanction": { "id": 41, "reference": "QDi.082", "name": "MOUNIR EL MOTASSADEQ", "reason": "..." },
  "is_current": true,
  "first_flagged_at": "2026-10-20T09:00:02.000000Z",
  "last_seen_at": "2026-10-20T09:00:02.000000Z",
  "cleared_at": null
}
```

- `subject.roles` names the portals a matched user is in: `tenant`, `landlord`, `vendor`, `app` (staff).
  For an applicant it is `["applicant"]`, and for a guarantor `["guarantor"]`.
- A match stays current while it keeps matching. It is **cleared** when the entry is delisted or the
  person's name changes. Cleared matches are kept for the history.
- A company sees matches on users, and on its own applicants and guarantors only.

`GET /matches/export` downloads the flagged cases, with the same filters:
- `format=excel` (the default): `frc-flagged-cases-YYYYMMDD.xlsx`, reasons in full.
- `format=pdf`: `frc-flagged-cases-YYYYMMDD.pdf`, on the company letterhead like the report PDFs. Long
  reasons are shortened to fit the page.

Any other `format` is a `422` keyed on `format`.

Nothing is blocked on a match: it is reported and notified only.

## Who is notified

Finance settings has **FRC Flag Notifiable Staff** (`frc_flag_notifiable_staff`), a staff picker like
Default Receivables Staff. That person is notified as follows:

- **New matches:** in-app, and **by email**. The email lists the new matches and attaches every current
  flagged case as Excel, with the new ones marked, so they can investigate and report from it.
- **A new FRC notice to review:** in-app only.

| Notification `type` | When | `data` |
|---|---|---|
| `App\Notifications\Compliance\FrcSanctionMatchesFoundNotification` | A screening run finds **new** matches they can see. One per run; a match already known is not notified again | `audit_id`, `new_matches`, `matches[]` (`subject_type`, `subject_id`, `subject_name`, `reference`, `sanctioned_name`, `matched_name`, `regime`) |
| `App\Notifications\Compliance\FrcNoticeAwaitingReviewNotification` | The nightly sync found and read a new FRC notice | `notice_id`, `entries` |

`resource_url` opens the screening run, or the notice. When nobody is set, no one is notified; the report
is still there.

## Lease application vetting

Vetting checks applicants and guarantors against this list, by the same matching rules. It no longer reads
the old watchlist that `frc:import` filled.
