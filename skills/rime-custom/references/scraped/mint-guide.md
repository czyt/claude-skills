[Skip to content](https://www.mintimate.cc/zh/guide/#VPContent)

[![](https://www.mintimate.cc/favicon.svg)oh-my-rime输入法](https://www.mintimate.cc/zh/)

搜索试试`` `K`

Main Navigation [更新日志](https://www.mintimate.cc/zh/changeLog/) [关于](https://www.mintimate.cc/zh/teamInfo.html) [主页](https://www.mintimate.cc/) [效果演示](https://www.mintimate.cc/zh/demo/) [配置教程](https://www.mintimate.cc/zh/guide/)

简体中文

[English](https://www.mintimate.cc/en/guide/)

[github](https://github.com/Mintimate/oh-my-rime)

简体中文

[English](https://www.mintimate.cc/en/guide/)

Appearance

[github](https://github.com/Mintimate/oh-my-rime)

Menu

此页面

Sidebar Navigation

## 配置教程

[引导](https://www.mintimate.cc/zh/guide/)

[安装rime](https://www.mintimate.cc/zh/guide/installRime.html)

[导入薄荷输入法](https://www.mintimate.cc/zh/guide/importMint.html)

[配置覆写和定制](https://www.mintimate.cc/zh/guide/configurationOverride.html)

[自定义默认激活方案](https://www.mintimate.cc/zh/guide/defaultActivationScheme.html)

[Emoji配置(OpenCC)](https://www.mintimate.cc/zh/guide/openccEmoji.html)

[模糊拼音设置](https://www.mintimate.cc/zh/guide/fuzzyPinyin.html)

[设置语言模型](https://www.mintimate.cc/zh/guide/languageModel.html)

[符号输入配置](https://www.mintimate.cc/zh/guide/symbolsInput.html)

[Lua功能扩展](https://www.mintimate.cc/zh/guide/luaExtensions.html)

[输入法快捷键](https://www.mintimate.cc/zh/guide/shortcutKeys.html)

[输入个性定制](https://www.mintimate.cc/zh/guide/customizationInput.html)

[多设备同步](https://www.mintimate.cc/zh/guide/deviceSync.html)

[AI MCP 服务](https://www.mintimate.cc/zh/guide/mcpService.html)

[\[可选\]问题答疑](https://www.mintimate.cc/zh/guide/faQ.html)

[![](https://www.mintimate.cc/image/oops.webp)](https://wwads.cn/page/whitelist-wwads)

[检测到您开启了广告拦截器… 广告有时候确实很烦人… 但请把它关掉í﹏ì，这样我们的服务器可能才不会饿死👻，十分感谢!\\
\\
Oops! Detected that you have turned on an ad blocker... Ads can sometimes be really annoying... But, if possible, please turn it off so that our servers don't starve to death 👻, thank you very much!](https://wwads.cn/page/whitelist-wwads) [万维广告](https://wwads.cn/page/end-user-privacy "万维广告 ~ 让广告更优雅,且有用")

此页面

- [基本概念](https://www.mintimate.cc/zh/guide/#%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5 "基本概念")
- [推荐教程](https://www.mintimate.cc/zh/guide/#%E6%8E%A8%E8%8D%90%E6%95%99%E7%A8%8B "推荐教程")
- [鸣谢](https://www.mintimate.cc/zh/guide/#%E9%B8%A3%E8%B0%A2 "鸣谢")
- [交流群](https://www.mintimate.cc/zh/guide/#%E4%BA%A4%E6%B5%81%E7%BE%A4 "交流群")

[Sponsor a scoop of matcha powder for better posts. \\
\\
(●'◡'●)ﾉ♥](https://ifdian.net/a/mintimate)

# 配置教程 [​](https://www.mintimate.cc/zh/guide/\#%E9%85%8D%E7%BD%AE%E6%95%99%E7%A8%8B)

现在，开始我们的薄荷拼音配置；其实配置的方法很简单，只需要下载并安装当前系统的rime输入法，之后导入薄荷拼音并重新部署即可。

当然，如果先根据自己的喜好进行深度定制，对于新手可能有一些难度；建议在配合本文档的情况下，配合搜索引擎和rime官方文档进行了解。

如果认为本文档或者薄荷拼音对你很有帮助，可以请我喝咖啡:

👉 如果认为本文档或者薄荷输入法(方案)对你很有帮助，可以请我喝咖啡 ☕

![WebChart Recognise](https://www.mintimate.cc/assets/recognise.DYWJGUjE.webp)

Tips:如果你有爱发电账号，那么也可以访问 [爱发电平台](https://ifdian.net/a/mintimate)

> 请务必备注『薄荷拼音』或者『oh-my-rime』，对于捐赠咖啡☕️的用户，将进入『 [鸣谢](https://www.mintimate.cc/zh/guide/#%E9%B8%A3%E8%B0%A2)』内(●'◡'●)ﾉ♥

![oh-my-rime](https://www.mintimate.cc/image/demo/guideAbstract.webp)

## 基本概念 [​](https://www.mintimate.cc/zh/guide/\#%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)

薄荷是一个输入方案，RIME 其实是一个算法核心，要组成完整的客户端输入法，还需要一个输入法框架， **三者共同组成一个输入法**。

鼠须管(macOS)和小狼毫(Windows)，可以理解为是输入法框架和 RIME 算法核心的结合体，只需要安装方案，就可以使用； 但是 Android 和 Linux 上的 Fcitx5 是输入法框架，需要安装 RIME 算法核心，才能安装 RIME 的方案，关系如下：

键盘事件传递

加载配置

使用词典

应用规则

界面设置

运算结果

渲染界面

用户选择

输出内容

⌨️ 用户按键操作

🖥️ 输入法框架

「如: ibus/fcitx」

⚙️ RIME 核心引擎

「小狼毫/鼠须管自带」

👑 方案配置

「薄荷/万象/雾凇」

📚 词典数据

📐 语法规则

词库处理

🎨 界面主题

配色/布局

👀 候选栏/状态窗

实时反馈

📝 文本输出到应用

s98dic

我制作了一张卡通图片，用于说明三者的相互依赖关系：

![输入方案和框架](https://www.mintimate.cc/image/guide/guideToRime.webp)

## 推荐教程 [​](https://www.mintimate.cc/zh/guide/\#%E6%8E%A8%E8%8D%90%E6%95%99%E7%A8%8B)

这里推荐一些教程，用于辅助深度定制：

- [Rime官方Wiki文档](https://github.com/rime/home/wiki)
- [Rime方案定制指南](https://github.com/LEOYoon-Tsaw/Rime_collections/blob/master/Rime_description.md)
- [雾凇拼音](https://dvel.me/posts/rime-ice/)

## 鸣谢 [​](https://www.mintimate.cc/zh/guide/\#%E9%B8%A3%E8%B0%A2)

与此同时，薄荷拼音的配置，离不开网上的丰富教程；薄荷拼音大量借鉴了：

- 雾凇拼音: [https://github.com/iDvel/rime-ice](https://github.com/iDvel/rime-ice)

感谢爱发电的支持用户：

| 时间 | 平台 | 用户 | 支持💵 | 留言 |
| --- | --- | --- | --- | --- |
| 2026/03/10 | 微信赞赏 | 微信用户: AA～电脑城-弯儿师傅 | 100¥ | 薄荷拼音很好用，电脑手机都用上了 |
| 2025/12/25 | 微信赞赏 | 微信用户: cyrasafia | 30¥ | -- |
| 2025/12/15 | 微信赞赏 | 微信用户 | 50¥ | 感谢维护，全平台换了薄荷拼音方案 |
| 2025/12/09 | 微信赞赏 | 微信用户: 周彦铭 Silver | 10¥ | 挺好用, gj! |
| 2025/11/23 | 微信赞赏 | 微信用户: ^^ | 50¥ | 输入法我只用薄荷 |
| 2025/11/22 | 微信赞赏 | 微信用户: 何唯是真名er | 10¥ | 昨天用上的，谢谢了。这么棒的作品 |
| 2025/11/13 | 微信赞赏 | 微信用户: tyfanchz | 20¥ | 感谢大大们维护的好用的方案！ |
| 2025/11/04 | 微信赞赏 | 微信用户: 培公啊 | 10¥ | -- |
| 2025/10/05 | 爱发电 | QQ用户: Lii(892\*\*\*084) | 100¥ | 薄荷输入法简直丝滑无比 |
| 2025/10/25 | 爱发电 | QQ用户: 白水(290\*\*\*894) | 100¥ | 期待您的回复 |
| 2025/09/03 | 微信赞赏 | 微信用户: Derek | 20¥ | Derek |
| 2025/07/24 | 爱发电 | 微信用户: Xinyi | 50¥ | 膜雨月大佬，教程写得很有帮助，希望能交流一次呀 |
| 2025/06/08 | 微信赞赏 | 微信用户: 东方 | 28¥ | 希望可以导入“王码” |
| 2025/06/06 | 微信赞赏 | 微信用户: 「匿名用户」 | 10¥ | 薄荷输入法 |
| 2025/04/10 | 微信赞赏 | 微信用户: fix u | 10¥ | 感谢作者开源🙏 |
| 2025/01/03 | 微信赞赏 | QQ用户:凌(873\*\*534) | 5¥ | 感谢在QQ群无私的帮助 |
| 2025/01/04 | 爱发电 | [爱发电用户\_NVKP](https://ifdian.net/u/b5636c3aca4d11ef8f5a5254001e7c00) | 15¥ | oh-my-rime |
| 2024/10/26 | 微信赞赏 | 微信用户: Jacian | 10¥ | 很好用的方案，希望一直维护下去 |
| 2024/10/20 | 微信赞赏 | 微信用户: Torjoy | 20¥ | 感谢大佬 |
| 2024/09/27 | 微信赞赏 | RIME输入法交流小群群友 | 10¥ | 叮，你要的奶茶 |
| 2024/09/06 | 微信赞赏 | 微信用户: YANGZhitao | 20¥ | 很好用! 感谢维护这套方案 |
| 2024/08/21 | 微信赞赏 | 微信用户: ZY | 20¥ | 谢谢你维护这套方案 |
| 2024/06/30 | 爱发电 | [奶茶不加冰](https://ifdian.net/u/802ed17a36bf11efa4db52540025c377) | 20¥ | 手机上已经用上了，体验非常好，感谢作者。 |
| 2024/06/12 | 爱发电 | [爱发电用户\_15aca](https://ifdian.net/u/15aca804289b11efa13952540025c377) | 36¥ | oh-my-rime |
| 2024/06/11 | 爱发电 | [爱发电用户\_9d84b](https://ifdian.net/u/9d84b3ac280011efa1d352540025c377) | 20¥ | oh-my-rime, perfect! |
| 2024/05/31 | 爱发电 | [爱发电用户\_sYNg](https://ifdian.net/u/c428e6701f1a11efab4a5254001e7c00) | 20¥ | 一个月前就准备请up来杯奶茶了~今天是时候兑现一下了！感谢up的薄荷拼音真的非常好用~我已经全平台跟进啦~ |
| 2024/05/28 | 微信 | 公众号用户: 晶码战士 | 50¥ | 薄荷输入法👍👍👍 |
| 2024/04/28 | 爱发电 | [爱发电用户\_UkCK](https://ifdian.net/u/8717bcc8054511efbfc052540025c377) | 20¥（一杯奶茶） | oh-my-rime |
| 2024/03/15 | 爱发电 | [爱发电用户\_520f9](https://ifdian.net/u/520f9e12e26111eeaa3a5254001e7c00) | 50¥（KFC） | 辛苦了，希望能持续更新下去！ |
| 2024/01/22 | 爱发电 | [爱发电用户\_8b769](https://ifdian.net/u/8b769b02b8c111ee928952540025c377) | 50¥（KFC） | Hi, 感谢维护oh-my-rime |

## 交流群 [​](https://www.mintimate.cc/zh/guide/\#%E4%BA%A4%E6%B5%81%E7%BE%A4)

如果你有QQ，并且想一起探索和探讨Rime:

- QQ群: 703260572 (拒绝广告，支持闲聊~) 倡导互助。若进群只等他人秒回问题，甚至抱怨本该自查（如看文档）的基础问题无人解答，请慎重加入或留群。

> 注意: 这是交流群，不是客服群；本身也是开源配置，也不存在售后和客服关系。

当然，你也可以在 GitHub [Issues](https://github.com/Mintimate/oh-my-rime/issues) 和 [Discussions](https://github.com/Mintimate/oh-my-rime/discussions) 内进行讨论。同样注意 GitHub 的 Issues 等规范，维护者也尽可能保持中立和开源态度，一些 Issues 可能处理后就关闭(这很正常，也符合常规 Issues 处理)，不要跑去已经关闭的 Issues 吐槽其他功能，甚至吐槽维护者“不承认”问题，这没有意义，commit 提交历史在哪里摆着。

> 我有没有商业，“不承认“什么问题有什么意思么？ 真的是”沉默的大多数不会表达感谢，只有少数遇到困难的才会不断抱怨“？可以当作没看到，处理一些 Issues 由感而发 😫。

Last updated: 3/16/26, 11:13 AM

Pager

[下一篇安装rime](https://www.mintimate.cc/zh/guide/installRime.html)

[Powered by creativity and powered by Mintimate](https://www.mintimate.cn/)

[闽ICP备2021000722号-3](https://beian.miit.gov.cn/) \| [闽公网安备 35021102001843号](http://www.beian.gov.cn/)

本项目 CDN 加速及安全防护由 [Tencent EdgeOne](https://edgeone.ai/zh?from=github) 赞助