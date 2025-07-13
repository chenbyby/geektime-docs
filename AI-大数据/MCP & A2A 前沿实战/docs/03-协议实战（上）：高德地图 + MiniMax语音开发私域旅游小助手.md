你好，我是黄佳。

欢迎来到 MCP 协议实战的第一课。今天我们将结合AI IDE（以 Cursor 为例），利用 MCP 服务快速搭建一个私人定制的“行旅小助手”系统。具体来说，我们会使用高德地图的地点推荐 MCP 服务获取景点信息，再借助 Minimax 的语音生成 MCP 服务合成导览音频，并最终通过 Agent 自动生成 HTML 页面，实现一个“西安旅游景点推介”的一条龙流程。

这是我们第一次动手实战，因此我特意选择了一个实操性非常强、而且又很实用、能看得见效果的MCP场景，我建议你跟着我一起一步一步的动起手来，感受一下大模型（MiniMax）开发、Cursor（AI Coder或称为AI IDE），以及MCP协议这三个技术栈相结合，构建一个MVP级别的产品是多么容易。

如果你是一个产品设计者或开发者，这些强大的工具组合在一起，会让你觉得自己长出了三头六臂。这将是一次利用了多种MCP服务和最新大模型（gemini-2.5-pro-exp-03-25）以及Cursor Coding Agent的Vibe Code新体验。

