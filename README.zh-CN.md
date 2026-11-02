# Danmakus API 指南（非官方 · 中文版）

English version: [README.md](README.md)

这是一份社区维护的 [Danmakus](https://danmakus.com) API 非官方指南，依据官方 Swagger 定义整理，并于 2026 年 11 月对全部接口做了实测核对。

> **免责声明**：本文档非官方出品，接口可能随时变更。标注 ✅ 的行为结论均来自真实调用观察。

## 节点地址

由 `GET /api/v2/meta/nodes` 实测返回：

| 节点 | 地址 |
| --- | --- |
| 默认线路 | `https://api.ukamnads.icu` |
| 备用线路 1 | `https://ukamnads.icu` |
| 备用线路 2（香港） | `https://api.danmakus.com`（部分出口 IP 会被 Cloudflare 拦截，错误 1010） |

Swagger UI：`https://api.ukamnads.icu/swagger/index.html`，包含 `v2` 与 `v3` 两套定义。两套定义的路径与结构目前**完全相同**（156 条路径、163 个操作）；v3 界面里标为新增的分组，其接口路径实际仍在 `/api/v2/` 下。

> ⚠️ 官方发布的 `swagger.json` 不是严格合法的 JSON：有一处示例里出现空值（`"kindValue": ,`）。用严格解析器或代码生成器处理前，需先把这类空值清洗为 `null`。

## 鉴权

公开的行情/榜单接口免登录；账户接口使用**自定义请求头**：

```
Token: <你的 Access Token>
```

实测对照：

| 带法 | 结果 |
| --- | --- |
| `Authorization: Bearer <token>` | `401` 请先登录账号 |
| 不带头 | `401` 未登录 |
| `Token: <错误值>` | `403` 无效的登录凭据 |
| `Token: <Access Token 原值>` | `200` |

Access Token 即网页登录 danmakus.com 时使用的那一条（网页端把它存在 `localStorage` 的键 `Token` 里，原值发送）。早期第三方客户端用 `POST /api/v2/accounts/token` 换取会话 token 的做法已失效：该接口在两个节点上均返回 404，当前规范中也不存在，不要再基于它开发。

## 通用约定

- **响应包**：多数接口返回 `{"code": 200, "message": "", "data": ...}`；少数返回裸数组/裸对象（如 `POST /api/v2/streamer-status`、`GET /api/v2/meta/nodes`）。
- **分页**：`pageNum` 从 0 起；列表返回 `{total, pageNum, pageSize, hasMore, data[]}`。
- **时间字段**：ISO 8601 为 UTC；`settledAt`、`generatedAt` 为 Unix **秒**；`timestamp`、`startDate` 等为 Unix **毫秒**。
- **ID 口径**：主播以 B 站 UID 为键（`streamerUid`，int64）。频道详情用 `GET /api/v2/channel?uid=<B站UID>`；另有一组路径参数为频道内部 UUID 的接口，那个 UUID 不是 UID。

## 快速开始

```bash
TOKEN='<你的 Access Token>'

# 大盘脉搏（免登录）
curl -s https://api.ukamnads.icu/api/v2/vstock/pulse

# 我的持仓（需鉴权）
curl -s https://api.ukamnads.icu/api/v2/vstock/portfolio -H "Token: $TOKEN"

# 单只股票
curl -s https://api.ukamnads.icu/api/v2/vstock/ticker/1932862336
```

## V股交易

`POST /api/v2/vstock/trade`，请求体只有三个字段：

```json
{"streamerUid": 631699988, "side": 1, "shares": 10}
```

`side` 为枚举 `VStockTradeSide`（1–4）：

| side | 含义 | 状态 |
| --- | --- | --- |
| 1 | 买入做多（buy_long） | ✅ 已实测 |
| 2 | 卖出平多（sell_long） | ✅ 已实测 |
| 3 | 开空（sell_short） | ✅ 已实测 |
| 4 | 平空（buy_short） | ✅ 已实测 |

该接口是**市价单**：提交即按 vAMM 价格成交，返回 `{executionPrice, grossAmount, feeAmount, realizedPnl, priceAfter, cashBalanceAfter}`。实测费率：买入与开空均为 **0.2%**；未满 12 小时卖出平多或平空，总费率 **1.0%**。开空会锁定名义值 **150%** 的保证金（平空后全额释放）；同一标的持有多头时禁止开空，接口返回 400 并明确提示须先卖出平多。没有挂单与撤单步骤，下单前请先用 `GET /api/v2/vstock/ticker/{uid}` 核对现价与 `pbRatio`、用 `portfolio` 核对现金与持仓。

两个写接口请特别谨慎：

- `POST /api/v2/vstock/burn`：把股份永久销毁换结晶点（与锁仓相关），不可逆。
- `POST /api/v2/vstock/portfolio/claim-bankruptcy-relief`：领取破产救济。

## 接口清单

状态说明：✅ 200 = 2026 年 11 月实测通过（GET 共 114 个，102 个返回 200）；`HTTP xxx（见文末说明）` = 该次实测的实际状态。参数带 `*` 为必填。

### Account

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/account/add-record` | uId:integer | ✅ 200 |
| POST `/api/v2/account/avatar-intent` | body:AvatarUploadIntentRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/account/channel-lives` | uId:integer; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/contributed-lives` | pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/core-config` | - | ✅ 200 |
| POST `/api/v2/account/core-config` | body:CoreControlConfig | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/account/del-record` | uId:integer | ✅ 200 |
| GET `/api/v2/account/info` | id:integer | ✅ 200 |
| GET `/api/v2/account/recording` | - | ✅ 200 |
| GET `/api/v2/account/search-points` | - | ✅ 200 |
| GET `/api/v2/account/search-points/history` | page:integer; pageSize:integer; filter:string | ✅ 200 |
| GET `/api/v2/account/statistics` | start:string; end:string | ✅ 200 |
| POST `/api/v2/account/statistics/report` | body:ReportStatisticRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/account/update-avatar` | body:RequestUpdateAvatar | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/account/update-name` | body:RequestUpdateUserName | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/account/update-public-profile` | body:RequestUpdateContributorPublicProfile | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/account/update-recording-setting` | - | 本轮未测（会改变状态的写接口）。 |

### AgencyWiki

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/agencies` | keyword:string; type:AgencyGroupType; pageNum:integer; pageSize:integer; sort:string | ✅ 200 |
| POST `/api/v2/agencies/avatar-complete` | body:CompleteAgencyAvatarRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/agencies/avatar-intent` | body:AvatarUploadIntentRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/agencies/propose` | body:SubmitAgencyUpsertRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/agencies/sync-vdb` | - | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/agencies/{uuid}` | uuid*:string | ✅ 200 |
| POST `/api/v2/agencies/{uuid}/propose` | uuid*:string; body:SubmitAgencyUpsertRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/agency-options` | - | ✅ 200 |
| GET `/api/v2/agency/{uuid}` | uuid*:string | ✅ 200 |
| GET `/api/v2/agency/{uuid}/timeline` | uuid*:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/{userId}/lineage` | userId*:integer; platform:string; includeSuspected:boolean | ✅ 200 |
| GET `/api/v2/channel/{userId}/timeline` | userId*:integer; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/affiliation/submit` | body:SubmitChannelAffiliationRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/wiki/channel/{userId}/career` | userId*:integer | ✅ 200 |
| GET `/api/v2/wiki/channel/{userId}/lives` | userId*:integer; keyword:string; limit:integer | ✅ 200 |
| GET `/api/v2/wiki/content` | scopeKey:string; targetId:string | HTTP 400（见文末说明） |
| POST `/api/v2/wiki/content/revise` | body:SubmitWikiContentRevisionRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer | HTTP 403（见文末说明） |
| POST `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer; body:UpdateWikiAccountStateRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/wiki/governance/profile` | - | ✅ 200 |
| GET `/api/v2/wiki/lineage` | platform:string; identity:string; includeSuspected:boolean | ✅ 200 |
| POST `/api/v2/wiki/lineage/submit` | body:SubmitLineageRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/lineage/{lineageId}/dispute` | lineageId*:string; body:SubmitWikiDisputeRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/wiki/proposals` | status:WikiProposalStatus; agencyUuid:string; streamerUid:integer; proposalType:WikiProposalType; scopeKey:string; mine:boolean; votedByMe:boolean; appealed:boolean; aiOutcome:WikiAiReviewOutcome; sort:WikiProposalSort; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/proposals/{proposalId}/appeal` | proposalId*:string; body:AppealWikiProposalRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/proposals/{proposalId}/emergency` | proposalId*:string; body:EmergencyWikiProposalRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/proposals/{proposalId}/vote` | proposalId*:string; body:VoteWikiProposalRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/proposals/{proposalId}/withdraw` | proposalId*:string; body:WithdrawWikiProposalRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/wiki/timeline` | userId:integer; agencyUuid:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/timeline/submit` | body:SubmitWikiNodeRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/timeline/{eventId}/reaction` | eventId*:string; body:ToggleWikiReactionRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/timeline/{nodeId}/dispute` | nodeId*:string; body:SubmitWikiDisputeRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/wiki/timeline/{nodeId}/revise` | nodeId*:string; diffSummary:string; body:SubmitWikiNodeRequest | 本轮未测（会改变状态的写接口）。 |

### AnonymousDanmakuIdentity

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}/danmakus-with-identities` | liveId*:string; offset:integer; limit:integer; type:array; includeEnter:boolean | ✅ 200 |

### ArchivedDanmakuCursor

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}/danmakus/after` | liveId*:string; limit:integer; afterTs:integer; afterSeq:integer; type:array | ✅ 200 |

### AudienceLeaderboard

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/channel/audience-leaderboard` | uId:integer; board:string; range:string; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/audience-trend` | uId:integer | ✅ 200 |
| GET `/api/v2/channel/compare` | uids:string; range:string | ✅ 200 |
| GET `/api/v2/channel/compare/presets` | - | ✅ 200 |
| GET `/api/v2/rank/audience/day` | date:string; board:string | ✅ 200 |

### Contribution

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/contribution/areas` | range:integer; topUsersPerArea:integer; areaCount:integer | ✅ 200 |
| GET `/api/v2/contribution/contributor` | accountId*:integer | HTTP 400（见文末说明） |
| GET `/api/v2/contribution/hub` | range:integer; metric:ContributionMetric; top:integer; topUsersPerArea:integer; areaCount:integer | ✅ 200 |
| GET `/api/v2/contribution/me` | start:string; end:string | ✅ 200 |
| GET `/api/v2/contribution/minute-today` | - | ✅ 200 |
| GET `/api/v2/contribution/overview` | range:integer; top:integer; metric:ContributionMetric | ✅ 200 |
| GET `/api/v2/contribution/rank/day` | year:integer; month:integer; day:integer; metric:ContributionMetric; count:integer | ✅ 200 |
| GET `/api/v2/contribution/rank/range` | range:ContributionRange; metric:ContributionMetric; count:integer | ✅ 200 |

