# 2026-09-22

## MWS jp-bot xhttp 节点修复


**根因**：`app.py`(单文件 Python VLESS-XHTTP 服务)的 GET 下行响应没有 `Transfer-Encoding: chunked` 框架(close-delimited),**Cloudflare 边缘对这类响应全量缓冲**——xhttp 下行数据直到会话结束连接关闭才吐出。TLS 长会话永远等不到首字节→超时；短 HTTP 请求像"勉强通"实为缓冲后一次性吐出。入站链路/订阅/协议实现/出口网关本身都是好的。

**修复**(已上传生效+重启验证)：
- 新增 `ChunkedWriter` 类；`handle_get` 200 头加 `Transfer-Encoding: chunked` + `Cache-Control: no-store` + `X-Accel-Buffering: no`
- `relay_down` 用 chunked 分帧写，结尾写 `0\r\n\r\n`
- 200 头延迟到 init_event(POST seq=0)之后再发；init 超时返回 504 并 cleanup
- 验证：xray(h2 订阅参数)全绿，google/youtube/CF TLS 0.6s 级，2MB 下载 ~600KB/s,出口 219.94.255.123 日本 SAKURA

**附带**：
- `.pc.conf` 是平台按 egress 自动生成(proxychains 到 10.201.0.1 网关),勿手改。
- 平台 API 技巧：文件读 `/api/bots/{id}/files/{path}`,写 multipart POST `/api/bots/{id}/upload`,日志 `/api/bots/{id}/logs/download`,egress POST `/api/bots/{id}/egress {"egress":""}`,重启 POST `/api/bots/{id}/restart`。
