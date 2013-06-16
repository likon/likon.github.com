---
layout: blog
title: clojure之旅：用agent实现网络爬虫
---

使用clojure的agent实现网络爬虫相当的简单，在这里实现的网络爬虫程序是很基本的，但是麻雀虽小，五脏俱全。

首先，我们先实现一些基本的用来解释网页的使用函数。links-from函数解释网页的链接，并返回链接序列(因为使用了for的序列生成器，因为该序列为惰性序列（lazy sequence))；words-from函数解释网页的单词，对每个单词进行小写转换，返回单词序列，此例中使用[enlive](https://github.com/cgrand/enlive)来分析html数据的clojure包，以下是两个函数的实现：

{% highlight clojure linenos %}
;; 引用网页解释的clojure包
(require '[net.cgrand.enlive-html :as enlive])
;; 引用clojure核心包，只使用lower-case函数，即转换单词小写
(use '[clojure.string :only (lower-case)])
(import '(java.net URL MalformedURLException))

(defn- links-from
  [base-url html]
  ; 删除空的链接， link为html内容中的a标记
  (remove nil? (for [link (enlive/select html [:a])]
                 ; 如果此link有attrs属性和href属性，href即是网址
                 (when-let [href (-> link :attrs :href)]
                   (try
                     ; 生成一个URL对象
                     (URL. base-url href)
                     ; 跳过错误的网址URL
                     (catch MalformedURLException e))))))

(defn- words-from
  [html]
  ; chunks就是未经过滤的字串序列
  (let [chunks (-> html
                   (enlive/at [:script] nil)
                   ; 选择body的所有的text-node
                   (enlive/select [:body enlive/text-node]))]
    (->> chunks
         ; 对每个字串分解出单词，使用正则表达式re-seq来分割单词
         ; mapcat 对每一字串分解出来的单词序列合并，返回一个单词序列
         (mapcat (partial re-seq #"\w+"))
         ; re-matches匹配时返回匹配的数据，否则返回nil
         ; 这样就可以用remove删除和数字匹配的字串，得到只有英文单词的序列
         (remove (partial re-matches #"\d+"))
         (map lower-case))))

{% endhighlight %}


我们使用三个状态数据集来维护网页抓取的数据，它们分别是:

* 我们将使用java线程安全的队列来维护将要被下载和分析的网页链接，名字叫做url-queue，紧跟着从网页链接得到网页数据后，我们将会...

* 根据网页数据找到所有的链接，并把已经抓取过的网页链接放到crawled-urls中，未抓取的网页放到url-queue中，最后，我们将会...

* 分解出网页中所有的单词，并统计单词出现的次数，最后我们会和word-freqs和并单词以及词频。

三个状态数据集分别用以下表示：

{% highlight clojure %}
(def url-queue (LinkedBlockingQueue.))
(def crawled-urls (atom #{}))
(def word-freqs (atom {})))
{% endhighlight %}