### CoreRuntime

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/core-runtime/clients` | - | ✅ 200 |
| GET `/api/v2/core-runtime/extraction-rules` | - | ✅ 200 |
| GET `/api/v2/core-runtime/heartbeat` | clientId:string | HTTP 204（见文末说明） |
| POST `/api/v2/core-runtime/heartbeat` | body:CoreHeartbeatRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/core-runtime/request-room` | body:RequestRoomRequest | 本轮未测（会改变状态的写接口）。 |
| DELETE `/api/v2/core-runtime/state` | clientId:string | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/core-runtime/sync` | body:CoreStateSyncRequest | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/core-runtime/upload-extracted` | - | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/core-runtime/ws` | - | HTTP 400（见文末说明） |

### DanmakuControllers

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/add` | uid*:integer; token*:string | ✅ 200 |
| GET `/api/v2/area` | - | ✅ 200 |
| GET `/api/v2/channel` | uId*:integer; includeLive:boolean; includeExtra:boolean; maxLives:integer; livesOffset:integer; includeFansHistory:boolean | ✅ 200 |
| GET `/api/v2/channel/filter` | type:ChannelRankType; range:integer; keyword:string; area:string; pageNum:integer; pageSize:integer; includeExtra:boolean | ✅ 200 |
| POST `/api/v2/channel/filter` | body:PostChannelInfo | 已实测。查询型 POST。 |
| GET `/api/v2/channels` | simple:boolean | ✅ 200 |
| POST `/api/v2/chat/add` | body:ChatModel | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/chat/get` | date:integer; count:integer | ✅ 200 |
| POST `/api/v2/comment/add` | uId:integer; body:AddCommentRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/comments` | targetType:SiteCommentTargetType; targetId:string | HTTP 400（见文末说明） |
| POST `/api/v2/comments` | body:AddSiteCommentRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/comments/counts` | targetType:SiteCommentTargetType; ids:string | HTTP 400（见文末说明） |
| GET `/api/v2/comments/stream` | viewerId:string | ✅ 200 |
| DELETE `/api/v2/comments/{id}` | id*:string | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/emoicons` | - | ✅ 200 |
| GET `/api/v2/guilds` | - | ✅ 200 |
| GET `/api/v2/live` | liveId*:string; uId:integer; type:array; pageNum:integer; pageSize:integer; includeExtra:boolean; useEmoji:boolean; includeDanmakus:boolean | ✅ 200 |
| POST `/api/v2/live/comment/add` | body:AddLiveCommentRequest | 本轮未测（会改变状态的写接口）。 |
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
| POST `/api/v2/user/comment/add` | body:AddUserCommentRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/user/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/user/name` | uid*:array | ✅ 200 |
| GET `/api/v2/user/watchedChannels` | uid*:integer; simple:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/user/watchedChannelsSimple` | uid*:integer; distinct:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/vup-list` | - | ✅ 200 |

### DanmakuSearch

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/danmaku/search` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200 |
| POST `/api/v2/danmaku/search/async` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | 本轮未测（会创建异步检索任务）。 |
| GET `/api/v2/danmaku/search/async/{taskId}` | taskId*:string | ✅ 200 |
| GET `/api/v2/danmaku/search/queue-stats` | - | ✅ 200 |

