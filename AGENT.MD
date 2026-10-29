# 本地业务定制说明

## 通知功能链已按业务确认关闭

本项目当前业务已确认不使用以下 Matrix 客户端功能：

- `GET /_matrix/client/v3/notifications` 通知列表；
- `/sync` 响应中的 `unread_notifications` 未读通知数和高亮数；
- `POST /rooms/{roomID}/read_markers` 读标记接口；
- 普通 `m.read`、`m.read.private` receipt 接口及其 User API 联动；
- 基于 push rules 的本地通知记录、通知未读数和通知表清理链路。

因此代码中对 Dendrite 默认通知链进行了本地业务裁剪。目的不是修复上游实现，也不是临时绕过数据库问题，而是停止当前产品不使用的功能，避免继续向 `userapi_notifications` 写入大量数据并造成数据库 IO 压力。

## 变更范围

升级或合并上游代码时，以下改动属于有意保留的业务定制，不应误判为无效代码而直接恢复：

- `clientapi/routing/routing.go`：不注册 `/notifications`、`read_markers` 和 receipt 路由；
- `clientapi/routing/account_data.go`：移除 read marker 处理逻辑；
- `clientapi/routing/receipt.go`：已删除，普通 receipt 接口关闭；
- `userapi/consumers/roomserver.go`：不执行 `notifyLocal` 通知链，不写入通知表、不统计未读通知、不发送通知数据；
- `userapi/consumers/clientapi.go`：已删除，不再消费 receipt 并更新通知表；
- `userapi/userapi.go`：不启动通知回执消费者和通知表定时清理任务；
- `userapi/internal/user_api.go`：读标记不再删除通知记录或发送通知统计；
- `syncapi/streams/stream_notificationdata.go`：不再向 `/sync` 房间响应填充 `unread_notifications`；
- `syncapi/types/types.go`：`unread_notifications` 不再序列化到客户端响应；
- `syncapi/syncapi.go`：不启动通知数据消费者。

## 升级冲突判断原则

1. 如果上游新增或修改了通知、push rules、receipt、read marker 或 `/sync` 未读通知逻辑，先确认是否仍符合本项目“通知功能关闭”的业务要求，再决定是否合并。
2. 不要仅因为上游文件删除内容与本地差异较大，就自动恢复这些路由、消费者或数据库调用。
3. 不要仅恢复某一个入口。例如恢复 `/notifications` 路由但不恢复写入链路，会导致接口长期返回空数据；恢复通知写入但不恢复清理，会重新引入表膨胀和 IO 风险。
4. 如果未来需要重新启用通知功能，应按完整功能链评估：路由、通知写入、未读统计、`/sync` 输出、receipt/read marker 联动、清理任务、数据库容量和监控必须一起设计和验证。
5. 当前关闭方式是代码裁剪，不是现成配置开关。若未来需要可配置启停，应该新增统一配置项并将上述入口改为一致的开关判断，避免只恢复部分功能。

## 数据库说明

当前改动停止新通知记录及相关读写，但没有在代码中直接删除 `userapi_notifications` 表结构。历史数据清理、归档或删表必须单独制定数据库变更方案，并先确认所有部署实例已运行不再访问该表的版本。

## 提交 `f386fe8aa2588c057f842175ec83c13bb4e0e879` 说明

该提交（2026-10-29，提交标题：`Optimize PostgreSQL indexes and profile update queries`）是一次 PostgreSQL 查询优化提交，不是通知功能重新启用，也不是业务功能变更。升级或处理冲突时需要保留其优化意图：

- `mediaapi/storage/postgres/media_repository_table.go`：新增 `(base64hash, media_origin)` 索引，优化按文件哈希查询媒体，避免在 `base64hash` 上全表扫描；
- `syncapi/storage/postgres/output_room_events_topology_table.go`：新增 `(room_id, topological_position, stream_position)` 索引，匹配按房间和拓扑位置查询同步事件的过滤条件；
- `userapi/storage/postgres/notifications_table.go`：新增 `(localpart, server_name, read)` 索引，优化未读通知统计时的过滤。
- `userapi/storage/postgres/profile_table.go`：为头像和显示名更新 SQL 增加 `old.localpart = new.localpart AND old.server_name = new.server_name` 连接条件，避免 `UPDATE ... FROM` 产生无连接条件的笛卡尔积，并确保变更判断只针对同一用户。

### 与通知功能关闭的关系

该提交中的 `userapi_notifications` 索引只是当时通知功能仍启用时的查询优化。当前项目已经关闭通知写入、通知未读数、`/notifications`、read marker 和 receipt 联动，因此升级冲突时：

1. 不要因为本地关闭通知功能，就误删该提交的其他三个 PostgreSQL 索引；它们分别服务媒体查询、同步事件查询和资料更新之外的独立链路。
2. `userapi_notifications_localpart_server_read_idx` 可以继续保留在已有数据库中，不会恢复通知功能；是否删除它应作为单独的数据库容量治理操作评估，不能在代码合并时直接执行 `DROP INDEX`。
3. 如果上游重新修改 `notifications_table.go`，优先保留本地“停止通知业务读写”的定制，同时核对索引定义是否仍会被数据库初始化 SQL 保留或重复创建。
4. `profile_table.go` 的同用户连接条件属于正确性修复，不应在解决冲突时删除；若上游重写 SQL，应验证更新前后返回值仍表示同一用户资料字段是否发生变化。

## 验证要求

修改或升级后至少检查：

- `GET /notifications`、`/rooms/{roomID}/read_markers` 和 receipt 路由不再开放；
- `/sync` 响应不再包含 `unread_notifications`；
- `userapi_notifications` 不再产生新写入；
- User API 不再启动通知回执消费者和通知清理定时任务；
- Go 格式化、编译和相关测试通过。

