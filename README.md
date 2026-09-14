\# 物联网协议转换网关模拟 (Node-RED) - 智能门禁 \& 智能冰箱



\## 📖 项目简介

本项目基于 Node-RED 可视化编程工具，模拟了真实物联网场景下的\*\*边缘网关\*\*功能。实现了将底层的\*\*Modbus 协议\*\*（智能门禁）和 \*\*HTTP 协议\*\*（智能冰箱）的数据，转换为云平台标准的 \*\*MQTT 协议\*\*，并完成了设备与云平台之间的\*\*双向通信闭环\*\*（数据上报、云端读取、云端修改）。



项目旨在验证物联网平台（QrLinks）的接入规范，并为后续在实际 Linux 网关盒子中编写部署代码提供逻辑参考。



\## 📂 目录结构

\* `/Node-RED \_mqtt,modbus\_files`：Modbus转MQTT（智能门禁）项目的相关资源文件。

\* `/Node-RED http转mqtt协议网关链路图\_files`：HTTP转MQTT（智能冰箱）项目的相关资源文件。

\* `modbus转mqtt.docx`：Modbus协议转换详细技术文档。

\* `《智能冰箱 HTTP 协议转 MQTT 物联网网关模...`：HTTP协议转换详细技术文档。

\* `Node-RED \_mqtt,modbus.html`：可直接导入 Node-RED 的门禁网关完整流。

\* `Node-RED http转mqtt协议网关链路图.html`：可直接导入 Node-RED 的冰箱网关完整流。



\## 🏗️ 核心架构与功能



\### 1. 智能门禁 (Modbus -> MQTT)

\* \*\*架构\*\*：网关代理子设备接入（`Smart\_door\_wg\_control\_dev` 代理 `Smart\_door\_control\_dev`）。

\* \*\*上行\*\*：Node-RED 定时通过 Modbus TCP 读取寄存器（指纹 `zhiwen`、红外 `hongwai`），打包为 JSON 并加上 `child` 路由发往云端。

\* \*\*下行\*\*：监听云端写指令，转换为 `Modbus Flex Write` 指令修改本地寄存器；支持多设备并发（Unit-ID 1 和 2）。

\* \*\*闭环\*\*：写入完成后自动触发回读，将最新状态上报给云端，完成状态同步。



\### 2. 智能冰箱 (HTTP -> MQTT)

\* \*\*架构\*\*：Postman 模拟 HTTP 设备（`bxsb`），Node-RED 作为网关代理。

\* \*\*上行链路 (HTTP -> MQTT)\*\*：`\[HTTP接收]` 接收 Postman 的 `POST` 上报，秒回 `200 OK`，随后将数据包装为物模型格式（`temp` 浮点数、`door` 整数）并发布至云端。

\* \*\*云端读取链路 (MQTT -> HTTP -> MQTT)\*\*：监听云端 `read` 指令，通过 `\[HTTP请求]` (GET) 向模拟设备索要数据，打包带 `messageId` 的 `reply` 回执给云端。

\* \*\*云端修改链路 (MQTT -> HTTP -> MQTT)\*\*：监听云端 `write` 指令，通过 `\[HTTP请求]` (POST) 向模拟设备下发控制，设备修改状态后\*\*复用上行链路触发上报\*\*，最后打包回执给云端。



\## ⚠️ 避坑指南（核心经验总结）

在开发过程中，我们总结了以下物联网网关开发中极其常见的错误与解决方案：



1\. \*\*底层节点覆盖 `msg.payload` 导致 `messageId` 丢失\*\*

&#x20;  \* \*\*问题\*\*：`HTTP请求`、`Modbus` 节点执行后会强制覆盖 `msg.payload` 为硬件响应，导致业务上下文（如 `messageId`）丢失。

&#x20;  \* \*\*解决\*\*：\*\*永远不要\*\*把需要跨节点传递的上下文放在 `msg.payload` 中。应使用自定义属性 `msg.messageId = msg.payload.messageId`。



2\. \*\*`http request` 报错 `ENOTFOUND bxsb`\*\*

&#x20;  \* \*\*问题\*\*：URL配置不完整（写成了 `http://bxsb/control`）。

&#x20;  \* \*\*解决\*\*：必须在代码或节点中写全 IP 和端口，如 `http://127.0.0.1:1880/bxsb/control`。



3\. \*\*`http request` 节点超时（TimeoutError）\*\*

&#x20;  \* \*\*问题\*\*：模拟设备的 HTTP 接口没有及时返回响应，网关一直等待直至超时。

&#x20;  \* \*\*解决\*\*：在模拟设备的 `\[HTTP接收]` 节点后，\*\*立即\*\*连接一个 `http response` 节点返回 `200 OK`，实现“秒回”。



4\. \*\*云端下发修改后，云端界面数字不更新\*\*

&#x20;  \* \*\*问题\*\*：网关只回复了指令回执（Reply），没有触发属性上报（Report）。云端不会因为收到回执而更新设备影子。

&#x20;  \* \*\*解决\*\*：在设备执行修改后，\*\*复用原有的上行链路\*\*，向云端发送一次新的属性上报（Report）。



5\. \*\*`http request` 节点报错 `Invalid topic specified`\*\*

&#x20;  \* \*\*问题\*\*：发往 `mqtt out` 节点的 `msg.topic` 为空。

&#x20;  \* \*\*解决\*\*：在协议转换节点中，务必动态拼接出完整的云端 Topic，如 `/bxcp\_wg/bxsb\_wg/child/bxsb/properties/report`。



6\. \*\*Node-RED 警告 `msg properties can no longer override...`\*\*

&#x20;  \* \*\*问题\*\*：在节点面板中硬编码了 URL/Method，同时又在代码里传入了 `msg.url`/`msg.method`。

&#x20;  \* \*\*解决\*\*：\*\*节点面板留空\*\*，完全依赖上游 Function 节点动态赋值。



7\. \*\*数据类型不匹配\*\*

&#x20;  \* \*\*问题\*\*：云端定义 `temp` 为 `float`，设备上报了整数 `8` 导致云端拒收。

&#x20;  \* \*\*解决\*\*：在协议转换节点中使用 `parseFloat()` 强制转换数据类型。



\## 🚀 如何使用

1\. 下载本仓库中的 `.html` 文件。

2\. 打开本地 Node-RED 编辑器 (`http://127.0.0.1:1880`)。

3\. 点击右上角菜单 -> `导入 (Import)` -> 粘贴 HTML 中的代码或选择文件导入。

4\. 双击 `mqtt out` 节点，配置为你自己的 MQTT 服务器地址和云平台三元组。

5\. 点击 `部署 (Deploy)` 即可开始模拟。



\## 👤 作者

\* 田思远

\* 日期：2026年9月14日