### DanmakuV3

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v3/lives/{liveId}` | liveId*:string; include:string | ✅ 200 |
| GET `/api/v3/lives/{liveId}/danmakus` | liveId*:string; offset:integer; limit:integer; userId:integer; type:array | ✅ 200 |
| GET `/api/v3/lives/{liveId}/delta` | liveId*:string; afterSeq:integer; includeEnter:boolean | HTTP 404（见文末说明） |
| GET `/api/v3/lives/{liveId}/full` | liveId*:string; include:string; includeEnter:boolean; offset:integer; limit:integer | ✅ 200 |
| GET `/api/v3/meta/emoicons` | - | ✅ 200 |
| GET `/api/v3/meta/index-background` | afterTs:integer; take:integer | ✅ 200 |
| GET `/api/v3/users/{userId}/bootstrap` | userId*:integer; page:integer; pageSize:integer; channelUserId:integer; hideOnlyEnter:boolean; maxChannels:integer | ✅ 200 |
| GET `/api/v3/users/{userId}/history` | userId*:integer; page:integer; pageSize:integer; channelUserId:integer; hideOnlyEnter:boolean | ✅ 200 |

### Identity

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/identity/account/{accountId}` | accountId*:integer | ✅ 200 |
| GET `/api/v2/identity/catalog` | - | ✅ 200 |
| POST `/api/v2/identity/equip` | body:IdentityEquipRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/identity/nameplates` | ids:string; streamerUid:integer | HTTP 400（见文末说明） |

### Meta

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/meta/nodes` | - | ✅ 200 |
| GET `/api/v2/meta/ping` | - | ✅ 200 |

