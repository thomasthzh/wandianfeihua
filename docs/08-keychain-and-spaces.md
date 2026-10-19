# 一次解锁、自动进入原项目：钥匙串与多桌面

记录日期：2026-10-19。**钥匙串与多桌面新版已部署，真实凭据已导入，本人已完成新版解锁；三个项目已在临时诊断条件下显示真实页面，第二个修复候选尚未部署。** 本文只公开重新编写的设计与经验，不包含真实凭据、配置、身份记录、生产日志或发布包。

## 为什么还要做钥匙串

网关完成密码和 Authenticator 验证后，原项目仍可能显示登录页。原站页面能在窗口打开，只证明入口与嵌入链路可用，不能代表 OS 已经能管理它。

目标因此进一步明确：主人解锁一次 OS，桌面按项目原生协议自动取得管理员能力。长期凭据保留在服务端，浏览器只取得绑定这次 OS 会话的短期使用许可。退出时既撤销新请求，也关闭已建立的项目通道。

“最高权限”指本人已有的项目管理员能力。原应用继续做真实权限判断，不能把前端显示成管理员当作授权，也不能通过一个任意 URL 或任意 shell 接口替代受管连接器。

## 四种应用，分别核对实际协议

| 应用 | 已部署的适配路线 | 需要保留的区别 |
| --- | --- | --- |
| Zashboard/Mihomo | 同源原生 UI 使用空 secret；服务端注入真实控制器鉴权 | 浏览器后端列表不保存真实 secret；HTTP、WS、配置响应分别处理 |
| NetAlertX | 支持的 Bearer 引导 PHP 会话，原站 Cookie 留在服务端 | 保留原生 CSRF；设置页与 API 也可能返回密码哈希和密钥 |
| MCSManager | 原生管理员 API key＋明确的 SPA 初始化适配 | 页面获得的是 broker UI handle，不能混称原生登录 token 或 OIDC；终端、文件仍用原生短期 passport |
| Harness | 按现场安装版本保护固定 Host RPC/WS | 已安装 RC 和最新主分支的浏览器认证协议不同，不能照抄最新版配置 |

