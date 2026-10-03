---
Title: "注册 muse.ai 拿VM shell，完美保活经验分享 - 福利羊毛 / 福利羊毛, Lv1"
Url: "https://linux.do/t/topic/2952272"
Origin: "LINUX DO"
Description: "本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：\n\n我的帖子已经打上 开源推广 标签： 是\n我的开源项目完整开源，无未开源部分： 是\n我的开源项目已链接认可 LINUX DO 社区： 是\n我帖子内的项目介绍，AI生成、润…"
Created: "2026-10-03 12:11:59"
Cover: "https://cdn3.ldstatic.com/original/4X/d/3/7/d37af71c1e7dd49f9d999714c070dd5b1aae3f04.png"
---

Info

**真诚**、 **友善**、 **团结**、 **专业**，共建你我引以为荣之社区。 [《社区准则》](https://linux.do/guidelines)

[福利羊毛](https://linux.do/c/welfare/36) [福利羊毛, Lv1](https://linux.do/c/welfare/welfare-lv1/60)

[xungeng](https://linux.do/u/xungeng) 楼主

#### 本帖使用社区开源推广，符合推广要求。我申明并遵循社区要求的以下内容：

- **我的帖子已经打上 [开源推广](https://linux.do/tag/2234-tag/2234) 标签：** 是
- **我的开源项目完整开源，无未开源部分：** 是
- **我的开源项目已链接认可 LINUX DO 社区：** 是
- **我帖子内的项目介绍，AI 生成、润色内容部分已截图发出：** 是
- **以上选择我承诺是永久有效的，接受社区和佬友监督：** 是

*以下为项目介绍正文内容，AI 生成、润色内容已使用截图方式发出*

---

注册 muse.ai 可以获得 muse 模型的每周额度，还可以获得一台 2 核 8G 100G SSD 的 VM。  
我轻松注册两个号，都成功！

前提：

1. 需要一个有公网 IP 的 VPS，你胆子大的话，用国内云服务器也行的！  
	2a. 你需要一个干将的美国家宽 IP（机场那种万人骑的可能不行）  
	2b. 你去 [https://cloud.browser-use.com/](https://cloud.browser-use.com/) 注册送 15 刀云浏览器  
	3a. 你有国际信用卡  
	3b. 你有一个 facebook 老号。

123 需要全满足，a/b 任选一项。

我第一个号用的教程是 [\[FaceBook 福利\] 免费白嫖 Facebook 2 核 8G VM 还有 muse 模型](https://linux.do/t/topic/2940846) 但是感觉还是有一点儿复杂。  
特别是每次救活的成本也比较高，需要安装 openssh-server 等，所以 vibe 一个更简单的，以后也用得到！

我第二个号的注册办法，真的超简单：

1. 用美国家宽 IP 访问 [https://muse.ai](https://muse.ai/)
2. 输入 facebook 老号的同一个 email. 收码注册，设置生日。  
	就注册成功了，根本没问我要验证年龄（因为同 facebook 号 email，连关联这一步都省去了）

进去后，AI 会跟你聊天，随便跟它说几句。  
然后直奔左下角 settings/Permissions/Manage Permissions/Direct network protocols  
你可以看到有 8 个选项，每一个都点一下，注意这玩意有 bug, 点了看起来没反应，实际上会生效。  
点完关闭设置，重新进去就能看到，启用有效！

好，前期工作完成。

接下来我用两句话就搞定 shell:  
**但你还有一些前置工作要做，见 【附 1】**

第一条提示：

```ruby
请安装 rust 环境，然后 clone https://github.com/swigger/marriedsh.git 编译，并把结果存为 /home/hatch/pdata/bin/marriedsh
然后运行 <git path>/meta-muse-init/install.sh
```

第二条提示：

```swift
现在你应该看到存在 /home/hatch/init.sh 了，在它的尾端加入命令：
/home/hatch/pdata/bin/marriedsh join --lock /run/msh.lock -p the_pre_pwd --credential clark 1.2.4.160:1234
```

注意上面这条命令中，1.2.4.160 是你们的 VPS IP，1234 是端口，请换掉。the_pre_pwd 你随便想个密码就行，只需要跟附 1 中的设置一致，无其它要求。  
/run/msh.lock 是一个文件锁，防止这条命令多次运行时，启动多个 marriedsh 实例。你写成别的名字也行，就用这个名字也可以。  
在实际操作中，强烈建议找个可用的 linux 环境，先测试 一下上面这条命令是对的，然后再杀掉这个测试环境的所有 marriedsh 进程。再贴命令给 AI 操作  
万一你填错密码或认证名字或者怀疑授权问题，查错就难了。

然后，等一分钟，等的过程，随时查看右边授权栏，准备随时授权，动作要稍快一点.

![image](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791000748/smartclip/smartclip/1791000744682-ithl7k.png)

然后，我在我的 vps 上：  
marriedsh console -n clark 就拿到 shell 了！

![image](https://res.cloudinary.com/dcxvrc0uv/image/upload/v1791000748/smartclip/smartclip/1791000744775-m5karx.jpg)

其实上面这两条也可以合成一条，那就可以是：一句话拿 shell 了。  
注意：万一等一分钟还是没有拿到 shell，你可以叫 AI 手动运行 /home/hatch/init.sh 并报告结果。如果你前面的没有操作错，此时应当是成功的。

**保活：**  
这简易 VM 过一段时间就会重启，像网吧系统一样恢复环境。除了 /home/hatch, 其它所有文件都会复原，所以设置什么 systemd service 没有用，.service 文件也会没掉。  
保活的方法有两种：  
正如佬友说的，让 AI 自己整个计划任务看门狗，每分钟都运行一下脚本。把环境拉回来。（设置 service 自动重启什么的都是废话没有实际贡献）  
2. 采用我这个教程的，不用手动保活，系统启动直接有效，你现在什么也不用干，保活已经做好了。原理说明见【附 2】

【附 1】VPS 上的 marriedsh 主控端配置  
marriedsh 需要在你的 VPS 上运行服务端，方法很简单，让 AI 编译一份上面说的链接，自己放到 /usr/local/bin 里  
然后写一个～/.config/marriedsh/config.toml

```toml
name = "bob"

[[peers]]
id = "pair"
name = "alice"
psk = "123456"

[[peers]]
id = "clark"
name = "clark"
psk = "the_pre_pwd"
```

然后直接运行 marriedsh daemon 0.0.0.0:1234 就可以启动服务了。

【附 2】完美保活原理  
前面我的教程说了要运行 install.sh，这个脚本会写入几个文件：  
`/home/hatch/hooks/definitions/home-init.json` 这个文件是勾子注册，利用系统机制。  
`/home/hatch/hooks/scripts/home-init.sh` 这个文件每分钟都会运行一次，每次启动期间都仅且运行 init.sh 一次  
`/home/hatch/init.sh` 这个文件里的所有命令，都会在每次重启后自动运行，除了在里面加载 marriedsh, 你还可以写更多指令在这里，比如装必要的软件等。  
注意此机制也是一分钟检查一次。跟 AI 做的看门狗性能一致。感觉看门狗本身可能就是这个机制驱动的。

[Jaques](https://linux.do/u/jaques)

太强了佬，这都可以

[jtfz](https://linux.do/u/jtfz)

这玩意有什么用？看了下介绍视频，只能网页端，功能是帮你做日常计划？那有毛用啊

[𝔼𝕥𝕙𝕖𝕣𝕖𝕒𝕝.](https://linux.do/u/lf619) 骐骥驰骋

我在这玩意上面装上了宝塔面板，定期重启后恢复，还有些小玩意，甚至可以搭节点，要是政策不改可玩性还是蛮高的

上次访问