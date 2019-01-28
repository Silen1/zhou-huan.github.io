---
layout: post
title: "一些常用工具"
subtitle: ''
author: "周欢"
header-style: text
tags:
  - tools
---

### [StrokesPlus](http://www.strokesplus.com/)

Free Mouse Gestures for Windows XP to Windows 10

### [cmder](http://cmder.net/)

Cmder is a software package created out of pure frustration over the absence of nice console emulators on Windows. It is based on amazing software, and spiced up with the Monokai color scheme and a custom prompt layout, looking sexy from the start.

```bash
cmder /register user

cmder /register all
```

### powershell插件 - 显示所在分支及该分支added/modified/deleted/conflict的数量

* 设置权限

    `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Confirm`

* 使用PowerShellGet安装

    `PowerShellGet\Install-Module posh-git -Scope CurrentUser`

* v全局导入posh-git

    `Add-PoshGitToProfile -AllHosts`

### fiddler自定义规则

#### 修改请求

    static function OnBeforeRequest(oSession: Session) {
        if (oSession.uriContains("https://......")) {
            oSession.utilReplaceInRequest('"prodId": 1016995', '"prodId": 11111')
        }
    }

#### 修改响应

    static function OnBeforeResponse(oSession: Session) {
        if (oSession.uriContains("......")) {
            oSession.utilReplaceInResponse('"status": "1"', '"status": "2"')
        }
    }

#### 修改url传参

    static function OnBeforeRequest(oSession: Session) {
        if (oSession.uriContains("......")) {
            oSession.fullUrl = oSession.fullUrl.Replace("prodId=1017159", "prodId=111111");
        }
    }

### vscode插件

* [Align](https://marketplace.visualstudio.com/items?itemName=steve8708.Align)

Align text in vscode like the atom-alignment package.

* [Auto Close Tag](https://marketplace.visualstudio.com/items?itemName=formulahendry.auto-close-tag)

Automatically add HTML/XML close tag, same as Visual Studio IDE or Sublime Text.

* [Auto Rename Tag](https://marketplace.visualstudio.com/items?itemName=formulahendry.auto-rename-tag)

Auto rename paired HTML/XML tag.

* [Autoprefixer](https://marketplace.visualstudio.com/items?itemName=mrmlnc.vscode-autoprefixer)

Parse CSS and add vendor prefixes automatically.

* [change-case](https://marketplace.visualstudio.com/items?itemName=wmaurer.change-case)

Quickly change the case (camelCase, CONSTANT_CASE, snake_case, etc) of the current selection or current word.

* [EditorConfig for VS Code](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig)

EditorConfig Support for Visual Studio Code.

* [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)

Integrates ESLint into VS Code.

* [expand-region](https://marketplace.visualstudio.com/items?itemName=letrieu.expand-region)

expand selection , It works similar to ExpandRegion for Emacs and “Structural Selection” (Control-W) in the JetBrains IDE's (i.e. IntelliJ IDEA).

* [Git History](https://marketplace.visualstudio.com/items?itemName=donjayamanne.githistory)

View git log, file history, compare branches or commits.

* [GitLens - Git supercharged](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)

Supercharge the Git capabilities built into Visual Studio Code — Visualize code authorship at a glance via Git blame annotations and code lens, seamlessly navigate and explore Git repositories, gain valuable insights via powerful comparison commands, and so much more.

* [Guides](https://marketplace.visualstudio.com/items?itemName=spywhere.guides)

An extension for more guide lines

* [Launcher](https://marketplace.visualstudio.com/items?itemName=ilich8086.launcher)

An easy way to launch your terminal, tools, scripts and batches from Visual Studio Code.

* [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer)

Launch a development local Server with live reload feature for static & dynamic pages.

* [Material Icon Theme](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme)

Material Design Icons for Visual Studio Code.

* [npm Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.npm-intellisense)

Visual Studio Code plugin that autocompletes npm modules in import statements.

* [Path Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.path-intellisense)

Visual Studio Code plugin that autocompletes filenames.

* [Redux DevTools](https://marketplace.visualstudio.com/items?itemName=jingkaizhao.vscode-redux-devtools)

vscode redux devtools wrapper.

* [stylelint](https://marketplace.visualstudio.com/items?itemName=shinnn.stylelint)

Modern CSS/SCSS/Less linter.

* [Template Literal Editor](https://marketplace.visualstudio.com/items?itemName=plievone.vscode-template-literal-editor)

Use Ctrl+Enter to open ES6 template literals and other configured multi-line strings or heredocs in any language in a synced editor, with language support (HTML, CSS, SQL, shell, markdown etc).

* [Vetur](https://marketplace.visualstudio.com/items?itemName=octref.vetur)

Vue tooling for VS Code.

* [VueHelper](https://marketplace.visualstudio.com/items?itemName=oysun.vuehelper)

Code snippets for Vue,Vue-router and Vuex.
