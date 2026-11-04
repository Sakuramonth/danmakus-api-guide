# Danmakus API Guide (Unofficial)

An unofficial, community-maintained guide to the [Danmakus](https://danmakus.com) API,
reverse-engineered from the official Swagger definitions and verified by live calls in November 2026.

> **Disclaimer**: This is not an official document. Endpoints may change without notice.
> The Chinese notes column in the original research is summarized in English here; behavior claims marked ✅ were observed live.

## Base URLs

Discovered via `GET /api/v2/meta/nodes`:

| Node | Origin |
| --- | --- |
| Primary | `https://api.ukamnads.icu` |
| Alternate 1 | `https://ukamnads.icu` |
| Alternate 2 (HK) | `https://api.danmakus.com` (some egress IPs hit Cloudflare error 1010) |

Swagger UI: `https://api.ukamnads.icu/swagger/index.html` — definitions `v2` and `v3`.
Both definitions currently expose the **same** paths and schemas (158 paths, 165 operations); the endpoints grouped as new in the v3 UI still live under `/api/v2/` paths.

> ⚠️ The published `swagger.json` is not strict JSON: one example payload contains an empty value (`"kindValue": ,`).
> Sanitize `: ,` / `: }` empty values to `null` before feeding it to a strict parser or code generator.

## Authentication

Public market/ranking endpoints need no auth. Account endpoints use a **custom header**:

```
Token: <your Access Token>
```

Observed behavior (verified live):

| Attempt | Result |
| --- | --- |
| `Authorization: Bearer <token>` | `401` — please log in first |
| No header | `401` — not logged in |
| `Token: <wrong value>` | `403` — invalid credentials |
| `Token: <Access Token>` | `200` |

Your Access Token is the one used to log in on danmakus.com (the web app stores the same value in `localStorage` under the key `Token`).
Older third-party clients exchanged an OAuth token at `POST /api/v2/accounts/token` — that endpoint is **gone** (404 on both ukamnads nodes, absent from the current spec). Do not build on it.

## Conventions

- **Envelope**: most responses are `{"code": 200, "message": "", "data": ...}`. A few endpoints return bare arrays/objects (e.g. `POST /api/v2/streamer-status`, `GET /api/v2/meta/nodes`).
- **Pagination**: `pageNum` starts at 0; list responses carry `{total, pageNum, pageSize, hasMore, data[]}`.
- **Time fields**: ISO 8601 values are UTC; `settledAt` / `generatedAt` are Unix **seconds**; `timestamp` / `startDate` are Unix **milliseconds**.
- **IDs**: streamers are keyed by their Bilibili UID (`streamerUid`, int64). Channels in detail endpoints are fetched with `GET /api/v2/channel?uid=<bilibili uid>`; a separate family of paths takes an internal channel UUID that is not the UID.

## Quickstart

```bash
TOKEN='<your access token>'

# market pulse (public)
curl -s https://api.ukamnads.icu/api/v2/vstock/pulse

# your portfolio (auth required)
curl -s https://api.ukamnads.icu/api/v2/vstock/portfolio -H "Token: $TOKEN"

# one ticker
curl -s https://api.ukamnads.icu/api/v2/vstock/ticker/1932862336
```

## VStock trading

`POST /api/v2/vstock/trade` with a JSON body of exactly three fields:

```json
{"streamerUid": 631699988, "side": 1, "shares": 10}
```

`side` is the `VStockTradeSide` enum (1–4):

| side | Meaning | Status |
| --- | --- | --- |
| 1 | Buy long (`buy_long`) | ✅ verified |
| 2 | Sell long / close long (`sell_long`) | ✅ verified |
| 3 | Open short (`sell_short`) | ✅ verified |
| 4 | Close short (`buy_short`) | ✅ verified |

It is a **market order**: it fills immediately at the vAMM price and returns
`{executionPrice, grossAmount, feeAmount, realizedPnl, priceAfter, cashBalanceAfter}`.
Fees observed live: **0.2%** on buys and on short opens; a sell or short close within 12 hours of opening is charged **1.0%** total.
Short positions lock margin of **150%** of notional (released in full on close), and the API rejects shorting a ticker you currently hold long:
`400` — holding a long position in this ticker, shorting is forbidden until the long is closed.
There is no order book and no cancel step — check `GET /api/v2/vstock/ticker/{uid}` (price, `pbRatio`) and your `portfolio` before sending a trade.

Two write endpoints deserve explicit caution:

- `POST /api/v2/vstock/burn` — permanently burns shares into crystal points (lock-up related). Irreversible.
- `POST /api/v2/vstock/portfolio/claim-bankruptcy-relief` — claims bankruptcy relief for the account.

## Endpoint reference

Status key: ✅ 200 = called successfully during the November 2026 verification pass (114 GET endpoints, 102 returned 200; write endpoints exercised one by one on 2026-11-04 — see per-row notes and the appendix).
`HTTP xxx (see Notes)` = the observed status for that call (server 503s, parameter validation 400s, or not-found probes). Required parameters are marked with `*`.

### Account

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/account/add-record` | uId:integer | ✅ 200 |
| POST `/api/v2/account/avatar-intent` | body:AvatarUploadIntentRequest | ✅ 200 (verified 2026-11: returns an upload key and a presigned uploadUrl; intent only, upload not completed). |
| GET `/api/v2/account/channel-lives` | uId:integer; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/contributed-lives` | pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/core-config` | - | ✅ 200 |
| POST `/api/v2/account/core-config` | body:CoreControlConfig | ✅ 200 (verified 2026-11: read the current config, then wrote it back unchanged). |
| GET `/api/v2/account/del-record` | uId:integer | ✅ 200 |
| GET `/api/v2/account/info` | id:integer | ✅ 200 |
| GET `/api/v2/account/recording` | - | ✅ 200 |
| GET `/api/v2/account/search-points` | - | ✅ 200 |
| GET `/api/v2/account/search-points/history` | page:integer; pageSize:integer; filter:string | ✅ 200 |
| GET `/api/v2/account/statistics` | start:string; end:string | ✅ 200 |
| POST `/api/v2/account/statistics/report` | body:ReportStatisticRequest | ✅ 200 (verified 2026-11: zero-value report accepted). |
| POST `/api/v2/account/update-avatar` | body:RequestUpdateAvatar | Tested ✅ round 2 (2026-11-04): intent → PUT upload → update all returned 200 and the avatar really changed (no API to unset an avatar; not reversible; update uses the avatarUrl in the body) |
| POST `/api/v2/account/update-name` | body:RequestUpdateUserName | ✅ 200 (verified 2026-11: renamed, verified, then renamed back). |
| POST `/api/v2/account/update-public-profile` | body:RequestUpdateContributorPublicProfile | ✅ 200 (verified 2026-11: current values written back unchanged). |
| POST `/api/v2/account/update-recording-setting` | - | ✅ 200 (verified 2026-11: the setting list is empty and was written back as-is; this endpoint replaces the whole recording-setting list — use with care). |

### AgencyWiki

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/agencies` | keyword:string; type:AgencyGroupType; pageNum:integer; pageSize:integer; sort:string | ✅ 200 |
| POST `/api/v2/agencies/avatar-complete` | body:CompleteAgencyAvatarRequest | Tested ✅ round 2 (2026-11-04): returned 200 together with avatar-intent and the PUT upload; the temporary agency proposal was withdrawn for cleanup |
| POST `/api/v2/agencies/avatar-intent` | body:AvatarUploadIntentRequest | Tested ✅ round 2 (2026-11-04, temporary agency): returned 200 with intent and upload URL; the presigned PUT upload returned 200; the temporary agency proposal was withdrawn for cleanup |
| POST `/api/v2/agencies/propose` | body:SubmitAgencyUpsertRequest | Tested ✅ round 2 (2026-11-04): returned 200 and created a real proposal (type=0 agency creation); withdrawn for cleanup (status set to 7) |
| POST `/api/v2/agencies/sync-vdb` | - | HTTP 403 (verified 2026-11: 'only wiki moderators may trigger a VDB sync'). |
| GET `/api/v2/agencies/{uuid}` | uuid*:string | ✅ 200 |
| POST `/api/v2/agencies/{uuid}/propose` | uuid*:string; body:SubmitAgencyUpsertRequest | Tested ✅ round 2 (2026-11-04): returned 200 against the temporary agency and created a real proposal (type=1 agency profile); withdrawn for cleanup |
| GET `/api/v2/agency-options` | - | ✅ 200 |
| GET `/api/v2/agency/{uuid}` | uuid*:string | ✅ 200 |
| GET `/api/v2/agency/{uuid}/timeline` | uuid*:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/{userId}/lineage` | userId*:integer; platform:string; includeSuspected:boolean | ✅ 200 |
| GET `/api/v2/channel/{userId}/timeline` | userId*:integer; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/affiliation/submit` | body:SubmitChannelAffiliationRequest | Tested ✅ round 2 (2026-11-04): returned 200 and created a real proposal (type=2 affiliation); withdrawn for cleanup |
| GET `/api/v2/wiki/channel/{userId}/career` | userId*:integer | ✅ 200 |
| GET `/api/v2/wiki/channel/{userId}/lives` | userId*:integer; keyword:string; limit:integer | ✅ 200 |
| GET `/api/v2/wiki/content` | scopeKey:string; targetId:string | HTTP 400 (see Notes) |
| POST `/api/v2/wiki/content/revise` | body:SubmitWikiContentRevisionRequest | Tested ✅ round 2 (2026-11-04): returned 200 against a channel of the temporary agency and created a real proposal (type=8 channel bio); withdrawn for cleanup |
| GET `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer | HTTP 403 (see Notes) |
| POST `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer; body:UpdateWikiAccountStateRequest | HTTP 403 (verified 2026-11: 'only wiki admins may change account roles/bans'). |
| GET `/api/v2/wiki/governance/profile` | - | ✅ 200 |
| GET `/api/v2/wiki/lineage` | platform:string; identity:string; includeSuspected:boolean | ✅ 200 |
| POST `/api/v2/wiki/lineage/submit` | body:SubmitLineageRequest | Really called, blocked by business rules (round 2, 2026-11-04): first 400 demanding a verifiable source link; after adding one, 400 'strong-consensus action: requires community Resident rank or above' — the required rank was not met |
| POST `/api/v2/wiki/lineage/{lineageId}/dispute` | lineageId*:string; body:SubmitWikiDisputeRequest | Not testable (round 2, 2026-11-04): the site currently has no lineage records (several channels' lineage lists are all empty), so no real lineageId exists to dispute |
| GET `/api/v2/wiki/proposals` | status:WikiProposalStatus; agencyUuid:string; streamerUid:integer; proposalType:WikiProposalType; scopeKey:string; mine:boolean; votedByMe:boolean; appealed:boolean; aiOutcome:WikiAiReviewOutcome; sort:WikiProposalSort; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/proposals/{proposalId}/appeal` | proposalId*:string; body:AppealWikiProposalRequest | Really called, blocked by preconditions (round 2, 2026-11-04): 400 — appeals are only allowed within 14 days after a malicious rejection; no eligible proposal available |
| POST `/api/v2/wiki/proposals/{proposalId}/emergency` | proposalId*:string; body:EmergencyWikiProposalRequest | Really called, blocked by permissions (round 2, 2026-11-04): 403 'emergency handling is restricted to wiki moderators' |
| POST `/api/v2/wiki/proposals/{proposalId}/vote` | proposalId*:string; body:VoteWikiProposalRequest | Really called, blocked by permissions (round 2, 2026-11-04): 403 — review voting is not unlocked (requires 30-day registration plus a contribution threshold) |
| POST `/api/v2/wiki/proposals/{proposalId}/withdraw` | proposalId*:string; body:WithdrawWikiProposalRequest | Tested ✅ (2026-11-04, round 1 once + round 2 six times): returns 200 and sets the proposal's status to 7 (withdrawn); calling it on an already-withdrawn/closed proposal returns 400 'only proposals still under review can be withdrawn' |
| GET `/api/v2/wiki/timeline` | userId:integer; agencyUuid:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/timeline/submit` | body:SubmitWikiNodeRequest | Tested ✅ round 2 (2026-11-04): returned 200 and created a real proposal (type=3 timeline create); withdrawn for cleanup |
| POST `/api/v2/wiki/timeline/{eventId}/reaction` | eventId*:string; body:ToggleWikiReactionRequest | Tested ✅ round 2 (2026-11-04): returned 200 lighting a 👍 on the account's own timeline node (verified by re-read); calling again toggles it off — cleaned up (net effect zero) |
| POST `/api/v2/wiki/timeline/{nodeId}/dispute` | nodeId*:string; body:SubmitWikiDisputeRequest | Really called, blocked by business rules (round 2, 2026-11-04): first 400 demanding a counter-evidence link; after adding one, 400 'strong-consensus action: requires community Resident rank or above' — rank insufficient |
| POST `/api/v2/wiki/timeline/{nodeId}/revise` | nodeId*:string; diffSummary:string; body:SubmitWikiNodeRequest | Tested ✅ round 2 (2026-11-04): returned 200 against the account's own timeline node and created a real proposal (type=4 timeline revision); withdrawn for cleanup |

### AnonymousDanmakuIdentity

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}/danmakus-with-identities` | liveId*:string; offset:integer; limit:integer; type:array; includeEnter:boolean | ✅ 200 |

### ArchivedDanmakuCursor

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}/danmakus/after` | liveId*:string; limit:integer; afterTs:integer; afterSeq:integer; type:array | ✅ 200 |

### AudienceLeaderboard

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/channel/audience-leaderboard` | uId:integer; board:string; range:string; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/audience-trend` | uId:integer | ✅ 200 |
| GET `/api/v2/channel/compare` | uids:string; range:string | ✅ 200 |
| GET `/api/v2/channel/compare/presets` | - | ✅ 200 |
| GET `/api/v2/rank/audience/day` | date:string; board:string | ✅ 200 |

### Contribution

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/contribution/areas` | range:integer; topUsersPerArea:integer; areaCount:integer | ✅ 200 |
| GET `/api/v2/contribution/contributor` | accountId*:integer | HTTP 400 (see Notes) |
| GET `/api/v2/contribution/hub` | range:integer; metric:ContributionMetric; top:integer; topUsersPerArea:integer; areaCount:integer | ✅ 200 |
| GET `/api/v2/contribution/me` | start:string; end:string | ✅ 200 |
| GET `/api/v2/contribution/minute-today` | - | ✅ 200 |
| GET `/api/v2/contribution/overview` | range:integer; top:integer; metric:ContributionMetric | ✅ 200 |
| GET `/api/v2/contribution/rank/day` | year:integer; month:integer; day:integer; metric:ContributionMetric; count:integer | ✅ 200 |
| GET `/api/v2/contribution/rank/range` | range:ContributionRange; metric:ContributionMetric; count:integer | ✅ 200 |

### CoreRuntime

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/core-runtime/clients` | - | ✅ 200 |
| POST `/api/v2/core-runtime/clients/remark` | body:ClientRemarkRequest | ✅ 200 (verified 2026-11: remark set on a self-created temporary client, then cleared; a fabricated clientId returns business code 400 'device not found'). Body: {clientId, remark}. |
| POST `/api/v2/core-runtime/clients/remove` | body:RemoveClientRequest | ✅ 200 (verified 2026-11: self-created temporary client removed; a fabricated clientId also returns success as a no-op). Body: {clientId, blockClientId}. |
| GET `/api/v2/core-runtime/extraction-rules` | - | ✅ 200 |
| GET `/api/v2/core-runtime/heartbeat` | clientId:string | HTTP 204 (see Notes) |
| POST `/api/v2/core-runtime/heartbeat` | body:CoreHeartbeatRequest | Tested ✅ round 2 (2026-11-04, temporary clientId): returns 204 No Content (do not JSON-parse the response — an empty body reads as a false failure); this round's temporary client was cleaned up via state(state=2) + clients/remove |
| POST `/api/v2/core-runtime/request-room` | body:RequestRoomRequest | ✅ 200 (verified 2026-11: desiredCount=0 call returns holdingRooms, newlyAssignedRooms, droppedRooms, effectiveCapacity). |
| DELETE `/api/v2/core-runtime/state` | clientId:string | ✅ 200 (verified 2026-11: state of the self-created temporary client cleared). |
| POST `/api/v2/core-runtime/sync` | body:CoreStateSyncRequest | ✅ 200 (verified 2026-11: state report accepted; creates a record in the clients list). |
| POST `/api/v2/core-runtime/upload-extracted` | - | Still not really testable (round 2, 2026-11-04): Swagger declares no request body; an empty body returns 400 'request body must not be empty', and six constructions (JSON array, three JSON object shapes, multipart, base64 envelope) all return 400 'failed to parse request body' — likely a core-client-only undisclosed format |
| GET `/api/v2/core-runtime/ws` | - | HTTP 400 (see Notes) |

### DanmakuControllers

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/add` | uid*:integer; token*:string | ✅ 200 |
| GET `/api/v2/area` | - | ✅ 200 |
| GET `/api/v2/channel` | uId*:integer; includeLive:boolean; includeExtra:boolean; maxLives:integer; livesOffset:integer; includeFansHistory:boolean | ✅ 200 |
| GET `/api/v2/channel/filter` | type:ChannelRankType; range:integer; keyword:string; area:string; pageNum:integer; pageSize:integer; includeExtra:boolean | ✅ 200 |
| POST `/api/v2/channel/filter` | body:PostChannelInfo | Verified live. Query-style POST. |
| GET `/api/v2/channels` | simple:boolean | ✅ 200 |
| POST `/api/v2/chat/add` | body:ChatModel | Really called, blocked (round 2, 2026-11-04): business code 403 'verification failed, possibly sending too frequently' — token is likewise a web CAPTCHA token; cannot be completed via pure API |
| GET `/api/v2/chat/get` | date:integer; count:integer | ✅ 200 |
| POST `/api/v2/comment/add` | uId:integer; body:AddCommentRequest | Really called, blocked (round 2, 2026-11-04): HTTP 200 with business code 403 'human verification failed' — token is a web CAPTCHA token; cannot be completed via pure API |
| GET `/api/v2/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/comments` | targetType:SiteCommentTargetType; targetId:string | HTTP 400 (see Notes) |
| POST `/api/v2/comments` | body:AddSiteCommentRequest | Really called, blocked (round 2, 2026-11-04): HTTP 200 with business code 403 'human verification failed' — the body's token is a web CAPTCHA token and no API endpoint issues one; cannot be completed via pure API |
| GET `/api/v2/comments/counts` | targetType:SiteCommentTargetType; ids:string | HTTP 400 (see Notes) |
| GET `/api/v2/comments/stream` | viewerId:string | ✅ 200 |
| DELETE `/api/v2/comments/{id}` | id*:string | Not testable (round 2, 2026-11-04): own comments cannot be created because the comment endpoints are CAPTCHA-blocked, leaving nothing of the account's own to delete; other users' comments are not deleted |
| GET `/api/v2/emoicons` | - | ✅ 200 |
| GET `/api/v2/guilds` | - | ✅ 200 |
| GET `/api/v2/live` | liveId*:string; uId:integer; type:array; pageNum:integer; pageSize:integer; includeExtra:boolean; useEmoji:boolean; includeDanmakus:boolean | ✅ 200 |
| POST `/api/v2/live/comment/add` | body:AddLiveCommentRequest | Really called, blocked (round 2, 2026-11-04): posting 'test' to live room 76693693 (streamer 上官林萤, 0 live viewers) returned HTTP 200 with business code 403 'human verification failed' — same CAPTCHA block; no visible comment was created |
| GET `/api/v2/live/comment/get` | liveId:string | ✅ 200 |
| GET `/api/v2/living` | includeExtra:boolean | ✅ 200 |
| GET `/api/v2/popularChannels` | count:integer | ✅ 200 |
| GET `/api/v2/popularUsers` | count:integer | ✅ 200 |
| GET `/api/v2/rank/day` | year*:integer; month*:integer; day*:integer; type*:RankType; area:string; count:integer | ✅ 200 |
| GET `/api/v2/rank/month` | year*:integer; month*:integer; type*:RankType; area:string; count:integer | ✅ 200 |
| GET `/api/v2/rank/range` | range*:RankRange; type*:RankType; area:string; count:integer | ✅ 200 |
| GET `/api/v2/simpleChannel` | uId*:integer | ✅ 200 |
| GET `/api/v2/site` | - | ✅ 200 |
| GET `/api/v2/user` | uid*:integer; target*:integer; pageNum:integer; pageSize:integer; useEmoji:boolean; hideOnlyEnter:boolean | ✅ 200 |
| POST `/api/v2/user/comment/add` | body:AddUserCommentRequest | Really called, blocked (round 2, 2026-11-04): HTTP 200 with business code 403 'human verification failed' — same as the comments group; cannot be completed via pure API |
| GET `/api/v2/user/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/user/name` | uid*:array | ✅ 200 |
| GET `/api/v2/user/watchedChannels` | uid*:integer; simple:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/user/watchedChannelsSimple` | uid*:integer; distinct:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/vup-list` | - | ✅ 200 |

### DanmakuSearch

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/danmaku/search` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200 |
| POST `/api/v2/danmaku/search/async` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200 (verified 2026-11: task created, returns taskId). |
| GET `/api/v2/danmaku/search/async/{taskId}` | taskId*:string | ✅ 200 |
| GET `/api/v2/danmaku/search/queue-stats` | - | ✅ 200 |

### DanmakuV3

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}` | liveId*:string; include:string | ✅ 200 |
| GET `/api/v3/lives/{liveId}/danmakus` | liveId*:string; offset:integer; limit:integer; userId:integer; type:array | ✅ 200 |
| GET `/api/v3/lives/{liveId}/delta` | liveId*:string; afterSeq:integer; includeEnter:boolean | HTTP 404 (see Notes) |
| GET `/api/v3/lives/{liveId}/full` | liveId*:string; include:string; includeEnter:boolean; offset:integer; limit:integer | ✅ 200 |
| GET `/api/v3/meta/emoicons` | - | ✅ 200 |
| GET `/api/v3/meta/index-background` | afterTs:integer; take:integer | ✅ 200 |
| GET `/api/v3/users/{userId}/bootstrap` | userId*:integer; page:integer; pageSize:integer; channelUserId:integer; hideOnlyEnter:boolean; maxChannels:integer | ✅ 200 |
| GET `/api/v3/users/{userId}/history` | userId*:integer; page:integer; pageSize:integer; channelUserId:integer; hideOnlyEnter:boolean | ✅ 200 |

### Identity

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/identity/account/{accountId}` | accountId*:integer | ✅ 200 |
| GET `/api/v2/identity/catalog` | - | ✅ 200 |
| POST `/api/v2/identity/equip` | body:IdentityEquipRequest | HTTP 200 with business code 400 (verified 2026-11: 'identity does not exist' — no such identity owned). |
| GET `/api/v2/identity/nameplates` | ids:string; streamerUid:integer | HTTP 400 (see Notes) |

