+++ 
draft = false
date = 2024-02-24T17:56:47+08:00
title = "使用karabiner实现vim的中英文输入法丝滑切换"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

#vim
## 背景
vim从编辑模式退出到普通模式下的时候，需要手动切换到英文输入法，其他编辑器的vim模式同样存在此问题，我想在vim退出编辑模式时自动切换为英文输入法

实现这个功能有两个思路：
- vim插件
- 修改键盘映射
vim插件的办法这里不做介绍，因为vim插件无法在其他编辑器的vim模式下使用

这里采用[karabiner](https://karabiner-elements.pqrs.org/)修改键位映射来实现，karabiner是一个很方便的键位修改软件，我一直在使用，解决此问题的思路是：修改ESC的键盘映射，实现在按下ESC的同时，修改输入法为英文

## 如何实现
karabiner 允许用户通过json的形式自定义键位映射，点击下面红框部分，可以通过一段json来自定义键位的行为
![](https://pic-1258720617.cos.ap-beijing.myqcloud.com/20240224174646.png)


下面是实现该功能的json配置，其含义是：
在当前输入法不是英文的前提下，按下ESC时，会先按下ESC，同时把输入法切换为英文

这段json的语法是karabiner的json配置语法，可以在karabiner官网找到详细的[karabiner json语法介绍](https://karabiner-elements.pqrs.org/docs/json/complex-modifications-manipulator-definition)，语法还是比较直观的
```json
{
    "description": "ESC: ESC and language to en",
    "manipulators": [
        {
            "type": "basic",
            "conditions": [
                {
                    "type": "input_source_unless",
                    "input_sources": [
                        {
                            "input_source_id": "^com\\.apple\\.keylayout\\.ABC$",
                            "language": "^en$"
                        }
                    ]
                }
            ],
            "from": {
                "key_code": "escape"
            },
            "to": [
                {
                    "key_code": "escape"
                },
                {
                    "select_input_source": {
                        "input_source_id": "^com\\.apple\\.keylayout\\.ABC$",
                        "language": "^en$"
                    }
                }
            ]
        }
    ]
}
```

在karabiner启用这段json配置之后，就可以体验丝滑的vim中英文切换了！