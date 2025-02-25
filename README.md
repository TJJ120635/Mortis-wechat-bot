# 实现带联网搜索能力的墨缇丝微信群聊助手 DeepSeek+AstrBot+Dify

开始之前先看看效果
![](attachments\Pasted image 20250224111805.png)

## 一、背景

想要将 DeepSeek-R1 以群机器人的方式接入到微信群中
网上有比较多微信/QQ机器人+大模型的框架
例如 LangBot/chatgpt-on-wechat/AstrBot/wechat-bot/chatgpt-mirai-qq-bot 等等

我希望具备的功能：
- 接入个人微信
- 支持记录微信群聊记录上下文
- 支持 DeepSeek-R1
- 支持联网搜索功能
- 未来考虑实现长时记忆能力（让大模型定期整理记忆）

综合考虑下面的技术路线：
- 大模型底座+联网搜索：火山方舟 https://console.volcengine.com/ark
- 核心框架：AstrBot https://github.com/Soulter/AstrBot
- 消息平台服务：Gewechat https://github.com/Devo919/Gewechat
- 大模型应用编排：Dify https://github.com/langgenius/dify
- OpenAI 格式转换：new-api https://github.com/Calcium-Ion/new-api?tab=readme-ov-file
如果不需要那么复杂的功能
我建议用 **火山方舟+AstrBot+Gewechat** 的组合即可
一键部署后简单配置就能开始玩，非常方便

技术路线考虑如下：
- 火山方舟底座：满血+无截断 DeepSeek-R1，服务稳定性优秀，免费 50万 tokens，邀请好友注册送 30 代金券。输入 0.0040 元/千tokens，输出 0.0160 元/千tokens
- 火山方舟联网搜索：支持 DeepSeek-R1+联网搜索结果分析，搜索质量优于腾讯大模型知识引擎，质量接近 Bing，注册比 Azure 方便
- AstrBot 框架：支持个人微信接入（基于 Gewechat），支持 Dify，有插件生态
- Gewechat 消息平台：AstrBot 用的是这个所以就用了
- Dify 编排框架：聊天助手/智能体/工作流/对话流 等多种形式，可视化编辑容易上手，支持工具调用、HTTP 请求、Python 代码等丰富功能
- new-api 服务：基于 one-api，支持 Dify

为什么要用 Dify？
- 最开始我想要实现 DeepSeek-R1+联网搜索
- AstrBot 有联网插件，基于 Agent 函数调用的方式，适合 GPT-4o、Gemini 等模型，DeepSeek-R1 不支持
- 想要实现联网要自己用 Dify 搭建工作流
为什么要用 new-api？
- AstrBot 支持接入 Dify 的聊天助手/工作流/对话流，但是传入的参数里没有群聊消息记录
- AstrBot 接入 Dify 模式无法传入群聊消息记录，只能使用 Dify 自带的上下文记忆，无法传入群友 ID
- AstrBot 接入 OpenAI 格式接口能够传入群聊消息记录，带有时间戳和群友 ID 等信息
- 因此考虑用 new-api 将 Dify 转成 OpenAI 格式接口
## 二、部署环境

需要一台支持 Docker 且 24 小时不关机的机器，Windows/Linux 不限
如果家里没有电脑可以考虑网上租一台便宜的云服务器
性能要求很低，我估计 2C4G 的 Linux 机器就行

我自己用的是家里的极摩客 K1 迷你电脑
平时用来挂 MAA，装了一个 Docker Desktop 部署服务

![](attachments\Pasted image 20250224113413.png)
## 三、注册火山方舟服务

#### 3.1 账号注册
进入火山引擎官网并注册登录 https://www.volcengine.com/
![](attachments\Pasted image 20250224114358.png)
顶部菜单-大模型-火山方舟 跳转页面
进入火山方舟控制台 https://console.volcengine.com/ark

#### 3.2 开通服务

【开通管理】
需要用到的大模型：
- DeepSeek-R1：核心推理模型
- Doubao-1.5-lite-32k：用于工作流意图判断、Dify 代码生成等
![](attachments\Pasted image 20250224114907.png)

【API Key 管理】
创建一个 API Key
![](attachments\Pasted image 20250224115211.png)