### Meta

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/meta/nodes` | - | ✅ 200 |
| GET `/api/v2/meta/ping` | - | ✅ 200 |

### StreamerStatus

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| POST `/api/v2/streamer-status` | body:StreamerStatusRequest | Verified live. Query-style POST. |

### UserAnalyzeJobs

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| POST `/api/v3/user-analysis-jobs` | body:CreateUserAnalyzeJobRequest | ✅ 200 (verified 2026-11: job created, returns jobId, and completes). |
| DELETE `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404 (verified 2026-11: even for a freshly created job that GET can read — the DELETE route appears unimplemented server-side). |
| GET `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404 (see Notes) |
| GET `/api/v3/user-analysis-jobs/{jobId}/events` | jobId*:string | HTTP 404 (see Notes) |
| GET `/api/v3/user-analysis-jobs/{jobId}/result` | jobId*:string | HTTP 404 (see Notes) |

### VStock

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/vstock/agencies` | sort:string | ✅ 200 |
| GET `/api/v2/vstock/agency/{uuid}/index` | uuid*:string | ✅ 200 |
| POST `/api/v2/vstock/burn` | body:VStockBurnRequest | HTTP 200 with business code 400 (verified 2026-11 with no holdings: 'target does not exist, cannot refine'; irreversible — no real burn performed). |
| GET `/api/v2/vstock/crystal-leaderboard` | streamerUid:integer; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/flows` | window:string | ✅ 200 |
| GET `/api/v2/vstock/headlines` | - | ✅ 200 |
| GET `/api/v2/vstock/index/bars` | interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/leaderboard` | type:string; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/live/{liveId}/settlement` | liveId*:string | ✅ 200 |
| GET `/api/v2/vstock/market` | board:VStockBoard; agencyUuid:string; keyword:string; pageNum:integer; pageSize:integer; sortBy:string; area:string; uids:string; activeOnly:boolean | ✅ 200 |
| GET `/api/v2/vstock/portfolio` | - | ✅ 200 |
| POST `/api/v2/vstock/portfolio/claim-bankruptcy-relief` | - | HTTP 200 with business code 400 (verified 2026-11: at NAV 9,990 it replies 'below the bankruptcy-relief threshold — must be under 200'). |
| GET `/api/v2/vstock/portfolio/ledgers` | type:VStockLedgerType; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/vstock/portfolio/position-stats` | - | ✅ 200 |
| POST `/api/v2/vstock/predictions/bet` | body:VStockPredictionBetRequest | ✅ 200 (verified 2026-11: a minimum-stake bet of 10 D-coins was placed successfully). |
| GET `/api/v2/vstock/predictions/history` | limit:integer | ✅ 200 |
| GET `/api/v2/vstock/predictions/open` | - | ✅ 200 |
| GET `/api/v2/vstock/pulse` | - | ✅ 200 |
| GET `/api/v2/vstock/tape` | - | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/bars` | uid*:integer; interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/flow` | uid*:integer; window:string | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/history` | uid*:integer; days:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer; body:VStockCommentPinRequest | Validation + really called, business-blocked (2026-11-04): a flat `{}` probe returns validation 400 (commentId must be a real Guid); in round 2, calling with a real Guid in the flat body binds normally and the block becomes the business rule 'you can only pin your own comments on this ticker' (business code 400) — and owning a comment is itself blocked by the comments group's CAPTCHA (see above), so the pin chain stops there |
| GET `/api/v2/vstock/ticker/{uid}/predictions` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/settlements` | uid*:integer; limit:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/sync` | uid*:integer | ✅ 200 (verified 2026-11: triggers a sync and returns fresh ticker data). |
| POST `/api/v2/vstock/trade` | body:VStockTradeRequest | Verified live (side 1/2). Market order, fills instantly. |
| POST `/api/v2/vstock/wiki-tag` | body:SubmitVtbIdentityTagRequest | Caution: an empty-body probe already returns 200 'tag application submitted to the wiki review pool' — the server does not validate defaults (a streamerUid=0 application gets submitted). Always send full parameters. (Follow-up: the round-1 stray streamerUid=0 submission has been withdrawn by the submitter.) |

## Notes on non-200 results from the verification pass

- **Write-endpoint verification pass (2026-11-04)**: all 48 previously untested write endpoints were called one by one — 15 real-call successes; 5 business/permission rejections (sync-vdb and wiki governance are 403 permission checks; equip, burn, and bankruptcy relief return business code 400); the analysis-job DELETE is a routing-level 404 (below); 26 validation-only probes (public-content endpoints were probed for reachability only, no real content submitted); 1 endpoint (POST core-runtime/heartbeat) could not be reached because the credential proxy errors on that exact path.
- **DELETE /api/v3/user-analysis-jobs/{jobId}**: returns a framework-level 404 even for a freshly created job that GET can read and that has completed — the DELETE route appears unimplemented server-side. Creating jobs (POST) works (200).
- **POST /api/v2/vstock/ticker/{uid}/pin**: round 1 read this as a schema mismatch, but round 2 retested with a real Guid: the flat `{commentId, bid}` body binds normally — the `req`-required error was a model-binding cascade caused by a fake non-Guid value; the real block is the business rule 'you can only pin your own comments on this ticker'.
- **POST /api/v2/vstock/wiki-tag**: an empty body already returns 200 and submits a defaulted application (streamerUid=0) to the wiki review pool — this endpoint performs no required-field validation; callers must ensure parameters are complete. (Follow-up: the stray submission has been withdrawn by the submitter and no longer sits in the review pool.)
- **Write-endpoint round-2 live tests (2026-11-04)**: endpoints that round 1 only validation-probed were upgraded to real calls — 10 wiki/agency proposal endpoints really submitted and were all withdrawn for cleanup (including the agencies avatar intent→upload→complete flow and a timeline reaction on/off toggle loop); account/update-avatar completed intent→upload→update (avatar really changed; no API to roll back); core-runtime/heartbeat confirmed 204 via raw response reads (round 1's failure was an empty body misread by JSON parsing); the whole comment family (comments, comment/add, user/comment/add, chat/add, live/comment/add) is blocked by human verification (business code 403; the token is a web CAPTCHA token with no issuing API endpoint); proposals vote/emergency/appeal are blocked by locked voting rights, moderator-only permission, and appeal preconditions respectively; lineage/submit and timeline dispute are blocked by the 'strong-consensus actions require community Resident rank' rule; ticker pin with a real Guid confirms the block is 'you can only pin your own comments' (and creating one is itself CAPTCHA-blocked); upload-extracted's payload format is undisclosed (six constructions all failed to parse); DELETE /comments/{id} could not be tested (no own comment can be created, others' comments are not deleted) and neither could lineage dispute (no lineage records exist on the site).
- **Contribution group**: returned 503 across the board during one sweep (transient server state); on re-test, `areas`, `me`, `minute-today`, `hub`, `overview`, `rank/day` and `rank/range` all returned 200. `GET /api/v2/contribution/contributor` returns 400 for a plain account id — its `id` parameter is not the account id (semantics undetermined).
- **400 validation** on `GET /api/v2/comments`, `GET /api/v2/identity/nameplates`, `GET /api/v2/wiki/content`: these need concrete target parameters (comment target, nameplate id set, content-block scope key such as `channel.bio` / `agency.description`) that a generic probe does not supply.
- **404** on `GET /api/v3/user-analysis-jobs/{jobId}*`: expected when probing with a non-existent job id. `GET /api/v3/lives/{liveId}/delta` also returned 404 while sibling resources for the same live id worked.
- **204** on `GET /api/v2/core-runtime/heartbeat` and **400** on `GET /api/v2/core-runtime/ws`: node-runtime probe / WebSocket upgrade endpoints, not plain REST reads.
- **403** on `GET /api/v2/wiki/governance/accounts/{accountId}`: moderator-only data; the permission check works as intended.
- **Platform parameter**: wiki/lineage endpoints expect `platform` to be one of `bilibili / acfun / youtube / twitch / douyin`.
- **State-changing GETs**: `GET /api/v2/account/add-record` and `GET /api/v2/account/del-record` mutate the recording list despite being GETs. Exclude them from any automated read-only sweep.
- **`GET /api/v2/user` in the wild**: the spec declares `uid` and `target` as required, but live checks (2026-11-04) show `target` may be omitted — it then behaves as `-1` (all channels) — and the user-id parameter also answers to the `uId` spelling. Third-party code in the wild, e.g. the Go library `NFS688/AnimeAPI`, calls it as `/api/v2/user?uId=<uid>&pageNum=0&pageSize=5&target=-1&useEmoji=true` to fetch a user's danmaku history; `target` filters by channel (streamer) UID.

## Key response shapes (observed)

- **VStockTickerDto** (`market`, `ticker/{uid}`): `streamerUid, userName, board (0–3), status, nav, pbRatio, currentPrice, previousClosePrice, emaFundamentalPoints, liquidityDepth, longShares, shortShares, liveCount30d, danmakuCount30d, income30d, isLiving, sparkline, topLongHolders, topShortHolders`.
- **VStockAccountDto** (`portfolio`): `accountId, seasonId, cashBalance, totalLongMarketValue, totalShortLiability, totalShortMarginLocked, netAssetValue, longPositions[] {shares, avgCostPrice, currentPrice, marketValue, unrealizedPnl, weightedAcquiredAt, shortHoldShares}, shortPositions[]`.
- **VStockLedgerDto** (`portfolio/ledgers`): `type` is an enum 1–15 (observed: 4 = trade, 5 = stream-end dividend); `note` is a human-readable Chinese description; plus `deltaCash, balanceAfter, realizedPnl, liveId, createdAt`.
- **VStockDividendSettlementDto** (`ticker/{uid}/settlements`): `liveId, liveFundamentalPoints, dividendPerShare, totalLongDividendPaid, totalShortDividendCharged, navAfter, boardAfter, promotedToMainBoard, settledAt`.
- **Search points** (`account/search-points`): `{earned, spent, available, eventsPerPoint}` — points accrue at 1 per 5,000 valid interactions (`eventsPerPoint: 5000`).
- **User danmaku history** (`GET /api/v2/user`): the paginated envelope's inner `data.records[]` items are `{channel{...}, live{...}, danmakus[]}` — one record per live session; `danmakus[]` holds the queried user's messages in that session (`uId, uName, type, sendDate`, plus `message` / `price` on text and paid entries).

## Etiquette

- Keep request rates close to normal app usage; the API is a community-run service.
- Never commit a real Access Token to a repository. Treat it like a password.
