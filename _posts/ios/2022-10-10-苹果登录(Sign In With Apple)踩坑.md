---
title: 苹果登录(Sign In With Apple)踩坑
date: 2022-10-10
tags: ["ios"]
category: iOS
math:       true
---

# 场景
最近公司新开发了一个新项目，使用到了苹果登录。但是在和后端联调的过程中，一直报错invalid_grant

```json
{
    "error": "invalid_grant",
    "error_description": "client_id mismatch. The code was not issued to com.xxx.www"
}
```

# 排查
一开始是怀疑客户端的bundle id和后端的client_id设置的不一样，但是经过多次确认后，并不是代码层面的问题。

# 解决
后来经过网上搜索发现了这条问答：[Apple Sign-in: Invalid_grant](https://developer.apple.com/forums/thread/679497), 决定尝试上传到**TestFlight**试试。
在打包的过程中，遇到了与平时打包流程不一样的页面，要求在App Store Connect上创建APP，于是就创建了，并且上传了app。
最后发现TestFlight上的APP和线装的APP都可以登录没有报错了。

# 总结
在id匹配，且代码层级没有bug的情况下，是苹果商店里面没有创建APP所导致的，只要创建了APP即可解决。