# e-qanun.az API reference (reconstructed)

Black-box map of the public, unauthenticated REST backend behind Azerbaijan's
official legislation portal, plus the Referer-guarded static full-text host.
All endpoints below were **verified live on 2026-07-30**. No authentication is
used and the client identifies itself by name rather than impersonating a
browser; content is public legal acts (Ministry of Justice).

Document counts quoted in this file are live figures that drift as acts are
published — see [`README.md`](README.md) for the current corpus breakdown.

- REST API base: `https://api.e-qanun.az` (Spring Boot; JSON)
- Full-text host: `https://e-qanun.az` (nginx; static MS-Word HTML, hotlink-guarded)
- Frontend: `https://e-qanun.az` (Next.js). The **AI** portal `e-qanun.ai` is
  Cloudflare-challenged with no public API — not integrable.

For the static full-text host a same-site `Referer: https://e-qanun.az/` is
**required** (bare request → 403). A browser-looking `User-Agent` is **not**
required — verified: an identifying UA plus the Referer returns 200, while a full
Chrome UA without the Referer returns 403. Identify your client honestly.

---

## Search — `GET /getDetailSearch`

The real search endpoint. (The earlier recon guessed `/framework/search`, which
exists but is a different/unused route; `/getDetailSearch` is what the site calls.)

Query parameters (as sent by the frontend, with confirmed semantics):

| Param | Required | Values | Meaning |
|---|---|---|---|
| `name` | no | string | search term(s) — the connector spec marks it required as a UX choice, but the API accepts its absence |
| `length` | **yes** | int | page size — omitting it returns **HTTP 500** |
| `orderColumn` | **yes** | int | sort column (site uses `8`) — omitting it returns **HTTP 500** |
| `array` | **yes** | comma-separated type ids, or empty | act-type filter — omitting the parameter returns **HTTP 500**; send it empty for no filter |
| `title` | no | `true`/`false` | search in act **titles** |
| `text` | no | `true`/`false` | search in act **full text** (broader) |
| `exact` | no | `true`/`false` | exact-match toggle |
| `statusId` | no | `0` / `1` / `2` | `0`=all, `1`=in force (`Qüvvədədir`), `2`=repealed (`statusName` returns `Ləğv olunmuşdur`) |
| `start` | no | int | pagination offset |
| `orderDirection` | no | `asc`/`desc` | sort direction |
| `codeType` | no | int | site sends `1` |
| `dateType` | no | int | site sends `1` |
| `secondType` | no | int | site sends `2` |
| `specialDate` | no | `true`/`false` | site sends `false` |

The three mandatory extras were established by bisection against a working
request: dropping `length`, `orderColumn` or `array` gives a 500, dropping any of
the other eleven still returns 200.

`array` is the **act-type filter**, not dead weight — its values are the ids
returned by `/all-type`. Verified 2026-07-30: `&array=73` → `totalCount` 464
(Constitutional Court decisions), `&array=73,87` → 578 (adding the 114 Supreme
Court Plenum decisions). Note that `/detailNameCount` **ignores** `array` (same
59,368 with and without it), so a type-filtered count has to come from
`getDetailSearch` with `length=1` and reading `totalCount`.

Example:

```
GET https://api.e-qanun.az/getDetailSearch?start=0&length=20&orderColumn=8&orderDirection=desc&name=m%C3%BClki&title=true&text=false&exact=false&statusId=1&codeType=1&dateType=1&secondType=2&specialDate=false&array=
```

Type-filtered example — Constitutional Court decisions only:

```
GET https://api.e-qanun.az/getDetailSearch?start=0&length=20&orderColumn=8&orderDirection=desc&name=&title=true&text=false&exact=false&statusId=0&codeType=1&dateType=1&secondType=2&specialDate=false&array=73
```

Response:

```json
{
  "data": [
    {
      "rowNum": 1,
      "id": 62242,
      "citation": "3-16/1-KQ/12/2026",
      "title": "…",
      "typeName": "Ədliyyə Nazirliyi",
      "statusName": "Qüvvədədir",
      "effectDate": null,
      "registerCode": "15202607220012",
      "registerDate": null,
      "acceptDate": "22.07.2026",
      "classCode": "2026-07-22"
    }
  ],
  "totalCount": 1610
}
```

The arithmetic to sanity-check the filters: title/all == title/in-force +
title/cancelled, and full-text/in-force is far larger than title/in-force.
Absolute figures drift daily as acts are published — re-run rather than trusting
a number written here.

## Result count — `GET /detailNameCount`

Cheap count without the result list. Params: `name`, `title`, `text`,
`statusId`, `exact`. The **count is returned in the `id` field**:

