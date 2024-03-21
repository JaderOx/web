---
title: 赛事代码管理
description: 网站开发接口文档
permalink: /code
---

> 要点：代码跟队伍走不跟角色走，编译跟代码走不跟比赛走，比赛跟角色走不跟代码走

### 流程描述（分工）

- 代码提交
  1. 选手在代码管理页面点击上传/拖入文件上传`AI.cpp`或`AI.py`
  2. 前端识别编程语言，并在数据库`contest_team_code`表插入新行，返回得`code_id`
  3. 前端根据`${code_id}.${lang}`重命名文件并上传至`cos`，上传路径见[COS存储桶访问路径约定](https://eesast.github.io/web/cos)
  4. 若语言为解释型语言(`py`)，则前端更改数据库`compile_status`为`No Need`（可与第二步合并）
  5. 若语言为编译型语言(`cpp`)，则前端向后端发请求`/code/compile-start`（见后），使后端开始编译代码
  6. 后端下载`cos`上的代码文件，在服务器上启动编译`docker`，并在数据库中更新`compile_status`为`Compiling`
  7. `docker`完成编译后，请求后端`/code/compile-finish`路由（见后）。若编译成功无报错，后端在数据库中更新`compile_status`为`Completed`；若编译出错，后端在数据库中更新`compile_status`为`Failed`
  8. 后端将可执行文件按`${code_id}`重命名、将`Compile log`按`${code_id}.log`重命名后上传至`cos`，同代码文件夹
  9. 前端通过`subscription`实时更新`compile_status`
- 代码重命名
  - 前端页面上和数据库中的`code_name`是代码文件上传的原名，仅作展示和下载时的命名之用，与后端和`cos`没有关系。用户可以修改这个名字来做版本管理，仅需前端修改数据库`contest_team_code`表即可。
- 编译日志获取
  - 前端直接下载`cos`上对应`${code_id}.log`
- 代码删除
  1. 用户在前端点击删除某份代码文件后，前端先删除`cos`上的`${code_id}.${lang}`、`${code_id}.log`和`${code_id}`可执行文件
  2. 删除成功后，前端再修改数据库删除行，最后提示删除成功
- 角色代码选择
  - 选手可以选择当时当刻进行天梯和比赛使用的代码，选择好后更新数据库`contest_team_player`表

### 后端路由路径

- `/code/compile-start`：下载代码文件并启动编译镜像。
  - 请求方法：`POST`
  - 请求：`body`中有`{contest_name: string, path: string, code_id: uuid, language: string}`，其中`contest_name`是数据库中的`name`、用于确定用于编译的镜像，`path`为代码文件在`cos`上的绝对路径、用于下载，`code_id`用于更改数据库，`language`为编程语言（增加其他语言前固定为`cpp`，不用管）
  - 响应：`200`：`Compiling...`
  - 错误：
    - `422`：`422 Unprocessable Entity: Missing credentials`（请求缺失参数）
    - `404`：`404 Not Found: Code unavailable`（无法成功下载代码）
    - `400`：`400 Bad Request: Too many codes for a single team`（队伍代码数超出限额）
    - `409`：`409 Confilct: Code already in compilation`（代码正在或已编译）
    - `500`：`undefined`（其他内部错误，返回报错信息）
- `/code/compile-finish`：代码完成编译的`hook`，在`docker`结束前调用。更新编译状态并保存可执行文件和`log`。
  - 请求方法：`POST`
  - 请求：`body`中有`{compile_status: string}`，`token`中有`{path: string, code_id: uuid}`（启动`docker`时传入）
  - 响应：`200`：`Compile status updated`
  - 错误：`500`：`undefined`（返回报错信息）

### 与赛事组的约定

1. 编译代码的镜像每次启动只能编译一份代码
2. 镜像启动时代码文件绑定在`/usr/local/mnt`文件夹下，编译产生的可执行文件和`log`请保存到此文件夹（命名与代码文件前缀相同）
3. 镜像启动时会设置环境变量`URL`和`TOKEN`，编译完成后需要请求`URL`（实际上是`/code/compile-finish`），请求时需要在`header`中加上`TOKEN`在`body`中加上`compile_status`