【在线推理-自定义推理接入点】
每个模型创建一个推理接入点
![](attachments\Pasted image 20250224115400.png)
接入点名称自定义
建议把模型名称写进接入点名称中，便于区分
![](attachments\Pasted image 20250224115544.png)
在 【自定义接入点-API调用】 部分可以看到 OpenAI 格式的调用方式
- 接入点 base_url：https://ark.cn-beijing.volces.com/api/v3
- API Key：ec67d64a-xxxx-xxxx-xxxx-xxx
- Model ID：ep-20250217120317-xxx
这里的 Model ID 相当于 gpt-4o/DeepSeek-R1 等模型型号，用的是推理接入点 ID

## 四、注册微信小号

参考 https://zhuanlan.zhihu.com/p/650276620
【设置-切换账号-添加账号-注册新账号】
通过当前微信的手机号辅助注册

如果手机系统支持多开微信最好用第二个微信登录小号
![](attachments\Pasted image 20250224141151.png)
## 五、部署 AstrBot+Gewechat

项目地址
AstrBot https://github.com/Soulter/AstrBot
Gewechat https://github.com/Devo919/Gewechat

#### 5.1 Docker 启动

统一采用 docker run 的方式启动

使用 Docker 部署 AstrBot https://astrbot.app/deploy/astrbot/docker.html
```bash
PS C:\Docker\astrbot> docker run -itd -p 6180-6200:6180-6200 -p 11451:11451 -v $PWD/data:/AstrBot/data --name astrbot soulter/astrbot:latest
```
启动之后最好检查一下容器的时区，以免出现时间戳不一致的问题
```bash
PS C:\WINDOWS\system32> docker exec -it astrbot /bin/bash
root@9d65634803a7:/AstrBot## date
Tue Feb 25 11:07:03 CST 2025
```
如果时区不对就进行修改
```bash
root@9d65634803a7:/AstrBot## ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
root@9d65634803a7:/AstrBot## echo "Asia/Shanghai" > /etc/timezone
```

启动 Gewechat Docker 服务 https://github.com/Devo919/Gewechat?tab=readme-ov-file#%E5%90%AF%E5%8A%A8%E6%9C%8D%E5%8A%A1
```bash
PS C:\Docker\astrbot> docker pull registry.cn-hangzhou.aliyuncs.com/gewe/gewe:latest
PS C:\Docker\astrbot> docker tag registry.cn-hangzhou.aliyuncs.com/gewe/gewe gewe
PS C:\Docker\astrbot> mkdir gewechat
PS C:\Docker\astrbot> docker run -itd -v gewechat:/root/temp -p 2531:2531 -p 2532:2532 --privileged=true --name=gewe gewe /usr/sbin/init
```
#### 5.2 配置 AstrBot+Gewechat

登录到 http://localhost:6185 ，默认账户密码 astrbot/astrbot

添加消息平台

![](attachments\Pasted image 20250225101326.png)

添加完之后重启 AstrBot，扫码登录微信

![](attachments\Pasted image 20250225102731.png)

如果遇到“创建设备失败，获取二维码失败”的报错，关掉代理之后再重启 AstrBot
https://github.com/Soulter/AstrBot/issues/398

这个时候还没有配置大模型，但是已经连接了微信
可以发送命令进行测试

第一次发送的时候会控制台会提示
```bash
 [03:33:21| INFO] [event_bus.py:21]: [gewechat] 微信名/wxid_xxx: /help
[2025-02-11 03:33:21 +0000] [1] [INFO] 172.17.0.1:47086 POST /astrbot-gewechat/callback 1.1 200 20 1345
 [03:33:21| INFO] [stage.py:41]: 会话 ID gewechat:FriendMessage:wxid_xxx 不在会话白名单中，已终止事件传播。请在配置文件中添加该会话 ID 到白名单。
```
在【配置】中开启白名单，然后保存重启 AstrBot

![](attachments\Pasted image 20250225103306.png)

再次测试命令

![](attachments\Pasted image 20250225103006.png)
#### 5.3 配置 AstrBot+LLM

添加大模型服务商
第一次建议添加一个具备 Agent 功能的模型进行测试，例如 gpt-4o-mini/doubao-1.5-lite-32k 之类的

前面我们已经开通了 DeepSeek-R1 和 doubao-1.5-lite-32k
- 接入点 base_url：https://ark.cn-beijing.volces.com/api/v3
- API Key：ec67d64a-xxxx-xxxx-xxxx-xxx
- Model ID：ep-20250217120317-xxx

