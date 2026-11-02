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
Both definitions currently expose the **same** paths and schemas (156 paths, 163 operations); the endpoints grouped as new in the v3 UI still live under `/api/v2/` paths.

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
| 3 | Open short (`sell_short`) | inferred from enum order + ledger wording, untested |
| 4 | Close short (`buy_short`) | inferred, untested |

It is a **market order**: it fills immediately at the vAMM price and returns
`{executionPrice, grossAmount, feeAmount, realizedPnl, priceAfter, cashBalanceAfter}`.
Fees observed live: **0.2%** on buys; a sell within 12 hours of acquisition is charged **1.0%** total.
There is no order book and no cancel step — check `GET /api/v2/vstock/ticker/{uid}` (price, `pbRatio`) and your `portfolio` before sending a trade.

Two write endpoints deserve explicit caution:

- `POST /api/v2/vstock/burn` — permanently burns shares into crystal points (lock-up related). Irreversible.
- `POST /api/v2/vstock/portfolio/claim-bankruptcy-relief` — claims bankruptcy relief for the account.

## Endpoint reference

Status key: ✅ 200 = called successfully during the November 2026 verification pass (114 GET endpoints, 102 returned 200).
`HTTP xxx (see Notes)` = the observed status for that call (server 503s, parameter validation 400s, or not-found probes). Required parameters are marked with `*`.