### StreamerStatus

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| POST `/api/v2/streamer-status` | body:StreamerStatusRequest | 已实测。查询型 POST。 |

### UserAnalyzeJobs

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| POST `/api/v3/user-analysis-jobs` | body:CreateUserAnalyzeJobRequest | 本轮未测（会创建服务端分析作业）。 |
| DELETE `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404（见文末说明） |
| GET `/api/v3/user-analysis-jobs/{jobId}/events` | jobId*:string | HTTP 404（见文末说明） |
| GET `/api/v3/user-analysis-jobs/{jobId}/result` | jobId*:string | HTTP 404（见文末说明） |

### VStock

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/vstock/agencies` | sort:string | ✅ 200 |
| GET `/api/v2/vstock/agency/{uuid}/index` | uuid*:string | ✅ 200 |
| POST `/api/v2/vstock/burn` | body:VStockBurnRequest | 本轮未测（不可逆操作：销毁股份换结晶）。 |
| GET `/api/v2/vstock/crystal-leaderboard` | streamerUid:integer; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/flows` | window:string | ✅ 200 |
| GET `/api/v2/vstock/headlines` | - | ✅ 200 |
| GET `/api/v2/vstock/index/bars` | interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/leaderboard` | type:string; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/live/{liveId}/settlement` | liveId*:string | ✅ 200 |
| GET `/api/v2/vstock/market` | board:VStockBoard; agencyUuid:string; keyword:string; pageNum:integer; pageSize:integer; sortBy:string; area:string; uids:string; activeOnly:boolean | ✅ 200 |
| GET `/api/v2/vstock/portfolio` | - | ✅ 200 |
| POST `/api/v2/vstock/portfolio/claim-bankruptcy-relief` | - | 本轮未测（破产救济领取）。 |
| GET `/api/v2/vstock/portfolio/ledgers` | type:VStockLedgerType; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/vstock/portfolio/position-stats` | - | ✅ 200 |
| POST `/api/v2/vstock/predictions/bet` | body:VStockPredictionBetRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/vstock/predictions/history` | limit:integer | ✅ 200 |
| GET `/api/v2/vstock/predictions/open` | - | ✅ 200 |
| GET `/api/v2/vstock/pulse` | - | ✅ 200 |
| GET `/api/v2/vstock/tape` | - | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/bars` | uid*:integer; interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/flow` | uid*:integer; window:string | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/history` | uid*:integer; days:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer; body:VStockCommentPinRequest | 本轮未测（会改变状态的写接口）。 |
| GET `/api/v2/vstock/ticker/{uid}/predictions` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/settlements` | uid*:integer; limit:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/sync` | uid*:integer | 本轮未测（会改变状态的写接口）。 |
| POST `/api/v2/vstock/trade` | body:VStockTradeRequest | 已实测（side 1/2）。市价即时成交。 |
| POST `/api/v2/vstock/wiki-tag` | body:SubmitVtbIdentityTagRequest | 本轮未测（会改变状态的写接口）。 |

