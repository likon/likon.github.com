---
layout: blog
title: emacs作为clojure的开发环境
---

clojure作为lisp的一种方言，用emacs来编辑简直就是绝配了，以下是配置emacs作为clojure的开发环境设置：


### 准备工作

首先第一步就是下载[lein](https://raw.github.com/technomancy/leiningen/stable/bin/lein.bat)工程管理软件，找到该脚本源代码的

{% highlight shell %}
if "x%LEIN_JAR%" == "x" set LEIN_JAR=!LEIN_HOME!\self-installs\leiningen-!LEIN_VERSION!-standalone.jar
{% endhighlight %}

修改成

{% highlight shell %}
if "x%LEIN_JAR%" == "x" set LEIN_JAR="!LEIN_HOME!\self-installs\leiningen-!LEIN_VERSION!-standalone.jar"
{% endhighlight %}

这样修改的原因是因为在windows下的路径有空格，导致安装完好lein以后照样用不了的问题。

然后执行<code>lein self-install</code>进行自安装，在执行命令之前请检查系统有wget或者curl下载软件，否则可能安装不成功（win7的内部有下载的命令，可以跳过）,至此lein应该安装完毕了，执行<code>lein repl</code>试试，clojure的repl应该会出来了，不行的话检查一下脚本吧。

### 设置emacs作为clojure的开发环境

安装nrepl需要用到emacs的package安装管理扩展，因为emacs24自带了package安装管理扩展，所以最好还是直接下载emacs24版本吧，省事。

* 初始化package环境，在.emacs配置文件中加入如下代码，重新启动后会自动加载的：

{% highlight lisp %}
(require 'package)
(add-to-list 'package-archives
             '("marmalade" . "http://marmalade-repo.org/packages/"))
(package-initialize)
{% endhighlight %}

* 使用package下载clojure-mode插件，<code>M-x package-install clojure-mode</code>，在.emacs配置文件中加入如下代码：

{% highlight lisp %}
;; (require 'paredit) if you didn't install via package.el
(defun turn-on-paredit () (paredit-mode 1))
(add-hook 'clojure-mode-hook 'turn-on-paredit)
{% endhighlight %}

这里最好还是把paredit插件装一下，因为它会帮助我们很好编辑lisp语言的，enjoy it!

* 使用package安装nrepl插件，<code>M-x package-install [RET] nrepl [RET]</code>，然后在.emacs配置文件加入如下代码： 

{% highlight lisp %}
(add-to-list 'load-path "~/emacs.d/vendor")
(require 'nrepl)
{% endhighlight %}

### 开始emacs的clojure之旅

* 在某个目录下使用<code>lein new test-project</code>来创建工程，然后用emacs打开project.clj工程文件，这样emacs就在当前文件夹下了，然后在emacs执行<code>M-x eshell[RET]</code>来开一个终端，在此终端下执行<code>lein repl</code>来开一个clojure的repl，再<code>M-x nrepl[RET]</code>，会有提示填写服务器IP和端口的，端口在开启一个repl时会给定的，这样就可以在emacs的repl里使用nrepl了。

* 在clojure的源码中进行跳转，在以上步骤都完成后，就可以打开源码文件进行跳转了，使用<code>M-.</code>来定位函数变量等，<code>M-,</code>返回。

* 详情可参见[nrepl的github](https://github.com/kingtim/nrepl.el)，在此就不赘述了。
