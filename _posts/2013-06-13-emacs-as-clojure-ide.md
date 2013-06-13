---
layout: blog
title: emacs��Ϊclojure�Ŀ�������
---


clojure��Ϊlisp��һ�ַ��ԣ���emacs���༭��ֱ���Ǿ����ˣ�����������emacs��Ϊclojure�Ŀ����������ã�


### ׼������

���ȵ�һ����������[lein](https://raw.github.com/technomancy/leiningen/stable/bin/lein.bat)���̹���������ҵ��ýű�Դ�����

<code>
if "x%LEIN_JAR%" == "x" set LEIN_JAR=!LEIN_HOME!\self-installs\leiningen-!LEIN_VERSION!-standalone.jar
</code>

�޸ĳ�

<code>
if "x%LEIN_JAR%" == "x" set LEIN_JAR="!LEIN_HOME!\self-installs\leiningen-!LEIN_VERSION!-standalone.jar"
</code>

�����޸ĵ�ԭ������Ϊ��windows�µ�·���пո񣬵��°�װ���lein�Ժ������ò��˵����⡣

Ȼ��ִ��<code>lein self-install</code>�����԰�װ����ִ������֮ǰ����ϵͳ��wget����curl���������������ܰ�װ���ɹ���win7���ڲ������ص��������������,����leinӦ�ð�װ����ˣ�ִ��<code>lein repl</code>���ԣ�clojure��replӦ�û�����ˣ����еĻ����һ�½ű��ɡ�

### ����emacs��Ϊclojure�Ŀ�������

��װnrepl��Ҫ�õ�emacs��package��װ������չ����Ϊemacs24�Դ���package��װ������չ��������û���ֱ������emacs24�汾�ɣ�ʡ�¡�

* ��ʼ��package��������.emacs�����ļ��м������´��룬������������Զ����صģ�

<code>
(require 'package)
(add-to-list 'package-archives
             '("marmalade" . "http://marmalade-repo.org/packages/"))
(package-initialize)
</code>

* ʹ��package����clojure-mode�����<code>M-x package-install clojure-mode</code>����.emacs�����ļ��м������´��룺

<code>
;; (require 'paredit) if you didn't install via package.el
(defun turn-on-paredit () (paredit-mode 1))
(add-hook 'clojure-mode-hook 'turn-on-paredit)
</code>

������û��ǰ�paredit���װһ�£���Ϊ����������Ǻܺñ༭lisp���Եģ�enjoy it!

* ʹ��package��װnrepl�����<code>M-x package-install [RET] nrepl [RET]</code>��Ȼ����.emacs�����ļ��������´��룺 

<code>
(add-to-list 'load-path "~/emacs.d/vendor")
(require 'nrepl)
</code>

### ��ʼemacs��clojure֮��

* ��ĳ��Ŀ¼��ʹ��<code>lein new test-project</code>���������̣�Ȼ����emacs��project.clj�����ļ�������emacs���ڵ�ǰ�ļ������ˣ�Ȼ����emacsִ��<code>M-x eshell[RET]</code>����һ���նˣ��ڴ��ն���ִ��<code>lein repl</code>����һ��clojure��repl����<code>M-x nrepl[RET]</code>��������ʾ��д������IP�Ͷ˿ڵģ��˿��ڿ���һ��replʱ������ģ������Ϳ�����emacs��repl��ʹ��nrepl�ˡ�

* ��clojure��Դ���н�����ת�������ϲ��趼��ɺ󣬾Ϳ��Դ�Դ���ļ�������ת�ˣ�ʹ��<code>M-.</code>����λ���������ȣ�<code>M-,</code>���ء�

* ����ɲμ�[nrepl��github](https://github.com/kingtim/nrepl.el)���ڴ˾Ͳ�׸���ˡ�
