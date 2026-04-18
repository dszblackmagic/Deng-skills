# Weaver open-doc 链接解析

## 1. 适用场景

当用户给的是 Weaver 的 open-doc 链接，而不是已经整理好的接口文本时，先把正文取出来，再做代码生成。

常见链接形态：

- `http://host:port/sp/opendoc/freepass/{version}/{lang}/{docId}`

## 2. 关键问题

这类页面很多时候返回的是前端壳 HTML，不是正文。

也就是说：

- 直接 `curl` 页面地址，通常只能拿到 HTML shell
- 真正的文档正文要靠前端再去拉 JSON

## 3. 优先解析规则

如果 URL 满足：

- 路径里包含 `/opendoc/freepass/{version}/{lang}/{docId}`

优先尝试直接取正文 JSON：

- `/build/opendoc/docs/{version}/files/f{docId}.{lang}.json`

示例：

- 页面：
  - `/sp/opendoc/freepass/10.0.2509.01/zh_cn/840693647360663552`
- 正文 JSON：
  - `/build/opendoc/docs/10.0.2509.01/files/f840693647360663552.zh_cn.json`

## 4. 解析步骤

### 4.1 先确认页面是否只是壳

如果页面里只有：

- `<div id="root"></div>`
- 前端静态资源脚本

说明正文不在 HTML 里。

### 4.2 按 URL 推导 JSON 地址

从页面 URL 提取：

- `version`
- `lang`
- `docId`

然后拼：

- `/build/opendoc/docs/{version}/files/f{docId}.{lang}.json`

### 4.3 从 JSON 中提取接口信息

重点提取：

- 请求 URL
- 请求方式
- 参数表
- 请求示例
- 返回示例
- 返回字段说明
- SDK 示例

## 5. 如果直接推导失败

备用方式：

1. 抓取页面引用的前端 JS
2. 定位 open-doc 前端如何组装 `docs/{version}/files/...json`
3. 或用无头浏览器观察实际请求

优先顺序：

- 直接推导 JSON
- 看前端 bundle 中的请求路径
- 最后才用浏览器跑页面

## 6. 解析完成后的动作

拿到文档正文后，不要停在“读懂文档”。

继续做：

1. 判断 `access_token` 放 query 还是 body
2. 判断 `userid` 是否必须
3. 判断是否有 count/list 双接口
4. 判断请求体是否要拆强类型 DTO
5. 按目标项目的现有分层生成代码
