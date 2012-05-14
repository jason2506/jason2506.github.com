---
layout: post
title: "在 Windows 上架設 Git Server"
date: 2012-05-14 16:18
comments: true
categories: [Note, Git, Gitolite]
---

![Gitolite - 圖片來源：http://rapapa.net/?tag=gitolite](/images/2012-05-14-setup-a-git-server-on-windows/gitolite.jpg)

這其實是大約半個月前的事情，由於工作需要，必須建立一個藏在內部網路的 git server，來統一控管散落在世界各地的<del>龍珠</del>程式碼。雖然我個人是希望能夠架在 FreeBSD 或是 Linux 上面，不過到最後也只要到一台 Windows 的機器來用，也只好將就著先拿來架啦。為了防止之後還需要架一台 server 的時候把安裝方法忘得一乾二淨，就趁現在還殘留著一點點記憶的時候記錄起來。

<!-- more -->

## Step 1. 安裝 Cygwin

首先，這次要裝來用的是 [gitolite](https://github.com/sitaramc/gitolite) 這套使用得滿廣泛的 git server。而在 Windows 上，gitolite 是需要執行在 [Cygwin](http://www.cygwin.com/) 上的。但是在安裝時要注意的是，有些會用到的 package 是 Cygwin 在預設情況下不會安裝的：

* Net > openssh
* Devel > git
* Editors > vim

其中 vim 沒裝應該也無所謂，只不過它本來就是我的愛用編輯器，所以就不加思索直接裝了。

## Step 2. 架設 SSH Server

Cygwin 裝好之後，開啟 Cygwin 並執行 `cyglsa-config`，以在透過 ssh 認證時，能夠藉由金鑰認證，而不需要提供使用者密碼：

    $ cyglsa-config

執行完就照著指示重開機。

接著要在 Cygwin 中設定 ssh server，執行 `ssh-host-config`：

    $ ssh-host-config

然後就可以啟動 ssh server 了：

    $ sc start sshd

接著就是建立一個專門用來當 git server 的 Windows account，於是就從控制台建立一個叫做 *git* 的使用者吧。需要注意的是，建立這個使用者的時候需要勾選「*密碼永久有效*」這個選項。

建好了之後，在 Cygwin 中執行：

    $ mkpasswd -l -u git >> /etc/passwd

這時就可以透過 ssh 來連入 git 的帳號囉：

    $ ssh git@localhost

在一台要當做管理員的 client 端（我是用本機的另一個帳號）用 `ssh-keygen` 產生一對用以做金鑰認證的 private key 與 public key（預設為 `$HOME/.ssh/` 目錄下的 `id_rsa` 與 `id_rsa.pub`）。接著用 `scp` 或是其它任何方式把 `id_rsa.pub` 複製一份到 git 帳號的 home 目錄下，並改名為 `[你的帳號].pub`。這組 public key 待會會被用來設定 gitolite 的管理者帳號。

## Step 3. 安裝 Gitolite

終於要開始裝 gitolite 了。首先先透過 ssh 登入 git 帳號，clone 一份 gitolite 的專案：

    $ git clone git://github.com/sitaramc/gitolite

然後安裝：

    $ ./gitolite/install

這時可以執行 `gitolite` 看看是否安裝成功了。接下來就要用到剛剛的 public key 啦，執行：

    $ gitolite setup -pk [username].pub

然後回到 client 端，我們要 clone 一份用以管理 git server 的 repo（註：如果你的 client 跟 server 不是同一台機器，請自行把 `localhost` 改成 server 的位置）：

    $ git clone git@localhost:gitolite-admin.git

如果有成功 clone 下來，就是成功囉。

## Step 4. 管理使用者與 Repo

若是要建立一個使用者，就同樣要讓該使用者建立自己的金鑰、將他的 public key 交給你，並同樣命名為 `[account].pub`。接著只要把 public key 擺在 admin 剛剛 clone 出來的 `gitolite-admin` repo 下的 `keydir` 目錄就行了。

而若是你有多台電腦，都需要透過同一個帳號連接 git server，可以將不同電腦各自的 public key 擺在 `keydir` 裡的不同子目錄中。

另外，想要新增/修改/刪除 repo 的話，則需要修改 `conf/gitolite.conf` 這個檔案：

```
repo gitolite-admin
    RW+         =   admin

repo testing
    RW+         =   @all
```

其實意義都不會很難猜：`repo [reponame]` 就代表一個 repo。想建立一個新的 repo，就直接照著新增上去：

```
repo my-repo
    RW+         =   admin
```

`RW+` 為存取權限，也可以在後面限定 branch 名稱。要新增可存取的使用者，就在 `=` 後面以空白隔開：

```
repo my-repo
    RW+         =   admin jane
    RW+ master  =   mary
    RW+ dev     =   jason
    R           =   tony
```

也可以建立使用者群組：

```
@manager        =   jane johnson

repo my-repo
    RW+         =   admin @manager
    RW+ master  =   mary
    RW+ dev     =   jason
    R           =   tony
```

在這邊做的任何修改，只要 push 上去就會立刻生效了。

## 後記

其實，原本還想架個 [gitlab](http://gitlabhq.com/) 當做一個方便操作的 web interface，不過在 Cygwin 上沒有可用的套件管理工具，實在難以設置執行環境，嘗試了幾回之後就懶得弄了。反正當前的情況是能用就好，隨便啦XD。

## 參考資料

* [How To Set Up A Git Server On Windows Using Cygwin And Gitolite](http://therightstuff.de/CommentView,guid,b969ea4d-8d2c-42af-9806-de3631f4df68.aspx)
* [Hosting git repositories](http://sitaramc.github.com/gitolite/index.html)
* [Linux 使用 Gitolite 架設 Git Server](http://blog.longwin.com.tw/2011/03/linux-gitolite-git-server-2011/)
* [Switching the user context without password](http://www.cygwin.com/cygwin-ug-net/ntsec.html#ntsec-nopasswd1)
* [使用 SSH 实现 Linux 下的安全数据传输](http://www.360doc.com/content/10/0111/15/146562_13261391.shtml)