```
GET /detailNameCount?name=m%C3%BClki&title=true&text=false&statusId=1&exact=false
→ {"id": <count>, "name": null}
```

## Act metadata — `GET /framework/{id}`

```
GET https://api.e-qanun.az/framework/46944
```

```json
{
  "data": {
    "id": 46944,
    "requisite": {
      "title": "Azərbaycan Respublikasının Mülki Məcəlləsi",
      "acceptDate": "28.12.1999",
      "effectDate": "01.09.2000",
      "typeName": "Məcəllələr",
      "statusName": "Qüvvədədir",
      "citation": "",
      "registerCode": "",
      "registerDate": null,
      "classCodes": ["020.010.000"]
    },
    "htmlUrl": "https://e-qanun.az/frameworks/46/f_46944.html",
    "isFavorite": false
  }
}
```

`htmlUrl` is canonical — read it, do not construct it. (The bucket folder is
`floor(id/1000)`, e.g. id 46944 → `/frameworks/46/`, but always trust the field.)

## Full text — `GET {htmlUrl}` (Referer required)

```
GET https://e-qanun.az/frameworks/46/f_46944.html
```

- Bare request → **403** (nginx hotlink guard).
- With `Referer: https://e-qanun.az/` → **200**, `text/html` (the Civil Code is
  ~9 MB, ~2M chars of text).
- The `Referer` is what the guard checks. A browser-looking `User-Agent` is not
  needed: identifying UA + Referer → 200; Chrome UA with no Referer → 403.
- **On the Referer:** sending it is a deliberate step past the publisher's
  hotlink protection, and it is worth naming rather than glossing. The reasoning:
  the content is public law, no authentication or paywall is being defeated, this
  client identifies itself and gives a contact route, and it fetches single acts
  at a low rate rather than mirroring. If the operator objects, stop — and prefer
  an official arrangement (see *Restricted* below) for anything at scale.
- **Encoding:** MS-Word HTML export declaring a legacy single-byte codepage that
  **varies per document** — windows-1251, windows-1252 and windows-1254 all occur
  (verified: act 46944 declares Windows-1254, act 12312 windows-1252, act 62170
  windows-1251). Azerbaijani Latin letters arrive as numeric character references
  (`&#601;` = ə, `&#305;` = ı). Always decode using the codepage declared in
  **that document's own `<meta>`**, never a fixed one; decoding as UTF-8 corrupts
  the symbol bytes (№, «», dashes). This package's `_html.decode_html` reads the
  declared charset per document and only falls back when none is present.

## Taxonomy / lookups

- `GET /sections` — 4 top sections: Normativ hüquqi aktlar, Normativ xarakterli
  aktlar, Qeyri-normativ hüquqi aktlar, Digər.
- `GET /all-type` — act-type tree (`id`, `name` with counts, `sectionId`,
  `parentId`, `canceled`). ~80 nodes: Konstitusiya, Qanunlar, Prezident
  Fərmanları, Məcəllələr, … Each `name` carries its live count in parentheses;
  the ids are what the `array` filter accepts.
- `GET /dictionary?categories=0&categories=1&…&categories=8` — dictionary/taxonomy.
- `GET /menu`, `GET /warnings/status` — misc site data.

## Index — `GET https://e-qanun.az/sitemap.xml`

~5.6 MB, enumerates `/framework/<id>` for sequential ids from 0 — an observed
property of the site, noted here for completeness.

**This project deliberately ships no bulk harvester and enumerating the corpus is
out of scope.** It is a lookup tool: search, fetch one act, read it. If you need
the full dataset, request it from justice.gov.az rather than crawling the id
space.

## Restricted

- `GET /swagger-ui/index.html`, `GET /v2/api-docs` → 403 (present but gated).
  An official OpenAPI spec / API key could be requested from justice.gov.az.

---

## Endpoint status matrix

| Endpoint | Method | Status | Purpose |
|---|---|---|---|
| `/getDetailSearch` | GET | ✅ verified | search (resolved) |
| `/detailNameCount` | GET | ✅ verified | result count |
| `/framework/{id}` | GET | ✅ verified | act metadata + htmlUrl |
| `{htmlUrl}` (static) | GET | ✅ verified | full text (Referer required, per-doc codepage) |
| `/sections` | GET | ✅ verified | top sections |
| `/all-type` | GET | ✅ verified | act-type tree |
| `/dictionary` | GET | ✅ verified | taxonomy |
| `/framework/search` | GET | ⚠️ exists, unused | superseded by /getDetailSearch |
| `/swagger-ui`, `/v2/api-docs` | GET | 🔒 403 gated | official spec (ask MoJ) |
| `e-qanun.ai` | — | ❌ Cloudflare | AI portal, no public API |