![](https://static001.geekbang.org/resource/image/d9/a9/d9c6f82b8ba1fd33296fdb2c9f7949a9.jpg?wh=3209x1837)

## 私域旅游小助手开发流程

这节课操作步骤较多，所以在开始实操之前，先给出你整个课程的流程：

1. 注册并配置高德地图 MCP 服务：获取城市与景点的推荐接口，实现地点信息检索。
2. 注册并配置 Minimax 语音生成 MCP 服务：将文字转为语音，生成导览音频。
3. 在 Cursor 中完成 MCP 服务的接入与调用配置：包括填写密钥、Model 兼容性设置、Agent 自动运行配置等。
4. 编写 Prompt 驱动 Agent：自动获取西安各类景点介绍，并按类别分条整理。
5. 调用 Minimax 生成语音导览：将生成的景点介绍文本自动转为语音文件。
6. 生成并部署 HTML 页面：利用 Agent 将整理好的文本输出为一个静态网页，并部署到线上。

学完这节课之后，你将掌握在 AI IDE（Cursor）中如何集成第三方 MCP 服务的方法，还能快速搭建一个小而美的“城市景点介绍与导览”示例。

## 使用高德开放平台的地图MCP服务

高德地图（amap）官方提供了自己的MCP服务sse接口，要使用它的MCP服务，我们需要在[这里](https://lbs.amap.com/)登录高德开放平台，创建一个应用，并获取相应的key。

### 登录高德开放平台

因为后面的个人实名认证步骤，需要使用支付宝认证，这里直接使用支付宝登录。

![图片](https://static001.geekbang.org/resource/image/1e/4f/1e2947019d2f3a05b97cd25f4792ce4f.png?wh=1920x901)  
![图片](https://static001.geekbang.org/resource/image/f3/6a/f354e98708481180e2462c2434ab746a.png?wh=1920x1032)

### 注册为高德开放平台开发者

下面，我们需要在[这里](https://console.amap.com/dev/id/select)注册自己为高德开放平台开发者。

进入个人控制台，会要求先完成开发者认证，这里选择个人开发者。

![图片](https://static001.geekbang.org/resource/image/95/e0/950d1381a6547605d5448546c5e0f3e0.png?wh=1920x934)

输入邮箱，填入验证码绑定邮箱，并完成个人实名认证。

![图片](https://static001.geekbang.org/resource/image/cc/7e/cc3ea99940ced634757f275cb490e77e.png?wh=1920x1007)

系统提示，已经完成个人认证，你就成为了高德开放平台的开发者。

![图片](https://static001.geekbang.org/resource/image/61/f7/61dd179d7189af49ca29218361ed82f7.png?wh=1920x909)

### 获取高德服务密钥

下一步，是获取高德服务密钥的步骤。刚才提到，要获得密钥，需要先创建一个应用才行。

进入控制台，选择 **应用管理-&gt;我的应用-&gt;创建新应用**。

![图片](https://static001.geekbang.org/resource/image/42/67/425bace0eaac07433b660c1b7dca3867.png?wh=1920x766)

应用类型选择“**出行**”。

![图片](https://static001.geekbang.org/resource/image/11/81/115a6d06316218e194e97226dbda8981.png?wh=1857x873)

创建应用完成后，点击“**添加key**”。

![图片](https://static001.geekbang.org/resource/image/95/80/95030d6a23cafed97610fe7b98824680.png?wh=1920x495)

填写“Key名称”，服务平台选择 “**Web服务**”，勾选已阅读协议，提交创建key。

![图片](https://static001.geekbang.org/resource/image/ca/7d/ca546d51d32b2f2169b7343f83f6c77d.png?wh=1058x1148)

此时，key就创建好了，可以把它复制下来，下一步在Cursor中配置MCP服务时会需要用到。

![图片](https://static001.geekbang.org/resource/image/83/e4/8363316bf44ffcf804fd1a3ae43df4e4.png?wh=1778x352)

高德开放平台每月会有个人学习的减免额度，而且额度较大，因此不需要担心这个旅行助手在你出游过程中信息获取的费用问题。

![图片](https://static001.geekbang.org/resource/image/0a/79/0a52bfed0cd57f253782738d0349f879.gif?wh=552x209)

### 在Cursor中配置高德MCP

下面，进入cursor，选择右上角settings-&gt;**MCP**-&gt;**Add new global MCP server**。

![图片](https://static001.geekbang.org/resource/image/9a/54/9a5d38306e99a58d17c5ec053558a754.png?wh=1776x535)  
复制下面的配置json到mcp.json中，这里需要用自己的amap的key来替换图中的占位符。

```plain
JSON
   {
     "mcpServers": {
       "amap-amap-sse": {
         "url":   "https://mcp.amap.com/sse?key=在高德开放平台获得的应用密钥"
       }
     }
   }
```

![图片](https://static001.geekbang.org/resource/image/fa/40/fa765d30475904b1ba0494a6d6c6f940.png?wh=858x241)

之后我们返回 Cursor 设置界面查看 MCP 服务工具状态。

![图片](https://static001.geekbang.org/resource/image/5d/e4/5d404eee561235771dbed8c4yy5ea9e4.png?wh=1215x440)

如果遇到任何问题，可以点击图右侧的绿色按钮重新Enable这些工具的功能。

## 使用MiniMax大模型的语音服务

配置好了高德地图MCP服务之后，我们再来配置我们这个应用程序中要使用到的下一个MCP服务—— MiniMax 提供的语音服务。

![图片](https://static001.geekbang.org/resource/image/62/ca/62f20664d619533159af28270b5d5eca.png?wh=1634x829)

MiniMax可以提供即时生成的多模态数据。对于MCP产品，目前支持stdio、sse两种传输方式，即可以本地部署或平台托管（这里我将使用ModelScope魔搭平台进行托管），并且提供本地路径、远程url两种处理图片音视频等资源输入输出的方式。

### 注册Minimax开放平台

首先，根据你的情况，通过下面的链接之一，在MiniMax开放平台进行注册。

- [https://minimaxi.com](https://minimaxi.com) 是MiniMax境内的官网
- [https://minimax.io](https://minimax.io) 是MiniMax境外的官网
- [https://api.minimax.chat](https://api.minimax.chat) 是境内服务的域名
- [https://api.minimaxi.chat](https://api.minimaxi.chat) 是境外服务的域名

在[这里](https://platform.minimaxi.com/login)注册MiniMax平台账号，并登录。

![图片](https://static001.geekbang.org/resource/image/b5/d3/b560f6d255b911ba62dea26774bb63d3.png?wh=1772x906)  
填写个人信息注册MiniMax平台

登录MiniMax平台后进入[账户管理](https://platform.minimaxi.com/user-center/basic-information/interface-key)，创建接口密钥。

![图片](https://static001.geekbang.org/resource/image/96/a3/96c7e4b8a7085680cb940ffc6f47b6a3.png?wh=1767x781)

### 在ModelScope平台部署Minimax sse

Minimax官方在 [ModelScope 平台](https://modelscope.cn/mcp/servers/@MiniMax-AI/MiniMax-MCP)提供了在线部署服务。

![图片](https://static001.geekbang.org/resource/image/13/f3/1326715a88e01f8d913a4aec3bd598f3.png?wh=732x958)  
分别填入：

1.  [https://api.minimax.chat](https://api.minimax.chat)

2.  刚才在Minimax开放平台创建的key密钥

3.  本地存放获取音视频的目录（有stdio和url两种模式，这里是必填项，先填一个本地存储生成音视频的目录）

点击“连接”，即在ModelScope部署完成了托管服务，复制这里的完整url。

![图片](https://static001.geekbang.org/resource/image/c0/af/c02ce6df73b0552c472e0326585bd2af.png?wh=855x951)

### 在Cursor中配置Minimax MCP

好，下面我们需要在Cursor中配置Minimax MCP，准备工作有两步。在Model平台托管的MiniMax-MCP配置URL，然后在Minimax开放平台账户管理中，查看并复制GroupID。

![图片](https://static001.geekbang.org/resource/image/0e/28/0e09d9ea46734d7321ec951da2842128.png?wh=1687x413 "Minimax 开放平台中的 GroupID")

我们在Cursor的MCP配置文件中，填入上面两个信息，编辑后保存。

```plain
JSON
   {
     "mcpServers": {
       "MiniMax-MCP": {
         "type":   "sse",
         "url":   "https://mcp.api-inference.modelscope.cn/在ModelScope托管的url/sse",
         "env": {
           "MINIMAX_GROUP_ID":   "1在账户信息里复制的GroupID一串数字"
         }
       },
       "amap-amap-sse": {
         "url":   "https://mcp.amap.com/sse?key=在高德开放平台获得的应用密钥"
       }
     }
   }
```

这里填入刚刚复制的：

1. ModelScope平台托管的Minimax的url
2. Minimax个人的GroupID

![图片](https://static001.geekbang.org/resource/image/db/bf/dbe3c28b808bb31480946a0d72652ebf.gif?wh=552x219)

回到MCP服务列表，打开这个服务的按钮打开，并刷新。等一会儿后，“Tools:”后面的“No Tools Available”会变成显示一系列的可用方法。如果一直显示“No Tools Available”，需要检查url有没有完整复制，账户的GroupID和key是否正确。

![图片](https://static001.geekbang.org/resource/image/19/7c/1901b4844f82dca0441e73d54785267c.gif?wh=552x106)

## 在Cursor中使用MCP服务开发自己的应用

下面我们就要开始在Cursor中使用刚才配置好的两种MCP服务了，不过在这之前，我们还要再做一些MCP兼容性的设置。

### 选择对MCP兼容性好的模型

在Cursor提供的模型中，部分对MCP的兼容性还不好。这里推荐Agent执行能力、对MCP反馈的整合能力都很强的模型：gemini-2.5-pro-exp-03-25。

在下图的Cursor设置中，调整备选的Models。

![图片](https://static001.geekbang.org/resource/image/e8/fc/e80ef0fb1cefa074dc7415a4f26085fc.png?wh=1669x1091)

### 允许自动流畅执行MCP工具调用

在Feature中找到auto-run mode，为了让整个交互过程更加流畅，选择如下设置：

- **勾选**（3）“允许Agent默认不询问直接调用工具”，
- **不勾选**（4）“阻止Agent自动运行MCP工具”

![图片](https://static001.geekbang.org/resource/image/eb/7e/eb90d7e34c63e26d9b77464a9fb1147e.png?wh=1667x1040)

好，上面就是MCP兼容性的设置，下面我们通过提示工程来实现所想要的功能。

### 获取城市景点的介绍

在Cursor的Chat对话框中，首先获取主要旅游景点信息。

```plain
根据高德地图，获取西安市内的主要旅游景点信息|
```

![图片](https://static001.geekbang.org/resource/image/2c/5a/2c55b2d545df6e2a86a14a057bf32c5a.png?wh=1444x356)

Cursor作为MCP Host，将通过gemini-2.5-pro-exp-03-25模型自动在对话中拉取调用MCP工具，完成上面的任务。

![图片](https://static001.geekbang.org/resource/image/59/29/59e63968ceb8af35b7509b97480ce529.png?wh=724x1166)

### **按照类别整理景点介绍**

下一步，再通过提示工程来按照类别，整理景点的介绍。

```plain
帮我对这些景点分类，在百度百科上搜索相关的介绍，对每个景点生成一个详细的介绍，生动活泼有条理，并写入一个markdown文件
```

![图片](https://static001.geekbang.org/resource/image/a1/ca/a1aa30c1b3fae909115dc1c42e77c9ca.png?wh=1289x666)

可以看到Agent在努力地调用MCP服务，自行分条检索信息并汇总。

![图片](https://static001.geekbang.org/resource/image/b7/74/b7cf6f4e398fe7098e40b85a78fd5f74.png?wh=1351x1160)

最后，Agent会把生成的内容保存成新的Markdown文件，各景点信息已经分门别类整理到位。

![图片](https://static001.geekbang.org/resource/image/45/79/45cfaaa0a60a58f626f7b7367853f579.png?wh=1627x1015)

### 调用Minimax生成语音导览

下面通过@或者拖动到对话框的方式引用文件，并在对话中输入以下提示。

```plain
生成各个景点的语音介绍，把得到的语音导览的链接放在标题的下方。
```

![图片](https://static001.geekbang.org/resource/image/e5/1a/e59220yy23084319c2b74e1daae8761a.png?wh=1613x476)  
Agent会识别、嵌入成功调用MCP并正确返回的url，自动修改markdown文件。

![图片](https://static001.geekbang.org/resource/image/38/19/386af07a783d3460a750a28ab0797b19.png?wh=763x1136)

由于刚刚生成的景点介绍文件较长，模型处理时可能会分段分次执行，每次返回解析url的方式可能不同，有时会在MCP Call中显示，有时会显式在输出中作为回复。

![图片](https://static001.geekbang.org/resource/image/24/cd/249709a9067bbe9b02f644d15ca8e1cd.png?wh=1258x1048)

### 网页制作和部署

好，现在你已经得到了每个景点的分类分条的文字和音频介绍，我们要用网页把内容呈现出来。这里我们让Agent按照给定的提示词生成网页并保存。

新建一个文件，叫 “prompt.md”，然后保存下面的内容。

```markdown
Markdown
   #   页面提示词
   ## 网页风格指南
   你是一位国际顶尖的数字景区导览设计师和前端开发专家，擅长将传统景区介绍与现代网页设计完美融合，创造出令人身临其境的视觉体验。请设计高品质景区介绍手机h5网页，将景区信息以生动优雅的方式呈现，让用户在浏览前就能感受到景区魅力。
   **可选设计风格：
   **1. **文化复古风格**
       适合历史文化类景区，使用仿古纸张纹理背景，搭配书法或篆刻风格的标题。色调以米黄、棕色等复古色系为主，点缀传统中国色。排版可参考古籍设计，正文使用简体宋体增强文化氛围。可加入水墨画元素作为分隔符或装饰。文案语调应庄重有文化底蕴，介绍中可穿插诗词或历史典故。
   2. **现代简约风格**
       采用极简设计理念，主色调限制在2-3种，确保视觉统一性。排版注重网格系统，所有元素严格对齐。字体选用无衬线体，增强现代感。景点信息使用卡片式设计，便于用户快速浏览。图标采用线性设计，保持风格一致性。动效应简洁克制，仅用于增强用户体验的关键环节。整体设计追求"少即是多"的理念，让内容成为主角。
   **必要页面组件：**
   1. **景区名称列表**：   
     1. 景区名称
     2. 音频开始按钮：点击音频可以直接播放，播放时可以暂停，暂停的音频还可以继续播放
   2. **景区介绍**：    
     1. 默认收缩，点击景区名称可弹出详细介绍
   ## 技术规范
   * 资源引用：  
     * Tailwind CSS:   `https://lf3-cdn-tos.bytecdntp.com/cdn/expire-1-M/tailwindcss/2.2.19/tailwind.min.css`  
     * Font Awesome:   `https://lf6-cdn-tos.bytecdntp.com/cdn/expire-100-M/font-awesome/6.0.0/css/all.min.css`  
     * 中文字体:   `https://fonts.googleapis.com/css2?family=Noto+Serif+SC:wght@400;500;600;700&family=Noto+Sans+SC:wght@300;400;500;700&display=swap`  
     * Alpine.js:   `https://unpkg.com/alpinejs@3.10.5/dist/cdn.min.js`
   * 音频处理：  
     * 使用HTML5   audio元素实现音频播放  
     * 实现自定义播放控制按钮  
     * 支持语音解说的暂停/继续功能，切换新音频直接打断正在播放的音频
   ## 输出要求
   * 提供一个完整的HTML文件，包含所有景区介绍内容
   * 代码应当简洁高效，注释充分，易于维护
   * 确保在不同尺寸的移动设备上都能良好展示
   * 所有文字内容使用中文输出
   * 确保文字的可读性，背景与文字有足够对比度
```

最后一步，准备好提示词和刚刚的景点介绍内容文件，交给Agent生成HTML页面。

![图片](https://static001.geekbang.org/resource/image/9a/32/9a1947edab12e5b380f6e4e2c859aa32.png?wh=1402x591)

可以看到Agent帮我们快速生成了HTML。

![图片](https://static001.geekbang.org/resource/image/d7/c0/d7bc08097dbaa7a34a874bf1ed4c27c0.png?wh=1303x1103)![图片](https://static001.geekbang.org/resource/image/22/68/22ae5587f11881735325c1a98c451a68.png?wh=1667x1102)

## 网页效果的呈现

下面，我们把这个页面上传到youware.com，进行部署和展示。

![图片](https://static001.geekbang.org/resource/image/3d/55/3d2f6479cfc37c66511469abc3d1e555.png?wh=1677x947)

下面就是实际在线部署的页面。你可以通过[这个链接](https://www.youware.com/project/a87n5fcmtr)来感受一下这个花费2个小时私人定制的旅游助理的效果。点击播放按钮，你将听到导游小助手甜美的播放各个旅游景点的介绍。

怎么样，这是不是一次非常完美的、利用了多种MCP服务和最新大模型（gemini-2.5-pro-exp-03-25）以及Cursor Agent的Vibe Coding体验？

## 思考题

1. ModelScope中的 [MCP 广场](https://modelscope.cn/mcp)已经发布了海量的MCP服务，你可以根据自己的需要，按照我们这节课中介绍的步骤，完成更多个性化的、有特色的应用程序开发。

![图片](https://static001.geekbang.org/resource/image/6d/66/6d01ea2accf65d677df9a014858f6666.png?wh=1920x945)  
2\. 这里我们通过纯提示工程完成了网页的开发和部署，没有写任何代码就生成了所需要的页面。请你通过代码（可以用Cursor或Trae等AI Coder辅助编码）进行自动化的工作，把上面的流程转换成一个有UI人机交互、有前端后端的应用程序，可以自动根据客户的需求来生成景点介绍，播放景点介绍的音频、甚至景点网站等。

欢迎你在留言区和我交流讨论，如果这节课对你有启发，别忘了分享给身边更多朋友。