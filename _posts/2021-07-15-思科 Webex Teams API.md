---
layout:         post
title:          思科 Webex Teams API
subtitle:		    Webex Teams 的 API 调用举例，仅用于备忘，其实是一些很基础的应用方式。
date:           2021-07-15
author:         Forest
cover:          '/assets/img/post-CiscoWebexTeams-bg.png'
tags:           API, Webex teams, postman
---

# API 工具介绍
最早开始接触 API，是备考思科 Devnet core DC 方向，去学习和 coding 相关的一些资料。

API(Application Programming Interface) 可以理解为应用程序之间的接口， 通过开放一些访问、写入权限，允许两个程序之间更有效率的通信。不同厂商之间的深度集成，就可以通过 API 来实现，比如思科 ACI 与虚拟化 VMware VC 的集成。

举一个栗子，**实际情况并不一定这样**：大众点评需要依据用户当前的位置来推送服务，用户位置信息可以从高德地图获取，大众点评自己可以不去做 map 这一块的功能。

![](/assets/img/post-CiscoWebexTeams-API.jpeg)

# 思科 Webex Teams
思科出品的 collaboration 协作产品，市场上的竞品包括 zoom，腾讯会议，阿里钉钉等。

因为在思科工作期间使用 teams 开发过几款脚本，做实时 message 提醒，对于 [Teams](https://teams.webex.com/signin) 的 [bot](https://developer.webex.com/docs/bots)，比较熟悉。后来在肖工的提醒下，去了解了一些 [Chatbot](https://developer.webex.com/blog/from-zero-to-webex-teams-chatbot-in-15-minutes) 的资料，做出了简单的 demo，所以便想着在离开思科以后，也能继续用下去。在这里把公开的资料和步骤做简单记录。

[Webex Teams 的 API](https://developer.webex.com/docs/api/getting-started) 功能很多，像是 meeting、message、file 等，我经常用到的是 rooms & message，可以直接在网页做一些 API 测试，也可以借助 postman 来测试。

# Webex Teams bot 测试截图

1. 首先申请 Webex Teams 账号，我用了自己 @163 的邮箱

2. 之后申请 Webex Teams bot，保存自己的 bot token

3. 设置 postman：
```
method: get
url   : https://webexapis.com/v1/<function>
Headers:
  Authorization: Bearer <Your bot token here>
  Content-Type:  application/json
```

以下截图，用于查看Webex teams 相关 bot 所在的 rooms：

![](/assets/img/post-CiscoWebexTeams-postman-1.png)

以下截图，用于发送消息，注意 method: **post**：

![](/assets/img/post-CiscoWebexTeams-postman-2.png)

收到消息的截图如下：

![](/assets/img/post-CiscoWebexTeams-postman-3.png)

4. 假如使用 **Python 脚本** 去做测试，下面脚本整合了三个功能，新建 room、添加 members、发送 message

```
import requests, sys, json

payload = {}
head = {
  'Authorization': 'Bearer <Your bot token here>',
  'Content-Type': 'application/json'}

rooms_url =      "https://webexapis.com/v1/rooms"
membership_url = 'https://webexapis.com/v1/memberships'
msg_url =        'https://webexapis.com/v1/messages'

# Create a new Teams, and review new created Teams details info
def create_new_team():
    res = requests.post(url = rooms_url,
                    headers = head, json = {'title': sys.argv[1]})
    global spaceId
    spaceId = res.json()['id']
    res.raise_for_status()
    if res.status_code == 200: print('\nThe status_code for create new teams: ' + str(res.status_code) + ', succeed\n')
    print('The new created Teams info: \n' + '**** speace-id:   ' + spaceId + '\n' + '**** space-title: ' + res.json()['title'] + '\n')
    return spaceId

# Add members to the new created Teams, and review members details in the new created Teams
def add_members():
    members = ['fushuang@cisco.com', 'huangfusen2013@163.com']
    for member in members:
        res = requests.post(url = membership_url,
                        headers = head,
                        json = {'roomId':spaceId, 'personEmail':member})
        res.raise_for_status()
        if res.status_code == 200:
            print('Added ' + str(res.json()['personDisplayName']).split(' ')[0] + ' to the group.')

def send_initial_msg():
    markdown = '* Thanks for joining this space, \n\n' + '* This space created by **Python & Webex_Teams API**.'
    res = requests.request("POST", msg_url + '?roomId=' + spaceId,
                                headers = head,
                                json = {'roomId':spaceId, 'markdown': markdown})
    res.raise_for_status()
    if res.status_code == 200:
        print('\nSent initial message to the Teams.\n')

if __name__ == '__main__':
    create_new_team()
    add_members()
    send_initial_msg()
```

执行 Python 脚本，以及 console log：

![](/assets/img/post-CiscoWebexTeams-python1.png)

Webex Teams 效果：

![](/assets/img/post-CiscoWebexTeams-python2.png)


# Webex Teams Chatbot 部署过程和测试截图

[Webex chatBot](https://developer.webex.com/blog/from-zero-to-webex-teams-chatbot-in-15-minutes) 快速部署，参考 link 提供了保姆级部署方式，一步一步做出来效果，还是很让人开心的 😄

先放一张借来的效果图：

![](https://images.contentstack.io/v3/assets/bltd74e2c7e18c68b20/bltdbd0d29a323c1901/5dee96c8162f1938620d47e1/bot-starter-example.gif)

**使用到的工具如下：**

  - Webex Teams 账户
  - Webex Teams bot/token
  - [webex-node-bot-framework](https://github.com/WebexSamples/webex-node-bot-framework) API
  - Webex Teams Chatbot 的后台服务器是需要能够通过 HTTP 交互的，[nGrok](https://dashboard.ngrok.com/get-started/setup) 可以把自己的电脑 ip_address:port 部署为 HTTP 服务器

      MBP Terminal:

        cd /Users/fushuang/Downloads/
        ./ngrok http 666 region=eu

      web page    : `localhost:4040`
  - JavaScript, web/server-side scripting 服务器端脚本，可提供的功能，会用到 Node.js & npm

      npm

        cd /Users/fushuang/Downloads/webex-bot-starter
        npm start

来一个我最近手绘的流程图吧(顺带安利一下 iPad，Apple pencil，goodnote5 😂)：

![](/assets/img/post-CiscoWebexTeams-Chatbot1.png)

**自定义webex-bot-starter的 configuration.json 以及 index.js 脚本**：

  - 通过修改 `configuration.json` 文件，提供 Chatbot 需要的 bot token，监听的 webhookUrl & port
```
{
  "webhookUrl": "http://936bd0449942.eu.ngrok.io",  // 注意`gGork` 若重启，这里的 link 就需要 update
  "token": "<Your bot token here>",
  "port": 666
}
```

  - 通过修改 `index.js` 脚本，可以定制化 Chatbot 提供的功能。下面是一些简单的说明:
    测试用的脚本放到了 workday-Focus 里面
```
framework.hears // 脚本监听的关键字
bot.say         // Chatbot 发送消息
bot.reply       // Chatbot 发送消息、文件
var xxx         // 创建对象，一般都是 function 内有效，并非 global
let xxx = xxx   // 赋值、调用
```

**调用 API 的测试脚本**
```
framework.hears('hi', function (bot) {
  console.log("someone asked for words, and this is powered by free API");
  responded = true;
  axios.get('https://api.fczbl.vip/hitokoto/') // will request the api every time someone say hi
    .then(abc => {
      responded = true;
      console.log(abc.data);  /* welcome */
      var hi = abc.data
  bot.say(hi)});
  responded = true;
/* console.log(abc);   <<< if you want to see the details repspons*/
});
```
补充一个 link，关于[免费 API，不需要 key 认证的](https://apipheny.io/free-api/)

使用另一个 bot **Friday** 做的截图：

![](/assets/img/post-CiscoWebexTeams-Chatbot2.png)

![](/assets/img/post-CiscoWebexTeams-Chatbot3.png)

针对 API 的调用：
![](/assets/img/post-CiscoWebexTeams-Chatbot4.png)