## 未通过项说明（实测口径）

- **贡献（Contribution）组**：某一轮扫描时整组返回 503（服务端临时状态）；随后复测 `areas`、`me`、`minute-today`、`hub`、`overview`、`rank/day`、`rank/range` 均已 200。`GET /api/v2/contribution/contributor` 传账号 ID 返回 400，其 `id` 参数并非账号 ID，口径待查。
- **400 参数校验**：`GET /api/v2/comments`、`GET /api/v2/identity/nameplates`、`GET /api/v2/wiki/content` 需要具体的业务参数（评论目标、名牌 id 集合、内容块 scopeKey 如 `channel.bio` / `agency.description`），通用探测未命中。
- **404**：`GET /api/v3/user-analysis-jobs/{jobId}*` 是用不存在的任务 ID 探测的预期结果；`GET /api/v3/lives/{liveId}/delta` 在同场其他子资源正常的情况下返回 404，疑该子资源当前节点未提供。
- **204 / 400**：`GET /api/v2/core-runtime/heartbeat` 返回 204（节点探针语义）；`GET /api/v2/core-runtime/ws` 是 WebSocket 升级端点，普通 GET 返回 400 属预期。
- **403**：`GET /api/v2/wiki/governance/accounts/{accountId}` 仅百科版主可看，权限校验正常生效。
- **平台参数**：wiki/lineage 一组接口的 `platform` 仅限 `bilibili / acfun / youtube / twitch / douyin`。
- **带副作用的 GET**：`GET /api/v2/account/add-record` 与 `GET /api/v2/account/del-record` 虽然是 GET，却会真实增删录制名单，任何只读盘点都要把它们当写接口排除。

## 关键返回结构速查（实测归纳）

- **VStockTickerDto**（market、ticker/{uid}）：streamerUid、userName、board（0–3）、status、nav、pbRatio、currentPrice、previousClosePrice、emaFundamentalPoints、liquidityDepth、longShares、shortShares、liveCount30d、danmakuCount30d、income30d、isLiving、sparkline、topLongHolders、topShortHolders。
- **VStockAccountDto**（portfolio）：accountId、seasonId、cashBalance、totalLongMarketValue、totalShortLiability、totalShortMarginLocked、netAssetValue、longPositions[]（shares、avgCostPrice、currentPrice、marketValue、unrealizedPnl、weightedAcquiredAt、shortHoldShares）、shortPositions[]。
- **VStockLedgerDto**（portfolio/ledgers）：type 为枚举 1–15（实测 4=交易、5=收播分红），note 为中文明细，另有 deltaCash、balanceAfter、realizedPnl、liveId、createdAt。
- **VStockDividendSettlementDto**（ticker/{uid}/settlements）：liveId、liveFundamentalPoints、dividendPerShare、totalLongDividendPaid、totalShortDividendCharged、navAfter、boardAfter、promotedToMainBoard、settledAt。
- **检索积分**（account/search-points）：`{earned, spent, available, eventsPerPoint}`，每 5000 条有效互动积 1 点（eventsPerPoint=5000）。

## 使用礼仪

- 请把请求频率控制在正常 App 使用的量级，这是社区运营的服务。
- Access Token 等同密码，永远不要把它提交进任何仓库。
