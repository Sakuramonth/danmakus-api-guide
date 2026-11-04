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

Swagger UI：`https://api.ukamnads.icu/swagger/index.html`，包含 `v2` 与 `v3` 两套定义。两套定义的路径与结构目前**完全相同**（158 条路径、165 个操作）；v3 界面里标为新增的分组，其接口路径实际仍在 `/api/v2/` 下。

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

状态说明：✅ 200 = 2026 年 11 月实测通过（GET 共 114 个，102 个返回 200；写接口于 2026-11-04 逐一实测，口径见各行与文末说明）；`HTTP xxx（见文末说明）` = 该次实测的实际状态。参数带 `*` 为必填。

### Account

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/account/add-record` | uId:integer | ✅ 200 |
| POST `/api/v2/account/avatar-intent` | body:AvatarUploadIntentRequest | ✅ 200（2026-11 实测：返回 key 与预签名 uploadUrl；仅取上传意图，未完成上传）。 |
| GET `/api/v2/account/channel-lives` | uId:integer; page:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/contributed-lives` | pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/account/core-config` | - | ✅ 200 |
| POST `/api/v2/account/core-config` | body:CoreControlConfig | ✅ 200（2026-11 实测：先 GET 现值，再原值回写成功）。 |
| GET `/api/v2/account/del-record` | uId:integer | ✅ 200 |
| GET `/api/v2/account/info` | id:integer | ✅ 200 |
| GET `/api/v2/account/recording` | - | ✅ 200 |
| GET `/api/v2/account/search-points` | - | ✅ 200 |
| GET `/api/v2/account/search-points/history` | page:integer; pageSize:integer; filter:string | ✅ 200 |
| GET `/api/v2/account/statistics` | start:string; end:string | ✅ 200 |
| POST `/api/v2/account/statistics/report` | body:ReportStatisticRequest | ✅ 200（2026-11 实测：零值上报成功）。 |
| POST `/api/v2/account/update-avatar` | body:RequestUpdateAvatar | 实测 ✅ 第二轮（2026-11-04）：意图→PUT 上传→更新全流程 200，头像已真实变更（API 无取消头像接口、不可回滚；更新走 body 里的 avatarUrl） |
| POST `/api/v2/account/update-name` | body:RequestUpdateUserName | ✅ 200（2026-11 实测：改名、核对、再改回原名均成功）。 |
| POST `/api/v2/account/update-public-profile` | body:RequestUpdateContributorPublicProfile | ✅ 200（2026-11 实测：现值原样回写成功）。 |
| POST `/api/v2/account/update-recording-setting` | - | ✅ 200（2026-11 实测：当前设置为空数组，原样回写成功；该接口整体覆盖录制设置，慎用）。 |

### AgencyWiki

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/agencies` | keyword:string; type:AgencyGroupType; pageNum:integer; pageSize:integer; sort:string | ✅ 200 |
| POST `/api/v2/agencies/avatar-complete` | body:CompleteAgencyAvatarRequest | 实测 ✅ 第二轮（2026-11-04）：配合 avatar-intent 与 PUT 上传调用返回 200；临时机构提案已 withdraw 撤回清理 |
| POST `/api/v2/agencies/avatar-intent` | body:AvatarUploadIntentRequest | 实测 ✅ 第二轮（2026-11-04，临时机构）：返回 200 意图与上传地址，配合 presigned PUT 上传 200；临时机构提案已 withdraw 撤回清理 |
| POST `/api/v2/agencies/propose` | body:SubmitAgencyUpsertRequest | 实测 ✅ 第二轮（2026-11-04）：返回 200 并生成真实提案（type=0 机构创建），已 withdraw 撤回清理（status 置 7） |
| POST `/api/v2/agencies/sync-vdb` | - | HTTP 403（2026-11 实测：「仅百科版主可触发 VDB 同步」，权限拦截）。 |
| GET `/api/v2/agencies/{uuid}` | uuid*:string | ✅ 200 |
| POST `/api/v2/agencies/{uuid}/propose` | uuid*:string; body:SubmitAgencyUpsertRequest | 实测 ✅ 第二轮（2026-11-04）：对临时机构调用返回 200 并生成真实提案（type=1 机构资料），已 withdraw 撤回清理 |
| GET `/api/v2/agency-options` | - | ✅ 200 |
| GET `/api/v2/agency/{uuid}` | uuid*:string | ✅ 200 |
| GET `/api/v2/agency/{uuid}/timeline` | uuid*:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/channel/{userId}/lineage` | userId*:integer; platform:string; includeSuspected:boolean | ✅ 200 |
| GET `/api/v2/channel/{userId}/timeline` | userId*:integer; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/affiliation/submit` | body:SubmitChannelAffiliationRequest | 实测 ✅ 第二轮（2026-11-04）：返回 200 并生成真实提案（type=2 隶属档案），已 withdraw 撤回清理 |
| GET `/api/v2/wiki/channel/{userId}/career` | userId*:integer | ✅ 200 |
| GET `/api/v2/wiki/channel/{userId}/lives` | userId*:integer; keyword:string; limit:integer | ✅ 200 |
| GET `/api/v2/wiki/content` | scopeKey:string; targetId:string | HTTP 400（见文末说明） |
| POST `/api/v2/wiki/content/revise` | body:SubmitWikiContentRevisionRequest | 实测 ✅ 第二轮（2026-11-04）：用自有临时机构频道调用返回 200 并生成真实提案（type=8 频道简介），已 withdraw 撤回清理 |
| GET `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer | HTTP 403（见文末说明） |
| POST `/api/v2/wiki/governance/accounts/{accountId}` | accountId*:integer; body:UpdateWikiAccountStateRequest | HTTP 403（2026-11 实测：「仅百科管理员可调整账号角色与封禁」）。 |
| GET `/api/v2/wiki/governance/profile` | - | ✅ 200 |
| GET `/api/v2/wiki/lineage` | platform:string; identity:string; includeSuspected:boolean | ✅ 200 |
| POST `/api/v2/wiki/lineage/submit` | body:SubmitLineageRequest | 真调用被业务拦截（2026-11-04 第二轮）：先返回 400 要求随附可核实来源链接，补链接后返回 400「强共识操作：仅社区居民及以上等级可执行」——等级不足，未能完成 |
| POST `/api/v2/wiki/lineage/{lineageId}/dispute` | lineageId*:string; body:SubmitWikiDisputeRequest | 未能真测（2026-11-04 第二轮）：站内暂无世系记录（抽查多个频道 lineage 均为空数组），无真实 lineageId 可用作争议目标 |
| GET `/api/v2/wiki/proposals` | status:WikiProposalStatus; agencyUuid:string; streamerUid:integer; proposalType:WikiProposalType; scopeKey:string; mine:boolean; votedByMe:boolean; appealed:boolean; aiOutcome:WikiAiReviewOutcome; sort:WikiProposalSort; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/proposals/{proposalId}/appeal` | proposalId*:string; body:AppealWikiProposalRequest | 真调用被前置条件拦截（2026-11-04 第二轮）：返回 400——只能在提案被恶意驳回后 14 天内申诉，无符合前置条件的提案可用 |
| POST `/api/v2/wiki/proposals/{proposalId}/emergency` | proposalId*:string; body:EmergencyWikiProposalRequest | 真调用被权限拦截（2026-11-04 第二轮）：返回 403「仅百科版主可执行紧急处理」 |
| POST `/api/v2/wiki/proposals/{proposalId}/vote` | proposalId*:string; body:VoteWikiProposalRequest | 真调用被权限拦截（2026-11-04 第二轮）：返回 403——复核投票权未解锁（需注册满 30 天且贡献达标），无法完成投票 |
| POST `/api/v2/wiki/proposals/{proposalId}/withdraw` | proposalId*:string; body:WithdrawWikiProposalRequest | 实测 ✅（2026-11-04，第一轮 1 次 + 第二轮 6 次）：返回 200，目标提案 status 置为 7（已撤回）；对已撤回/已结案提案再调用返回 400「只能撤回仍在审议中的提案」 |
| GET `/api/v2/wiki/timeline` | userId:integer; agencyUuid:string; category:WikiCategory; minImportance:integer; pageNum:integer; pageSize:integer | ✅ 200 |
| POST `/api/v2/wiki/timeline/submit` | body:SubmitWikiNodeRequest | 实测 ✅ 第二轮（2026-11-04）：返回 200 并生成真实提案（type=3 里程碑新建），已 withdraw 撤回清理 |
| POST `/api/v2/wiki/timeline/{eventId}/reaction` | eventId*:string; body:ToggleWikiReactionRequest | 实测 ✅ 第二轮（2026-11-04）：对自有里程碑返回 200 点👍，复查确认点亮；再次调用即取消（服务端 toggle），已清理（净效果为零） |
| POST `/api/v2/wiki/timeline/{nodeId}/dispute` | nodeId*:string; body:SubmitWikiDisputeRequest | 真调用被业务拦截（2026-11-04 第二轮）：先返回 400 要求随附反证链接，补链接后返回 400「强共识操作：仅社区居民及以上等级可执行」——等级不足，未能完成 |
| POST `/api/v2/wiki/timeline/{nodeId}/revise` | nodeId*:string; diffSummary:string; body:SubmitWikiNodeRequest | 实测 ✅ 第二轮（2026-11-04）：对自有里程碑调用返回 200 并生成真实提案（type=4 里程碑修订），已 withdraw 撤回清理 |

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
| POST `/api/v2/core-runtime/clients/remark` | body:ClientRemarkRequest | ✅ 200（2026-11 实测：对自建临时客户端真改备注并已改回；虚构 clientId 返回业务码 400「设备不存在」）。请求体：{clientId, remark}。 |
| POST `/api/v2/core-runtime/clients/remove` | body:RemoveClientRequest | ✅ 200（2026-11 实测：移除自建临时客户端成功；虚构 clientId 亦返回成功，属 no-op）。请求体：{clientId, blockClientId}。 |
| GET `/api/v2/core-runtime/extraction-rules` | - | ✅ 200 |
| GET `/api/v2/core-runtime/heartbeat` | clientId:string | HTTP 204（见文末说明） |
| POST `/api/v2/core-runtime/heartbeat` | body:CoreHeartbeatRequest | 实测 ✅ 第二轮（2026-11-04，临时 clientId）：返回 204 无内容（勿用 JSON 解析读响应，会误报失败）；本轮的临时客户端已用 state(state=2) + clients/remove 清理 |
| POST `/api/v2/core-runtime/request-room` | body:RequestRoomRequest | ✅ 200（2026-11 实测：desiredCount=0 调用成功，返回 holdingRooms、newlyAssignedRooms、droppedRooms、effectiveCapacity）。 |
| DELETE `/api/v2/core-runtime/state` | clientId:string | ✅ 200（2026-11 实测：清除自建临时客户端状态成功）。 |
| POST `/api/v2/core-runtime/sync` | body:CoreStateSyncRequest | ✅ 200（2026-11 实测：状态上报成功，会在 clients 列表生成客户端记录）。 |
| POST `/api/v2/core-runtime/upload-extracted` | - | 仍未能真测（2026-11-04 第二轮）：Swagger 未声明请求体，空体返回 400「请求体不能为空」，JSON 数组与 3 种 JSON 对象、multipart、base64 信封共 6 种构造均返回 400「请求体解析失败」——疑为核心客户端专用未公开格式 |
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
| POST `/api/v2/chat/add` | body:ChatModel | 真调用被拦截（2026-11-04 第二轮）：业务码 403「验证失败，可能是发送过于频繁」——token 同为网页验证码令牌，纯 API 无法完成 |
| GET `/api/v2/chat/get` | date:integer; count:integer | ✅ 200 |
| POST `/api/v2/comment/add` | uId:integer; body:AddCommentRequest | 真调用被拦截（2026-11-04 第二轮）：HTTP 200 但业务码 403「人机验证失败」——token 为网页验证码令牌，纯 API 无法完成 |
| GET `/api/v2/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/comments` | targetType:SiteCommentTargetType; targetId:string | HTTP 400（见文末说明） |
| POST `/api/v2/comments` | body:AddSiteCommentRequest | 真调用被拦截（2026-11-04 第二轮）：HTTP 200 但业务码 403「人机验证失败」——body 的 token 是网页验证码令牌，API 内没有签发端点，纯 API 无法完成 |
| GET `/api/v2/comments/counts` | targetType:SiteCommentTargetType; ids:string | HTTP 400（见文末说明） |
| GET `/api/v2/comments/stream` | viewerId:string | ✅ 200 |
| DELETE `/api/v2/comments/{id}` | id*:string | 未能真测（2026-11-04 第二轮）：自有评论被评论接口的人机验证拦截无法创建，无可删的自有评论；不删除他人评论 |
| GET `/api/v2/emoicons` | - | ✅ 200 |
| GET `/api/v2/guilds` | - | ✅ 200 |
| GET `/api/v2/live` | liveId*:string; uId:integer; type:array; pageNum:integer; pageSize:integer; includeExtra:boolean; useEmoji:boolean; includeDanmakus:boolean | ✅ 200 |
| POST `/api/v2/live/comment/add` | body:AddLiveCommentRequest | 真调用被拦截（2026-11-04 第二轮）：在播房间 76693693（主播「上官林萤」，实时观看 0）发「test」返回 HTTP 200 但业务码 403「人机验证失败」——同为人机验证拦截，未产生可见评论 |
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
| POST `/api/v2/user/comment/add` | body:AddUserCommentRequest | 真调用被拦截（2026-11-04 第二轮）：HTTP 200 但业务码 403「人机验证失败」——同 comments 组，纯 API 无法完成 |
| GET `/api/v2/user/comment/get` | uId:integer | ✅ 200 |
| GET `/api/v2/user/name` | uid*:array | ✅ 200 |
| GET `/api/v2/user/watchedChannels` | uid*:integer; simple:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/user/watchedChannelsSimple` | uid*:integer; distinct:boolean; limit:integer | ✅ 200 |
| GET `/api/v2/vup-list` | - | ✅ 200 |

### DanmakuSearch

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/danmaku/search` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200 |
| POST `/api/v2/danmaku/search/async` | q*:string; start*:integer; end*:integer; channelUid:integer; uid:integer; limit:integer; priority:boolean; requestId:string; cursor:string | ✅ 200（2026-11 实测：创建成功，返回 taskId）。 |
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
| POST `/api/v2/identity/equip` | body:IdentityEquipRequest | HTTP 200／业务码 400（2026-11 实测：无此身份，返回「身份不存在」）。 |
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
| POST `/api/v3/user-analysis-jobs` | body:CreateUserAnalyzeJobRequest | ✅ 200（2026-11 实测：创建成功返回 jobId，作业随即完成）。 |
| DELETE `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404（2026-11 实测：对刚创建且 GET 可见的作业调用仍返回框架级 404，疑服务端未实现该 DELETE 路由）。 |
| GET `/api/v3/user-analysis-jobs/{jobId}` | jobId*:string | HTTP 404（见文末说明） |
| GET `/api/v3/user-analysis-jobs/{jobId}/events` | jobId*:string | HTTP 404（见文末说明） |
| GET `/api/v3/user-analysis-jobs/{jobId}/result` | jobId*:string | HTTP 404（见文末说明） |

### VStock

| 方法与路径 | 参数 | 状态 / 说明 |
| --- | --- | --- |
| GET `/api/v2/vstock/agencies` | sort:string | ✅ 200 |
| GET `/api/v2/vstock/agency/{uuid}/index` | uuid*:string | ✅ 200 |
| POST `/api/v2/vstock/burn` | body:VStockBurnRequest | HTTP 200／业务码 400（2026-11 实测：无持仓，以不存在标的调用返回「标的不存在，无法提纯」；不可逆操作，未做真实销毁）。 |
| GET `/api/v2/vstock/crystal-leaderboard` | streamerUid:integer; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/flows` | window:string | ✅ 200 |
| GET `/api/v2/vstock/headlines` | - | ✅ 200 |
| GET `/api/v2/vstock/index/bars` | interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/leaderboard` | type:string; limit:integer | ✅ 200 |
| GET `/api/v2/vstock/live/{liveId}/settlement` | liveId*:string | ✅ 200 |
| GET `/api/v2/vstock/market` | board:VStockBoard; agencyUuid:string; keyword:string; pageNum:integer; pageSize:integer; sortBy:string; area:string; uids:string; activeOnly:boolean | ✅ 200 |
| GET `/api/v2/vstock/portfolio` | - | ✅ 200 |
| POST `/api/v2/vstock/portfolio/claim-bankruptcy-relief` | - | HTTP 200／业务码 400（2026-11 实测：净资产 9990 时返回「未达到破产救济门槛，需低于 200」）。 |
| GET `/api/v2/vstock/portfolio/ledgers` | type:VStockLedgerType; pageNum:integer; pageSize:integer | ✅ 200 |
| GET `/api/v2/vstock/portfolio/position-stats` | - | ✅ 200 |
| POST `/api/v2/vstock/predictions/bet` | body:VStockPredictionBetRequest | ✅ 200（已实测：以最小注 10 D币下注成功）。 |
| GET `/api/v2/vstock/predictions/history` | limit:integer | ✅ 200 |
| GET `/api/v2/vstock/predictions/open` | - | ✅ 200 |
| GET `/api/v2/vstock/pulse` | - | ✅ 200 |
| GET `/api/v2/vstock/tape` | - | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/bars` | uid*:integer; interval:string; hours:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/flow` | uid*:integer; window:string | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/history` | uid*:integer; days:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/pin` | uid*:integer; body:VStockCommentPinRequest | 校验 + 真调用业务拦截（2026-11-04）：以扁平 `{}` 探测时返回 validation 400（commentId 须为真实 Guid、另报 req 必填）；第二轮改用真实 Guid 调用，扁平结构正常绑定，拦截点变为业务规则「只能置顶自己在该标的下发布的评论」（业务码 400）——自有评论又被评论接口的人机验证挡住（见 comments 组），置顶链条止于此 |
| GET `/api/v2/vstock/ticker/{uid}/predictions` | uid*:integer | ✅ 200 |
| GET `/api/v2/vstock/ticker/{uid}/settlements` | uid*:integer; limit:integer | ✅ 200 |
| POST `/api/v2/vstock/ticker/{uid}/sync` | uid*:integer | ✅ 200（2026-11 实测：触发同步并返回最新 ticker 数据）。 |
| POST `/api/v2/vstock/trade` | body:VStockTradeRequest | 已实测（side 1/2）。市价即时成交。 |
| POST `/api/v2/vstock/wiki-tag` | body:SubmitVtbIdentityTagRequest | 注意：空 body 探测即返回 200「标定申请已提交至百科审议池」，服务端对缺省值不做校验（streamerUid=0 的申请会被提交）；调用前务必带全参数。（后续：第一轮误提交的这条 streamerUid=0 提案已由提交者本人撤回清理。） |

## 未通过项说明（实测口径）

- **写接口实测覆盖（2026-11-04）**：此前 48 个未实测写接口已逐一调用：真调用成功 15 个；业务码/权限拦截 5 个（sync-vdb 与 wiki governance 为 403 权限、equip/burn/破产救济为业务码 400）；DELETE 分析作业为路由级 404（见下）；仅做参数校验探测 26 个（公共内容类只探通路，未提交真实内容）；1 个（POST core-runtime/heartbeat）因凭据代理对该路径报错未能取得响应。
- **DELETE /api/v3/user-analysis-jobs/{jobId}**：对刚创建、GET 可见且已完成的作业调用，两次均返回框架级 404（Not Found），疑服务端未实现该 DELETE 路由；作业创建（POST）本身 200 正常。
- **POST /api/v2/vstock/ticker/{uid}/pin**：校验错误显示 commentId 须为 Guid，且另提示 `req` 字段必填——第一轮所见的 `req` 必填报错实为伪造非 Guid 值引发的模型绑定连锁错误；第二轮以真实 Guid 复测，扁平 `{commentId, bid}` 可正常绑定，真实拦截点是业务规则「只能置顶自己在该标的下发布的评论」。
- **POST /api/v2/vstock/wiki-tag**：空 body 即返回 200 并把 streamerUid=0 的缺省申请提交至百科审议池，服务端对该接口不做必填校验，调用方须自行保证参数完整。（后续：该条误提交提案已由提交者本人撤回清理，不再占用审议池。）
- **写接口第二轮实测（2026-11-04）**：第一轮仅做参数校验的端点已升级真调用——wiki/agency 提案类 10 个真提交成功并全部 withdraw 撤回清理（含 agencies 头像 intent→上传→complete 全流程、timeline reaction 开/关 toggle 闭环）；account/update-avatar 走完意图→上传→更新全流程（头像真实变更、无接口可回滚）；core-runtime/heartbeat 用原始响应确认 204（第一轮失败系空响应体被 JSON 解析误报）；评论全家（comments、comment/add、user/comment/add、chat/add、live/comment/add）均被人机验证拦截（业务码 403，token 是网页验证码令牌且 API 内无签发端点）；proposals 的 vote/emergency/appeal 分别被投票权未解锁、仅版主、申诉前置条件拦截；lineage/submit 与 timeline dispute 被「强共识操作需社区居民等级」拦截；ticker pin 以真实 Guid 调用确认拦截点是「只能置顶自己评论」（自建评论又被人机验证挡住）；upload-extracted 的载荷格式未公开，6 种构造均解析失败；DELETE /comments/{id} 因无法自建评论且不删他人评论、lineage dispute 因站内无世系记录，均未能真测。
- **贡献（Contribution）组**：某一轮扫描时整组返回 503（服务端临时状态）；随后复测 `areas`、`me`、`minute-today`、`hub`、`overview`、`rank/day`、`rank/range` 均已 200。`GET /api/v2/contribution/contributor` 传账号 ID 返回 400，其 `id` 参数并非账号 ID，口径待查。
- **400 参数校验**：`GET /api/v2/comments`、`GET /api/v2/identity/nameplates`、`GET /api/v2/wiki/content` 需要具体的业务参数（评论目标、名牌 id 集合、内容块 scopeKey 如 `channel.bio` / `agency.description`），通用探测未命中。
- **404**：`GET /api/v3/user-analysis-jobs/{jobId}*` 是用不存在的任务 ID 探测的预期结果；`GET /api/v3/lives/{liveId}/delta` 在同场其他子资源正常的情况下返回 404，疑该子资源当前节点未提供。
- **204 / 400**：`GET /api/v2/core-runtime/heartbeat` 返回 204（节点探针语义）；`GET /api/v2/core-runtime/ws` 是 WebSocket 升级端点，普通 GET 返回 400 属预期。
- **403**：`GET /api/v2/wiki/governance/accounts/{accountId}` 仅百科版主可看，权限校验正常生效。
- **平台参数**：wiki/lineage 一组接口的 `platform` 仅限 `bilibili / acfun / youtube / twitch / douyin`。
- **带副作用的 GET**：`GET /api/v2/account/add-record` 与 `GET /api/v2/account/del-record` 虽然是 GET，却会真实增删录制名单，任何只读盘点都要把它们当写接口排除。
- **`GET /api/v2/user` 的第三方用法**：规范声明 `uid` 与 `target` 必填，但 2026-11-04 实测 `target` 可省略（省略时等价于 `-1`＝全部频道），且用户 ID 参数用 `uId` 拼写同样有效。开源 Go 库 `NFS688/AnimeAPI` 即以 `/api/v2/user?uId=<uid>&pageNum=0&pageSize=5&target=-1&useEmoji=true` 拉取用户弹幕史；`target` 用于按频道（主播）UID 过滤。

## 关键返回结构速查（实测归纳）

- **VStockTickerDto**（market、ticker/{uid}）：streamerUid、userName、board（0–3）、status、nav、pbRatio、currentPrice、previousClosePrice、emaFundamentalPoints、liquidityDepth、longShares、shortShares、liveCount30d、danmakuCount30d、income30d、isLiving、sparkline、topLongHolders、topShortHolders。
- **VStockAccountDto**（portfolio）：accountId、seasonId、cashBalance、totalLongMarketValue、totalShortLiability、totalShortMarginLocked、netAssetValue、longPositions[]（shares、avgCostPrice、currentPrice、marketValue、unrealizedPnl、weightedAcquiredAt、shortHoldShares）、shortPositions[]。
- **VStockLedgerDto**（portfolio/ledgers）：type 为枚举 1–15（实测 4=交易、5=收播分红），note 为中文明细，另有 deltaCash、balanceAfter、realizedPnl、liveId、createdAt。
- **VStockDividendSettlementDto**（ticker/{uid}/settlements）：liveId、liveFundamentalPoints、dividendPerShare、totalLongDividendPaid、totalShortDividendCharged、navAfter、boardAfter、promotedToMainBoard、settledAt。
- **检索积分**（account/search-points）：`{earned, spent, available, eventsPerPoint}`，每 5000 条有效互动积 1 点（eventsPerPoint=5000）。
- **用户弹幕史**（`GET /api/v2/user`）：分页信封内层 `data.records[]` 为 `{channel{...}, live{...}, danmakus[]}`，一条记录对应一场直播；`danmakus[]` 是被查用户在该场发出的消息（`uId、uName、type、sendDate`，文本/付费条目另有 `message` / `price`）。

## 使用礼仪

- 请把请求频率控制在正常 App 使用的量级，这是社区运营的服务。
- Access Token 等同密码，永远不要把它提交进任何仓库。