### Account

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/account/add-record` | uId:integer | ✅ 200 |
| POST `/api/v2/account/avatar-intent` | body:AvatarUploadIntentRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/account/channel-lives` | uId:integer; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/contributed-lives` | pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/core-config` | - | ✅ 200 |
| POST `/api/v2/account/core-config` | body:CoreControlConfig | Not tested in this pass (state-changing write). |
| GET `/api/v2/account/del-record` | uId:integer | ✅ 200 |
| GET `/api/v2/account/info` | id:integer | ✅ 200 |
| GET `/api/v2/account/recording` | - | ✅ 200 |
| GET `/api/v2/account/search-points` | - | ✅ 200 |
| GET `/api/v2/account/search-points/history` | page:integer; pageSize:integer; filter:string | ✅ 200 |
| GET `/api/v2/account/statistics` | start:string; end:string | ✅ 200 |
| POST `/api/v2/account/statistics/report` | body:ReportStatisticRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/account/update-avatar` | body:RequestUpdateAvatar | Not tested in this pass (state-changing write). |
| POST `/api/v2/account/update-name` | body:RequestUpdateUserName | Not tested in this pass (state-changing write). |
| POST `/api/v2/account/update-public-profile` | body:RequestUpdateContributorPublicProfile | Not tested in this pass (state-changing write). |
| POST `/api/v2/account/update-recording-setting` | - | Not tested in this pass (state-changing write). |

### AgencyWiki

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/agencies` | keyword:string; type:AgencyGroupType; pageNum:integer; pageSize:integer; sort:string | ✅ 200 |
| POST `/api/v2/agencies/avatar-complete` | body:CompleteAgencyAvatarRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/agencies/avatar-intent` | body:AvatarUploadIntentRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/agencies/propose` | body:SubmitAgencyUpsertRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/agencies/sync-vdb` | - | Not tested in this pass (state-changing write). |
| GET `/api/v2/agencies/{uuid}` | uuid*:string | ✅ 200 |
| POST `/api/v2/agencies/{uuid}/propose` | uuid*:string; body:SubmitAgencyUpsertRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/agency-options` | - | ✅ 200 |
| GET `/api/v2/agency/{uuid}` | uuid*:string | ✅ 200 |
| GET `/api/v2/agency/{uuid}/timeline` | uuid*:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/{userId}/lineage` | userId*:integer; platform:string; includeSuspected:boolean | ✅ 200 |
| GET `/api/v2/channel/{userId}/timeline` | userId*:integer; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/affiliation/submit` | body:SubmitChannelAffiliationRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/wiki/channel/{userId}/career` | userId*:integer | ✅ 200 |
| GET `/api/v2/wiki/channel/{userId}/lives` | userId*:integer; keyword:string; limit:integer | ✅ 200 |
| GET `/api/v2/wiki/content` | scopeKey:string; targetId:string | HTTP 400 (see Notes) |
| POST `/api/v2/wiki/content/revise` | body:SubmitWikiContentRevisionRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer | HTTP 403 (see Notes) |
| POST `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer; body:UpdateWikiAccountStateRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/wiki/governance/profile` | - | ✅ 200 |
| GET `/api/v2/wiki/lineage` | platform:string; identity:string; includeSuspected:boolean | ✅ 200 |
| POST `/api/v2/wiki/lineage/submit` | body:SubmitLineageRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/lineage/{lineageId}/dispute` | lineageId*:string; body:SubmitWikiDisputeRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/wiki/proposals` | status:WikiProposalStatus; agencyUuid:string; streamerUid:integer; proposalType:WikiProposalType; scopeKey:string; mine:boolean; votedByMe:boolean; appealed:boolean; aiOutcome:WikiAiReviewOutcome; sort:WikiProposalSort; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/proposals/{proposalId}/appeal` | proposalId*:string; body:AppealWikiProposalRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/proposals/{proposalId}/emergency` | proposalId*:string; body:EmergencyWikiProposalRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/proposals/{proposalId}/vote` | proposalId*:string; body:VoteWikiProposalRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/proposals/{proposalId}/withdraw` | proposalId*:string; body:WithdrawWikiProposalRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/wiki/timeline` | userId:integer; agencyUuid:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/timeline/submit` | body:SubmitWikiNodeRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/timeline/{eventId}/reaction` | eventId*:string; body:ToggleWikiReactionRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/timeline/{nodeId}/dispute` | nodeId*:string; body:SubmitWikiDisputeRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/wiki/timeline/{nodeId}/revise` | nodeId*:string; diffSummary:string; body:SubmitWikiNodeRequest | Not tested in this pass (state-changing write). |

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
| GET `/api/v2/core-runtime/extraction-rules` | - | ✅ 200 |
| GET `/api/v2/core-runtime/heartbeat` | clientId:string | HTTP 204 (see Notes) |
| POST `/api/v2/core-runtime/heartbeat` | body:CoreHeartbeatRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/core-runtime/request-room` | body:RequestRoomRequest | Not tested in this pass (state-changing write). |
| DELETE `/api/v2/core-runtime/state` | clientId:string | Not tested in this pass (state-changing write). |
| POST `/api/v2/core-runtime/sync` | body:CoreStateSyncRequest | Not tested in this pass (state-changing write). |
| POST `/api/v2/core-runtime/upload-extracted` | - | Not tested in this pass (state-changing write). |
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
| POST `/api/v2/chat/add` | body:ChatModel | Not tested in this pass (state-changing write). |
| GET `/api/v2/chat/get` | date:integer; count:integer | ✅ 200 |
| POST `/api/v2/comment/add` | uId:integer; body:AddCommentRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/comments` | targetType:SiteCommentTargetType; targetId:string | HTTP 400 (see Notes) |
| POST `/api/v2/comments` | body:AddSiteCommentRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/comments/counts` | targetType:SiteCommentTargetType; ids:string | HTTP 400 (see Notes) |
| GET `/api/v2/comments/stream` | viewerId:string | ✅ 200 |
| DELETE `/api/v2/comments/{id}` | id*:string | Not tested in this pass (state-changing write). |
| GET `/api/v2/emoicons` | - | ✅ 200 |
| GET `/api/v2/guilds` | - | ✅ 200 |
| GET `/api/v2/live` | liveId*:string; uId:integer; type:array; pageNum:integer; pageSize:integer; includeExtra:boolean; useEmoji:boolean; includeDanmakus:boolean | ✅ 200 |
| POST `/api/v2/live/comment/add` | body:AddLiveCommentRequest | Not tested in this pass (state-changing write). |
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
| POST `/api/v2/user/comment/add` | body:AddUserCommentRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/user/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/user/name` | uid*:array | ✅ 200 |
| GET `/api/v2/user/watchedChannels` | uid*:integer; simple:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/user/watchedChannelsSimple` | uid*:integer; distinct:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/vup-list` | - | ✅ 200 |