这里注意【文本生成模型】填写的是接入点 ID 而不是 doubao-1.5-lite-32k

![](attachments\Pasted image 20250225103528.png)

可以在【配置】中设置
例如人格情景、联网插件、群聊上下文记录等

![](attachments\Pasted image 20250225104058.png)

然后保存并重启 AstrBot，在微信进行测试
用 /persona 切换人格，/provider 切换模型，/new 和 /switch 和 /ls 管理对话等

在私聊中直接对话即可
在群聊中把墨缇丝拉进群然后 @墨缇丝 进行对话，也可以在配置中加上自定义的唤醒前缀词
在微信中手动 @ 和复制消息的 @ 是不同的

![](attachments\Pasted image 20250225103734.png)

### 六、部署 Dify+new-api

至此我们能够跑通 AstrBot 为核心的，接入大模型供应商的微信机器人
那么如果我们想要更丰富的功能，就需要使用 Dify 框架
这里我选择了使用 Dify 对话流进行编排

项目地址
Dify https://github.com/Soulter/AstrBot
new-api https://github.com/Calcium-Ion/new-api
#### 6.1 Docker 启动

Dify 部署需要使用 Docker Compose https://docs.dify.ai/zh-hans/getting-started/install-self-hosted/docker-compose
```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d
```
会启动一共 9 个容器，注意检查全部进入 running 状态即启动成功
如果有问题就检查 docker logs
一般是 redis 容器和 db 容器启动失败一直 restarting，通过 docker-compose.yml 中修改容器 priviledged 等方式解决

![](attachments\Pasted image 20250225110243.png)

new-api 通过 Docker 部署 https://github.com/Calcium-Ion/new-api?tab=readme-ov-file#%E7%9B%B4%E6%8E%A5%E4%BD%BF%E7%94%A8-docker-%E9%95%9C%E5%83%8F
```bash
docker run --name new-api -d --restart always -p 3000:3000 -e TZ=Asia/Shanghai -v /home/ubuntu/data/new-api:/data calciumion/new-api:latest
```
#### 6.2 配置 Dify

启动之后首次登录 Dify http://localhost/install 设置管理员

![](attachments\Pasted image 20250225111050.png)
设置完之后登录主页【右上角-设置-模型供应商】，添加火山模型
模型名称自定义，【鉴权方式】选择 API Key

![](attachments\Pasted image 20250225111250.png)

#### 6.3 创建初始对话流

在【工作室-创建空白应用】，推荐选择 Chatflow
目前 new-api 支持聊天助手、文本生成、Chatflow、工作流，好像不支持 Agent

![](attachments\Pasted image 20250225111441.png)

创建完了之后是一个简单的对话，右上角发布即可

![](attachments\Pasted image 20250225111645.png)

可以先在【预览】中进行本地测试，确认模型跑通

![](attachments\Pasted image 20250225111745.png)

跑通之后在【访问API】部分，创建 API Key：app-xxx
在外部调用 Dify 不同的工作流，是通过 API Key 进行区分的

![](attachments\Pasted image 20250225111831.png)
#### 6.4 配置 new-api+AstrBot

完成 Dify 初始对话流的创建之后
登录 new-api http://localhost:3000 ，默认账号密码 root/123456

在【渠道-添加渠道】添加 Dify 工作流
- 名称随意
- BaseURL 不是 http://localhost 而是 http://host.docker.internal
- 密钥是前面在 Dify 对话流中创建的 API Key
- 模型建议清空所有模型列表，然后填入一个自定义的名称

![](attachments\Pasted image 20250225112618.png)

在【令牌-添加令牌】新建一个访问权限
- 可以设置永不过期+无限额度
复制得到 new-api 的 API Key：sk-xxx

注意，一个 Dify 工作流对应一个渠道，而令牌只创建一个即可

![](attachments\Pasted image 20250225112907.png)

回到 AstrBot，添加 new-api 的访问方式
- API Key 复制刚才 new-api 的令牌
- 地址对应 new-api 的端口
- 模型名称对应 new-api 渠道里填入的模型名称
完成配置后用 /provider 在聊天里切换模型

![](attachments\Pasted image 20250225113154.png)

## 七、设计 Dify 对话流

我们已经走通了 Dify+new-api+AstrBot+Gewechat 的整个框架
接下来就进入正题开始构建对话流了