这里的依据是官方源码和实际安装版本的代码。参考：[Mihomo 鉴权路由](https://github.com/MetaCubeX/mihomo/blob/Meta/hub/route/server.go)、[NetAlertX 固定版本认证](https://github.com/netalertx/NetAlertX/blob/v26.8.5/front/php/templates/security.php)、[MCSManager 权限中间件](https://github.com/MCSManager/MCSManager/blob/v10.18.0/panel/src/app/middleware/permission.ts)、[Harness 官方协议说明](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/connection/README.md)。最后一项描述当前主分支，不能作为旧 RC 已具备同样能力的证据。

## 授权怎样进入窗口

1. OS 的启动接口验证主人会话、精确来源、CSRF 和项目白名单。
2. 一次性许可限定 App，30 秒有效，单次使用。
3. 许可通过表单 POST 到指定 iframe；不放入 URL、共享域 Cookie 或持久存储。
4. 项目 broker 兑换为该来源专属的安全 Cookie，重定向到不带秘密的原页面。
5. 后续每个 HTTP/WS 请求仍依赖活跃的 OS 会话；服务端只访问固定登记上游。
6. 锁定、退出或过期后拒绝新请求，并中止已有连接。

密文库使用独立 master key 与 AEAD 加密，并将 owner、App、凭据类型和清单纳入完整性校验。系统服务通过受控凭据通道取得 master。密码＋TOTP 决定能否使用钥匙串，不是用六位动态码充当加密密钥，也不是对服务器 root 保密的零知识方案。

## 这轮发现的实际问题

请求头藏住密钥还不够：原生设置响应、HTML 内嵌 JSON、导出、JSON 对象键、编码字符串与 WebSocket 帧都可能把秘密带回浏览器。独立审查以合成凭据复现了几个泄漏点，随后补入结构化过滤和回归测试。原项目的短期终端 passport 则不能被当成长期密码一律清空。

另一个问题是“设置保存”。有的原生页面一次提交整份配置。直接用屏蔽文字替代秘密再透传，会把真实值覆盖成占位符。候选先取得服务端最新配置，严格匹配 schema 和字段集合，只把已识别的秘密占位符恢复为最新原值，再提交普通设置；密钥轮换、完整导入导出等另走后续受管动作。

传输也需要实际运行验证。候选已用临时 HTTP/WS 上游验证超过普通 API 限额的双向流式传输、实际字节上限和过期中断。当前 Bun HTTP listener 可能将已知长度的流转成 chunked；中断在客户端可能表现为普通 EOF。因此不能把连接关闭等同于浏览器已明确提示文件不完整，真实项目下载仍需单独验收。

## Spaces 是工作状态，不只是按钮

新版支持最多8个持久桌面，可创建、重命名、移动窗口和切换；关闭桌面时把窗口迁到相邻桌面。Dock 能返回已有窗口所在桌面，切换和最小化保持 iframe 与未保存内容，刷新恢复窗口归属。旧布局保留，以便回退。

隔离浏览器已验证未保存文件跨桌面移动、关闭桌面后迁回、Dock 寻回与刷新恢复。相同功能已随新版部署，本人也已完成新版解锁；公网已验证新建空桌面、切回时保留同一实例管理 iframe 和原页面，再关闭测试桌面。公网多桌面的完整交互仍需复验。布局组、统一搜索、通知事件流和更多受管系统能力继续保留在路线图中，不宣称 macOS 全效果已经完成。

## 合并发布如何减少重复命令

把调查、源码核对、协议测试、输入冻结、网关解析和回退演练先自动完成，再将必要的本人提权收敛到一次入口。这个入口先验证候选，随后在服务器内部导入精确凭据字段，必要时按原应用支持的方式准备管理员 API，最后切换独立版本和应用路由。

原账号密码和 TOTP 不重置；新管理凭据有私有备份、权限约束和幂等确认。若原生修改结果或密文提交结果不明，就保留状态并报告不确定，不能盲目回退成另一套不匹配的密钥。回退步骤独立尝试，不能因为恢复某个文件失败而跳过其余服务恢复。

## 浏览器验收补记：授权成功后还会遇到什么

### 约两秒半的时钟差，足以让有效许可被前端拒绝

现场的服务器时钟比浏览器所在设备快约 2.5 秒。启动接口已经返回成功，票据格式与固定应用地址也正确，前端却要求服务端的过期时间不能超过“浏览器现在加 30 秒”。服务端刚签发的 30 秒许可，在客户端看来就可能超过这个上限，导致窗口留在接入提示页。

修复候选让浏览器只检查时间戳是正的安全整数，以及票据和目标地址的格式；真正的有效期由服务端时间判断。服务端仍在签发时限定 30 秒、绑定主人会话，并在兑换时检查过期、单次使用与撤销。前端已加入正负约 2.6 秒及正负 24 小时的时钟偏移回归；服务端的过期与重放规则保留。候选测试通过，生产修复尚未部署。

### 隐私策略会影响跨来源表单的 Origin

一次性许可通过跨来源表单 POST 进入项目窗口。现场 OS 文档的 `no-referrer` 策略使这次导航请求出现 `Origin: null`，从而被精确来源检查拒绝。临时对照将 OS 文档改为 `strict-origin` 后，表单能够携带预期来源；它只发送来源信息，不向项目暴露 OS 页面的路径或查询参数，HTTPS 降级时也不发送 referrer。

这里需要同时核对文档策略、表单导航和 iframe 行为，不能只修改 iframe 的 referrer 属性。来源边界继续限定固定 HTTPS 来源，`null` 不进入允许名单。临时对照已撤销，持久修复纳入下一候选。[Fetch 的 Origin 生成规则](https://fetch.spec.whatwg.org/#append-a-request-origin-header)与 [W3C 的 strict-origin 定义](https://www.w3.org/TR/referrer-policy/#referrer-policy-strict-origin)解释了这两个请求头之间的关联。

### Zashboard 必须按现场版本初始化

现场公开静态资源对应 Zashboard 3.18.0。这个版本在没有已保存后端时只尝试默认本机控制器；新版主分支的同源自动探测不能直接作为现场能力。它支持从 URL 取得后端参数，并在 SetupPage 初始化时自动提交。下面使用保留示例域名，省略 `secret`，让原生密码保持为空，真实控制器鉴权仍由服务器 broker 注入：

```text
https://zashboard.os.example.invalid/ui/?hostname=zashboard.os.example.invalid&port=443&https=1#/setup
```

该版本优先读取文档 query，只有 query 为空时才读取 hash 参数。把参数放在 `#` 前并进行完整导航，可避免只更改已挂载路由的 fragment、没有重新执行初始化的问题。可见表单不会自动同步 URL 参数，因此表单仍显示默认值不能单独证明探测没有执行；应检查实际探测请求与最终应用页面。[固定版本参数解析](https://github.com/Zephyruso/zashboard/blob/v3.18.0/src/helper/utils.ts)和 [SetupPage 初始化](https://github.com/Zephyruso/zashboard/blob/v3.18.0/src/views/SetupPage.vue)是这条配置路线的依据。URL 初始化本身仍不能证明原站已成功进入。

### 原生 Authorization 与 MFA 需要分别处理

进一步观察发现，Zashboard 已经发出控制器探测请求，但带原生 Authorization 的请求在其 10 秒期限到达后被取消；同一地址的普通页面导航能立即得到成功响应。固定版本在密码为空时仍生成空 Bearer 头，这一差异需要继续沿网关、MFA 子请求和 broker 分别定位。[原生探测实现](https://github.com/Zephyruso/zashboard/blob/v3.18.0/src/api/clash.ts)

Authelia 的头部认证与 Cookie 会话是不同策略；格式错误的 Authorization 可能在 Cookie 检查前使认证失败。不能把它概括成“所有 Bearer 都优先于 Cookie”：实际行为还取决于头部解析、允许的 scheme 和策略配置。[对应版本的认证策略](https://github.com/authelia/authelia/blob/v4.39.28/internal/handlers/handler_authz_authn.go)及[头部解析](https://github.com/authelia/authelia/blob/v4.39.28/internal/model/authorization.go)支持这一调查方向。

下一候选拟隔离受管应用的原生鉴权头，让 MFA 子请求继续验证 SSO Cookie，项目 broker 再注入它保管的上游凭据。带头超时是现场观察；完整故障链、修复后的自动进入及长连接仍待部署后验收，当前不能写成已经修复。

## 本人发布后的实际状态

首轮在暂存文件权限检查停止，后续又暴露 NetAlertX 磁盘配置与运行时凭据不一致的兼容性假设；两次失败均保留记录，详见[排错手册](05-troubleshooting.md)。修复先经普通 Linux 用户的临时目录回归与独立审查，再由本人执行独立修复版发布。

本人执行已完成版本安装与激活，随后只读核对确认活动版本正确，桌面、身份与网关三项服务运行。Mihomo、NetAlertX 和 MCSManager 的三个真实 Bearer 凭据已导入密文库：NetAlertX 保留经原生 API 验证的同一运行时值并受控持久化；MCSManager 按支持的管理员 API 流程准备并验证管理凭据。原账号密码、Authenticator 绑定、角色与业务 DNS 沿用原状。

独立 IPv4 检查通过六个站点的可信 TLS 与匿名认证边界，四个应用 broker 均拒绝无许可访问。这证明发布与入口边界已进入运行状态，仍不能代替登录后的原站业务验收。内嵌浏览器曾显示连接关闭，而独立客户端检查正常；这一差异继续用新连接与实际页面核对，不据此直接判定服务故障。

本人已用原密码和 Authenticator 完成新版公网解锁。在临时时钟与文档策略对照条件下，MCSManager 显示真实实例列表及管理员操作入口，NetAlertX 显示真实设备页面，Harness 显示真实工作空间。这些页面证据没有验证所有管理写入，也不能替代持久修复部署后的完整验收。

临时时钟与文档 meta 对照均已撤销；第二个修复候选尚未部署。Zashboard 最终正常进入仍待确认。真实终端、上传下载、长连接、锁定与退出后的完整撤销，以及 SMTP 投递和负向 MFA，继续等待对应证据。独立外网 IPv6 请求仍未通过；地址一致、本机监听与本机 TLS 成功都不能替代外部路径验收。