### DanmakuSearch

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/danmaku/search` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200 |
| POST `/api/v2/danmaku/search/async` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | Not tested in this pass (creates an async search task). |
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
| POST `/api/v2/identity/equip` | body:IdentityEquipRequest | Not tested in this pass (state-changing write). |
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
| POST `/api/v3/user-analysis-jobs` | body:CreateUserAnalyzeJobRequest | Not tested in this pass (creates a server-side job). |
| DELETE `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | Not tested in this pass (state-changing write). |
| GET `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404 (see Notes) |
| GET `/api/v3/user-analysis-jobs/{jobId}/events` | jobId*:string | HTTP 404 (see Notes) |
| GET `/api/v3/user-analysis-jobs/{jobId}/result` | jobId*:string | HTTP 404 (see Notes) |

### VStock

| Method & path | Parameters | Status / notes |
| --- | --- | --- |
| GET `/api/v2/vstock/agencies` | sort:string | ✅ 200 |
| GET `/api/v2/vstock/agency/{uuid}/index` | uuid*:string | ✅ 200 |
| POST `/api/v2/vstock/burn` | body:VStockBurnRequest | Not tested in this pass (irreversible; burns shares for crystal points). |
| GET `/api/v2/vstock/crystal-leaderboard` | streamerUid:integer; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/flows` | window:string | ✅ 200 |
| GET `/api/v2/vstock/headlines` | - | ✅ 200 |
| GET `/api/v2/vstock/index/bars` | interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/leaderboard` | type:string; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/live/{liveId}/settlement` | liveId*:string | ✅ 200 |
| GET `/api/v2/vstock/market` | board:VStockBoard; agencyUuid:string; keyword:string; pageNum:integer; pageSize:integer; sortBy:string; area:string; uids:string; activeOnly:boolean | ✅ 200 |
| GET `/api/v2/vstock/portfolio` | - | ✅ 200 |
| POST `/api/v2/vstock/portfolio/claim-bankruptcy-relief` | - | Not tested in this pass. |
| GET `/api/v2/vstock/portfolio/ledgers` | type:VStockLedgerType; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/vstock/portfolio/position-stats` | - | ✅ 200 |
| POST `/api/v2/vstock/predictions/bet` | body:VStockPredictionBetRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/vstock/predictions/history` | limit:integer | ✅ 200 |
| GET `/api/v2/vstock/predictions/open` | - | ✅ 200 |
| GET `/api/v2/vstock/pulse` | - | ✅ 200 |
| GET `/api/v2/vstock/tape` | - | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/bars` | uid*:integer; interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/flow` | uid*:integer; window:string | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/history` | uid*:integer; days:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer; body:VStockCommentPinRequest | Not tested in this pass (state-changing write). |
| GET `/api/v2/vstock/ticker/{uid}/predictions` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/settlements` | uid*:integer; limit:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/sync` | uid*:integer | Not tested in this pass (state-changing write). |
| POST `/api/v2/vstock/trade` | body:VStockTradeRequest | Verified live (side 1/2). Market order, fills instantly. |
| POST `/api/v2/vstock/wiki-tag` | body:SubmitVtbIdentityTagRequest | Not tested in this pass (state-changing write). |

## Notes on non-200 results from the verification pass

- **Contribution group**: returned 503 across the board during one sweep (transient server state); on re-test, `areas`, `me`, `minute-today`, `hub`, `overview`, `rank/day` and `rank/range` all returned 200. `GET /api/v2/contribution/contributor` returns 400 for a plain account id — its `id` parameter is not the account id (semantics undetermined).
- **400 validation** on `GET /api/v2/comments`, `GET /api/v2/identity/nameplates`, `GET /api/v2/wiki/content`: these need concrete target parameters (comment target, nameplate id set, content-block scope key such as `channel.bio` / `agency.description`) that a generic probe does not supply.
- **404** on `GET /api/v3/user-analysis-jobs/{jobId}*`: expected when probing with a non-existent job id. `GET /api/v3/lives/{liveId}/delta` also returned 404 while sibling resources for the same live id worked.
- **204** on `GET /api/v2/core-runtime/heartbeat` and **400** on `GET /api/v2/core-runtime/ws`: node-runtime probe / WebSocket upgrade endpoints, not plain REST reads.
- **403** on `GET /api/v2/wiki/governance/accounts/{accountId}`: moderator-only data; the permission check works as intended.
- **Platform parameter**: wiki/lineage endpoints expect `platform` to be one of `bilibili / acfun / youtube / twitch / douyin`.

## Key response shapes (observed)

- **VStockTickerDto** (`market`, `ticker/{uid}`): `streamerUid, userName, board (0–3), status, nav, pbRatio, currentPrice, previousClosePrice, emaFundamentalPoints, liquidityDepth, longShares, shortShares, liveCount30d, danmakuCount30d, income30d, isLiving, sparkline, topLongHolders, topShortHolders`.
- **VStockAccountDto** (`portfolio`): `accountId, seasonId, cashBalance, totalLongMarketValue, totalShortLiability, totalShortMarginLocked, netAssetValue, longPositions[] {shares, avgCostPrice, currentPrice, marketValue, unrealizedPnl, weightedAcquiredAt, shortHoldShares}, shortPositions[]`.
- **VStockLedgerDto** (`portfolio/ledgers`): `type` is an enum 1–15 (observed: 4 = trade, 5 = stream-end dividend); `note` is a human-readable Chinese description; plus `deltaCash, balanceAfter, realizedPnl, liveId, createdAt`.
- **VStockDividendSettlementDto** (`ticker/{uid}/settlements`): `liveId, liveFundamentalPoints, dividendPerShare, totalLongDividendPaid, totalShortDividendCharged, navAfter, boardAfter, promotedToMainBoard, settledAt`.
- **Search points** (`account/search-points`): `{earned, spent, available, eventsPerPoint}` — points accrue at 1 per 5,000 valid interactions (`eventsPerPoint: 5000`).

## Etiquette

- Keep request rates close to normal app usage; the API is a community-run service.
- Never commit a real Access Token to a repository. Treat it like a password.
