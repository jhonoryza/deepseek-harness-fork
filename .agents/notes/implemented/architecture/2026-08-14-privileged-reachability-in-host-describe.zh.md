# Agent Note: 可信 LAN 客户端通过 host.describe 的特权判定打开配置面板

Status: implemented

[English](2026-08-14-privileged-reachability-in-host-describe.md) | 中文

## 问题

Web GUI 的配置面板（设置 — 插件配置页卡片、本地文档操作、欢迎确认、以及产出文件"在文件夹中显示"操作）此前在客户端以 `connection.isLoopback` 作为门槛。那只是页面主机的启发式判断，与服务器的信任围栏无关：在可信家庭 LAN 部署中，当设置 `allowPrivilegedFromTrustedHosts` 时，服务器已经允许来自声明的 `trustedHosts` 的特权方法（见配置面板边界[谁能触达它](2026-07-30-config-plane-boundaries.md)），但浏览器以 `http://192.168.x.x` 访问时仍把自己归类为远程，导致插件配置面板渲染为空（`persistence: 'memory'` 的作用域从不读取 Host 设置文档）。服务器围栏——按请求 Host 的 `isTrustedApiRequest`——是唯一正确的真值来源，但客户端插件收不到任何 cordis 配置，因此客户端无法得知自己的判定结果。

## 决策

连接层在就绪握手的 `host.describe` 响应上标注该请求自身的特权判定。在 node-half `/api` 回退中，`privilegedReachable = isTrustedApiRequest(request, allowPrivilegedFromTrustedHosts ? trustedHosts : [])`——与门控 `PRIVILEGED_METHODS` 相同的谓词——并把成功的 `host.describe` 信封的 `result.value.privilegedReachable` 设为该值（非成功或非信封响应原样通过）。该字段在 schema 与契约类型上均为可选，因此从不标注它的进程内 handler 路径继续可用，客户端回退到 `isLoopback`。

客户端消费方通过既有的 `connection.hostDescription` 通道响应式读取该判定（握手在流打开之后运行，而 UI 插件已经 apply 完毕——`bind()`/`apply()` 先以握手前的回退构造，再由 `hostDescription.subscribe` 在收到首个标注描述时升级）：

- `SettingsScopeController` 新增 `upgradeToHost()`；`SettingsScopeBinder.bind` 以 `hostDescription.getSnapshot()?.privilegedReachable ?? isLoopback` 构造并订阅，把内存作用域升级为 Host 持久化，从而发起真正的 `settings.describe` 读取。
- `WelcomeNoticeStore` 新增 `upgradeToHost()`；ui-settings-models 的 apply 接入同样的响应式构造与升级。
- ui-settings-general 移除 loopback 门槛，总是注册打开文档操作：特权 403 让 `SettingsDocumentStore.load()` 降级为 `unavailable`，操作渲染为 null，因此未获特权的远程浏览器只是看不到该操作。
- ui-deliverables 的 `ProducedFiles` 通过 `useHostDescription` 计算 `canOpenPath = (privilegedReachable ?? isLoopback) && hostCanOpenPath`。

默认安全性不变：未获特权的远程浏览器保持内存持久化与 unavailable/不渲染状态，绝不出现永久 spinner——`upgradeToHost` 是唯一的迁移方向，且只会在服务器标注 `privilegedReachable: true` 时触发。

## 曾考虑的替代方案

- **新增一个暴露判定的 ConnectionHandle API**（方法或专用 source）——否决：`hostDescription` 已携带握手值及其订阅通道；可选 schema 字段沿既有类型传递，不新增表面积。
- **在 apiproxy handler 中计算判定**——否决：`toFetchHandler` 与 `ApiProxy` 实现与请求无关（看不到 Host 头），只有连接插件的回退闭包同时拥有请求与配置。
- **把 schema 字段设为必需**——否决：精确对象测试与进程内 handler 路径（fixture/`InProcessApiClient`）会为无线上收益而破坏；缺省仍表示"未标注"。

## 结果

- 可信 LAN 浏览器（服务器 `allowPrivilegedFromTrustedHosts: true` 且来自可信权威）现在渲染完整的插件配置、本地文档、欢迎与产出文件表面，因为服务器自身的围栏判定到达了客户端。
- 客户端保持单一响应式真值来源：握手前为页面权威，握手后为服务器判定；重连时按代重新发布描述。
- 未受信任的远程行为与之前完全一致（memory/unavailable），包括 describe 上的 403 路径——现在降级为隐藏操作而非不注册。
- Host 线上协议为 `host.describe` 增加一个可选布尔字段；不升级协议版本（客户端与宿主机一同发布）。
