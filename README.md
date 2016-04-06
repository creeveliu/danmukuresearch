#弹幕功能调研

现在很多视频类App都加入了弹幕互动功能（比如Bilibili，优酷等）。

##功能
![](http://ww4.sinaimg.cn/large/006fzSq4jw1f2mslizhanj30vk0hsahy.jpg)

弹幕功能分为直播弹幕和点播弹幕，类似斗鱼直播等游戏直播的弹幕是直播弹幕，而优酷土豆里面的点播视频就是点播弹幕。

直播弹幕实现这一功能：当用户观看某一直播的时候，他可以看到其他正在观看此直播的用户的实时评论。当他选择输入评论的时候，其他正在观看的用户也能观看到他的这条评论弹幕。

点播除了可以观看当前正在观看的实时评论外，还可以观看以往用户在当前用户观看的时间点发送的评论弹幕。


##直播弹幕实现

直播弹幕类似一个聊天室系统，假设用户A用户点击观看某频道的直播，直播内容很精彩，A点开发送弹幕输入框，输入『太好看了』，点击发送（如果用户未登录，会提示登录）。此时A会向服务器发送『太好看了』信息，服务器收到此信息后，会向每一个正在观看此视频的用户（包括未注册的用户）转发此条弹幕消息，各个用户所用的终端（包括iOS，Android和Web）进行解析即可。

要实现这一过程，目前比较流行的技术有客户端轮询、XMPP、WebSocket，目前大多数直播弹幕都用了WebSocket技术，相对于XMPP，WebSocket实现更为简单，下面是一个基本的弹幕/聊天室服务器的Python代码（使用了[EventMachine](https://github.com/igrigorik/em-websocket)库）

```
require "em-websocket"

EventMachine.run do
  @channel = EM::Channel.new
  
  EventMachine::WebSocket.start(host: "0.0.0.0", port: 8080, debug: true) do |ws|
    ws.onopen do
      sid = @channel.subscribe { |msg| ws.send(msg) }
      @channel.push("#{sid} connected")
      ws.onmessage { |msg| @channel.push("<#{sid}> #{msg}") }
      ws.onclose { @channel.unsubscribe(sid) }
    end
  end
end
```

除了Python之外，各种语言/框架也有成熟的实现方案（[socket.io of Node.js](http://socket.io/)，[ws of Node.js](https://github.com/websockets/ws)，[Jetty of Java](http://www.eclipse.org/jetty/)等）

客户端实现也比较简单，iOS可以使用[SocketRocket](https://github.com/square/SocketRocket)实现，Android可以使用[Java-WebSocket](https://github.com/TooTallNate/Java-WebSocket)来实现。

以下是iOS的主要核心代码

```
//连接服务器
NSString *urlString = @"ws://localhost:8080";
  SRWebSocket *newWebSocket = [[SRWebSocket alloc] initWithURL:[NSURL URLWithString:urlString]];

newWebSocket.delegate = self;

[newWebSocket open];


// 连接成功
- (void)webSocketDidOpen:(SRWebSocket *)newWebSocket {
  webSocket = newWebSocket;
  [webSocket send:[NSString stringWithFormat:@"Hello from %@", [UIDevice currentDevice].name]];
}

// 发送信息

- (IBAction)sendMessage:(id)sender {
  [webSocket send:self.messageTextField.text];
  self.messageTextField.text = nil;
}

// 收到信息
- (void)webSocket:(SRWebSocket *)webSocket didReceiveMessage:(id)message {
  self.messagesTextView.text = [NSString stringWithFormat:@"%@\n%@", self.messagesTextView.text, message];
}
```

可以看到，相对于XMPP，WebSocket实现更为简单。此外我们也更方便的定义消息的格式，以及后续定制化一些扩展功能。

采用WebSocket后，服务器端要实现一个转发服务器，类似上面那个十几行的Python代码。后续也可以定制化更多功能，比如定义弹幕发送到指定用户，屏蔽某些比较敏感的词汇等。

此外，如果需要定义弹幕的大小，颜色，字体，滚动速度等更详细的信息，我们还需要制定弹幕信息的格式规范，使得各个平台能按此规范进行解析。


##点播弹幕实现

点播具有直播的实时弹幕发送/显示功能，同时也能显示以往的弹幕。所以除了WebSocket之外，客户端还需要从服务器端获取以往的弹幕信息，这个弹幕信息包含了发送时间，文字内容，发送用户ID这些基本信息，后续也可以包含弹幕的大小，颜色，字体，滚动速度等扩展信息。

服务器端需要实现的功能主要是将用户观看时发送的弹幕信息保存到一个弹幕文件中，并提供一个类似`getDanmukuWithProgramId:(NSString *)programId`的接口，让终端能获取到这个弹幕信息。



###弹幕格式参考

抓包看了下一些弹幕的格式：

优酷：

```
{
    "count": 166,
    "filtered": false,
    "result": [
        {
            "status": 99,
            "mat": 0,
            "iid": 380803623,
            "propertis": "{\"color\":0,\"pos\":6,\"effect\":0,\"size\":1}",
            "uid": 331586722,
            "id": 56758271,
            "ct": 1001,
            "lid": 26378220,
            "ouid": 811143503,
            "ver": 1,
            "level": 0,
            "playat": 41000,
            "ipaddr": 3060013098,
            "content": "还不赶紧去加上",
            "aid": 304446,
            "type": 1,
            "createtime": 1459821853000
        },
        {
            "status": 99,
            "mat": 0,
            "iid": 380803623,
            "propertis": "{\"pos\":4,\"size\":1,\"effect\":0,\"color\":4294967295}",
            "uid": 89736140,
            "id": 56758469,
            "ct": 3002,
            "lid": 0,
            "ouid": 811143503,
            "ver": 1,
            "level": 0,
            "playat": 5292,
            "ipaddr": 174329654,
            "content": "‘生死狙击是’",
            "aid": 0,
            "type": 1,
            "createtime": 1459822107000
        },
        {
            "status": 99,
            "mat": 0,
            "iid": 380803623,
            "propertis": "{\"size\":\"1\",\"alpha\":\"1\",\"pos\":\"6\",\"color\":\"16773376\"}",
            "uid": 842731736,
            "id": 56758483,
            "ct": 3001,
            "lid": 0,
            "ouid": 811143503,
            "ver": 1,
            "level": 0,
            "playat": 8000,
            "ipaddr": 174329658,
            "content": "Hhhhh",
            "aid": 0,
            "type": 1,
            "createtime": 1459822122000
        }
    ]
}
```

腾讯视频：

```
{
    "errCode": 0,
    "data": {
        "targetid": 1003395546,
        "display": 1,
        "total": 4909,
        "reqnum": 30,
        "retnum": 30,
        "maxid": "6123038506787349072",
        "first": "6123038506787349072",
        "last": "6115854205784236586",
        "hasnext": true,
        "commentid": [
            {
                "id": "6123024682733434939",
                "rootid": "0",
                "targetid": 1003395546,
                "parent": "0",
                "timeDifference": "今天 15:54:04",
                "time": 1459842844,
                "content": "月英漂亮是漂亮。那眉毛怎么了。。。",
                "title": "",
                "up": "0",
                "rep": "0",
                "type": "1",
                "hotscale": "0",
                "checktype": "1",
                "checkstatus": "1",
                "isdeleted": "0",
                "tagself": "",
                "taghost": "",
                "source": "9",
                "location": "",
                "address": "",
                "rank": "-1",
                "custom": "",
                "extend": {
                    "at": 0,
                    "ut": 0
                },
                "orireplynum": "0",
                "richtype": 0,
                "userid": "351399430",
                "poke": 0,
                "abstract": "",
                "thirdid": "msgid=2142080775890&userid=217402299&datakey=targetid%3D1003395546%26vid%3D%26cid%3Dxfvsrjfi52lgjlf%26lid%3D%26type%3D1%26out%3D0",
                "replyuser": "",
                "replyuserid": 0,
                "replyhwvip": 0,
                "replyhwlevel": 0,
                "replyhwannual": 0,
                "userinfo": {
                    "userid": "351399430",
                    "uidex": "ec6944e5c872884b21bc1b0fde2ae4f35c",
                    "nick": "对方正在输入…",
                    "head": "http://wx.qlogo.cn/mmopen/ZP9tYorTxPvQU1cEicHjHC2VRibJXMicoGudGHbIbQhwIOeFb5oBbBjvA2tg45anibuTeAQyrmgeALJoR7QvKia3AaedU2WZpIcc9/46",
                    "gender": 2,
                    "viptype": "0",
                    "mediaid": 0,
                    "region": "中国::",
                    "thirdlogin": 4,
                    "hwvip": 0,
                    "hwlevel": 0,
                    "hwannual": 0,
                    "identity": "",
                    "wbuserinfo": [],
                    "remark": "",
                    "fnd": 0
                }
            }
		],
        "targetinfo": {
            "orgcommentnum": 5653,
            "commentnum": "4909",
            "checkstatus": "0",
            "checktype": "1",
            "city": "",
            "voteid": "10106557"
        }
    },
    "info": {
        "time": 1459847473
    }
}
```

此外，bilibili使用了xml，acfun用了json，目前主流还是采用json，就目前我们4.2系统来说，也是json实现较为方便。

具体的格式比如字体，颜色，大小等可以后续再确定，先保证能实现基本的功能。

##其他需要注意的点

1. 服务器需要向每个在播放某一视频的客户端发送消息，当用户量大的时候，是否会对服务器性能有一定要求

##预计开发周期

客户端：1人x1周
服务器端：1人x1周

##相关链接

http://www.websocket.org/aboutwebsocket.html
http://colobu.com/2015/02/27/WebSockets-tutorial-on-Wildfly-8/
https://www.v2ex.com/t/194143
https://cnodejs.org/topic/54fd8d4a1e9291e16a7b3598
http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/HomeWebsocket/WebsocketHome.html
http://www.elabs.se/blog/66-using-websockets-in-native-ios-and-android-apps