如果不想看构建对话流的过程可以直接抄作业
保存 dsl 文件直接导入到 Dify 即可
DSL 文件太长了，我后面在评论区贴出来
![](attachments\Pasted image 20250225114418.png)
#### 7.1 输入处理

AstrBot 支持群聊上下文，需要 OpenAI 格式的接口才会启用，默认的 Dify 工作流不启用
这是为什么我们需要 new-api 中转

让我们在 Dify 后台看看群聊的格式
```json
{
  "sys.query": "SYSTEM: \n\nCurrent datetime: 2025-02-21 14:38\nYou are now in a chatroom. The chat history is as follows: \n[愿逐月华/06:37:50]:  /new\n---\n[愿逐月华/06:37:51]:  /reset\n---\n[愿逐月华/06:38:01]:  你好啊墨缇丝\nUSER: \n\n[User ID: wxid_xxx, Nickname: 愿逐月华]\n你好啊墨缇丝\n",
  "sys.files": [],
  "sys.conversation_id": "6a8684c5-f373-46fb-8aad-d664ab99152d",
  "sys.user_id": "api-user",
  "sys.dialogue_count": 0,
  "sys.app_id": "b29d7337-c169-4a0b-8d72-8b52c1906609",
  "sys.workflow_id": "e91d8eae-937a-4067-93b0-bb72091c230b",
  "sys.workflow_run_id": "6852c677-19b6-4367-9b1a-5501d1ec1a63"
}
```
![](attachments\Pasted image 20250225162558.png)
![](attachments\Pasted image 20250225162834.png)

整理下来大概是这样的内容
```python
'''
SYSTEM:
You are now in a chatroom. The chat history is as follows:
[ʕ •ᴥ•ʔ/15:24:50]: 你好

USER:
[User ID: wxid_xxx, Nickname: 愿逐月华]
你好啊墨缇丝

ASSISTANT:
💭 好的，我现在需要处理用户发来的消息“你好啊墨缇丝”。首先，用户使用了中文打招呼，这可能意味着他们希望用中文进行交流。接下来，我要确认用户是否在之前的对话中有特定的需求或上下文需要处理。

查看历史记录，用户之前发送了/new和/reset命令，这可能表示他们想要开始新的对话或重置之前的会话。当前用户的消息是“你好啊墨缇丝”，看起来像是一般的问候，但结合之前的命令，用户可能希望重新开始对话，所以需要确认是否需要重置之前的上下文。

另外，用户ID是wxid_xxx，昵称是“愿逐月华”。在回应时，使用昵称可以增加亲切感。需要确保回应友好且符合用户的语言风格，即使用中文进行回复。

考虑到用户可能已经重置了对话，回应应该简洁，同时保持开放，邀请用户进一步说明需求。例如，询问有什么可以帮助他们的地方。同时，注意不要提及之前的对话历史，因为用户可能已经通过/reset清除了之前的上下文。

最后，要检查是否有任何系统或时间相关的信息需要处理，当前时间是2025年2月21日14:38，但用户的消息中没有涉及时间敏感的内容，因此可以忽略时间因素。总结回应需要友好、简洁，并鼓励用户提出具体请求。

USER:
[User ID: wxid_xxx, Nickname: ʕ •ᴥ•ʔ]
你好
'''
```
其中，SYSTEM 里面包括了背景信息和一部分的对话历史
后面的对话历史以 USER: ASSISTANT: USER: 这样的格式进行

那么我们首先需要将最新一条 USER 作为 prompt_user
前面的所有历史记录作为 prompt_system

这样做是为了区分最新一条信息的意图，在条件分支中判断搜索
也可以用 doubao-lite 进行问题分类器意图判断

同时挂一个获取当前时间的工具，便于搜索查询
![](attachments\Pasted image 20250225163805.png)
代码执行部分如下
```python
import re

def main(query: str) -> dict:
    ## 按最后一次出现 "USER: \n\n" 切分
    parts = query.rsplit('USER: \n\n', 1)
    if len(parts) == 2:
        query_system = parts[0].strip()
        query_user = parts[1].strip()
        ## 如果需要替换系统提示中的某些文字，可自行调整，比如：
        query_system = query_system.replace('You are now in a chatroom. The chat history is as follows', '群聊的历史纪录如下')
        return {
            'query_system': query_system,
            'query_user': query_user
        }
    return {
        'query_system': '',
        'query_user': query
    }
```
#### 7.2 直接回复

如果不需要搜索，那么就走直接回复的路线
我们设定 system prompt 即可

我尝试过在 prompt 里塞墨缇丝的设定
但是效果不好，回答的时候有股怪味，不适合当群聊机器人
DeepSeek是知道Ave Mujica的，但是不知道墨缇丝
一旦开始过度 Cosplay 之后的回答方向都会跑偏

因此最后只告诉了她名字
```text
你是群聊的一个成员，大家可能会叫你墨缇斯、大莫老师、小莫、Mortis等。不需要过多夸张的角色扮演，群友也知道你是DeepSeek，保持正常风格交流即可，请以群友的身份和大家聊天。
基于微信群聊的聊天，因此回答中不要带有markdown中的格式渲染符号，例如用**表示加粗等。
当前时间：{{当前时间}}
{{query_system}}
```
![](attachments\Pasted image 20250225164356.png)
#### 7.3 开通火山引擎联网搜索

在尝试火山引擎之前，我尝试过 Jina 和 Bing

Jina 要 \$20/1B tokens，相当于 \$0.02/1M tokens
搜索质量很高，而且经过加工成Markdown之后再给模型
但是不是很舍得一次性给20刀

Bing 要 VISA 卡，每天有免费额度
我专门去开了个卡，结果 API 开通不了
据网上说是 Bing 官方开通接口有问题

其他搜索引擎不用考虑了，例如 DuckDuckGo、SearXNG 等等，搜索国内内容质量不可用

暂时使用曲线救国的方式
【火山方舟-我的应用-创建应用-零代码模式】

![](attachments\Pasted image 20250225165323.png)

创建一个空白的带联网插件的应用即可，然后发布
不需要提示词，由 Dify 提供提示词

![](attachments\Pasted image 20250225165346.png)
#### 7.4 Dify 接入火山联网应用

因为火山应用需要用应用 ID `bot-xxx` 代替模型 ID `ep-xxx`
而且需要设置非流式推理等参数
因此我选择使用 HTTP 方式调用，而不是 Dify 内置 LLM 方式

具体使用方式在【火山方舟-应用-编辑-API调用指南】中

使用 POST 方式
Authorization：Beaver xxxx（填入API Key）
Body：选择 JSON 方式，注意非流式推理并设置最大输出长度
```json
{
  "model": "bot-20250223031134-xxx",
  "stream": false,
  "max_tokens": 1024,
  "messages": [
    {
      "role": "system",
      "content": "你是群聊的一个成员，大家可能会叫你墨缇丝、大莫老师、小莫、Mortis等。不需要过多夸张的角色扮演，群友也知道你是DeepSeek，保持正常风格交流即可，请以群友的身份和大家聊天。\\n基于微信群聊的聊天，因此回答中不要带有markdown中的格式渲染符号，例如用**表示加粗等。适当地用换行或者两个换行进行分段，更加美观整齐\\n当前时间：{{text}}\\n{{query_system}}"
    },
    {
      "role": "user",
      "content": "{{#1740122268305.query_user#}}"
    }
  ]
}
```
![](attachments\Pasted image 20250225165834.png)

考虑到联网搜索的思维链会很长，微信有最大消息长度限制
因此联网搜索部分不输出思维链，只输出结果
直接回复时选择 answer
```python
import json


def main(response: str) -> dict:
    body_dict = json.loads(response)
    choices = body_dict["choices"]
    message = choices[0]["message"]
    content = message["content"]
    reasoning_content = message["reasoning_content"]
    return {
        "reasoning": reasoning_content,
        "answer": content
    }
```
到这里就完成整个工作流了
## 附录

看看 AstrBot 对话上下文轮次设置为 100 时的 Token 消耗
多的时候一天消耗大概 200k，不到 0.8 元

![](attachments\Pasted image 20250225170440.png)

联网搜索引用数量10+上下文轮次100 的时候，14 次调用消耗了 74k
![](attachments\Pasted image 20250225170730.png)
建议是邀请群友注册火山方舟，一个人送30块，好友能送15块
然后微信群聊上下文设置低一点
以后可能会考虑加入长时记忆功能，让大模型定期总结聊天记录
或者是接入 Bing 之后自己处理联网搜索的过程