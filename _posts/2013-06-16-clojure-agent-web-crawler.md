---
layout: blog
title: clojure之旅：用agent实现网络爬虫
---

使用clojure的agent实现网络爬虫相当的简单，在这里实现的网络爬虫程序是很基本的，但是麻雀虽小，五脏俱全。

首先，我们先实现一些基本的用来解释网页的实用函数。links-from函数解释网页的链接，并返回链接序列(因为使用了for的序列生成器，所以该序列为惰性序列（lazy sequence))；words-from函数解释网页的单词，对每个单词进行小写转换，返回单词序列，此例中使用[enlive包](https://github.com/cgrand/enlive)来解析html数据的，以下是两个函数的实现：

{% highlight clojure %}
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

为了能更好和最大化使用系统资源，我们将建立一堆agent来实现网页抓取、解析网页（提取出网页链接和单词）、合并单词的有限状态机的转换过程，在使用agent时，需要多考虑agent维护的状态和对该状态进行什么样的处理。在多数情况下，我们可以使用agent来实现有限状态机（即是agent维护的当前状态和对该状态进行什么样的处理），在此网络爬虫中每个agent的状态机如下所示：
![网页抓取解释的有限状态机](http://likon.github.com/images/clojure-web-crawler.jpg)

* 此三个动作将对应下面将要介绍的get-url,process和handle-results函数。

一个agent的状态扮演的角色已经很明显了：在从网页链接获取网页之前，agent需要获得URL地址；在提取出网页链接和单词之前，agent需要获得网页内容；在合并单词之前，agent需要获得解析的结果。既然agent的状态不多，我们可以简化成为每个agent提供当前状态和下一步将要做的事情。

说了这么多，已经迫不及待看看怎么实现这些agent了吧。根据有限状态机图示，首先所有agent的初始化状态应该是维护链接的队列url-queue，因为下一步我们需要需要从url-queue中获取一个链接来抓取网页数据，这样我们可以这样定义每个agent：

{% highlight clojure %}
; 因为即将要用到get-url函数，所以要提前定义
(declare get-url)
; 每个agent都有::t字段，表示下一步将要执行的动作
(def agents (set (repeatedly 25 #(agent {::t #'get-url :queue url-queue}))))
{% endhighlight %}

get-url函数将会等待线程安全队列获取网页链接，如果等到后，获取该链接的网页。然后返回包含该网页地址、网页内容和下一步要执行的动作（即是process）的map：

{% highlight clojure %}
(declare run process handle-results)
;; ^::blocking是函数的meta data，表示该函数式阻塞式的
;; 对使用者来说有用
(defn ^::blocking get-url
  [{:keys [^BlockingQueue queue] :as state}]
  ;; 从url-queue中获取一个链接，一直等待状态
  (let [url (as-url (.take queue))]
    (try
      ;; 如果该网页已经抓去过，则原封不动返回
      (if (@crawled-urls url)
        state
        ;; 否则抓取该链接的网页，并返回包含内容和下一步动作的map
        {:url url
         :content (slurp url)
         ::t #'process})
      ;; 出错时原封不动返回
      (catch Exception e
        state)
      ;; 这里保证有限状态机不会停止，最后启动动作，继续执行下一步动作
      ;; *agent*表示当前的agent
      (finally (run *agent*)))))
{% endhighlight %}


process函数对网页进行解析，使用最先定义的links-from和words-from实用函数来提取网页链接和提取单词，并统计每个单词的词频，最后返回所有网页链接序列、单词以及其词频组合的序列、当前网页链接和下一步将要执行的动作（即是handle-results)的map：

{% highlight clojure %}
(defn process
  [{:keys [url content]}]
  (try
    ;; 从字串转换成StringReader对象
    (let [html (enlive/html-resource (java.io.StringReader. content))]
      {::t #'handle-results
       :url url
       ;; links包含了网页中所有的链接
       :links (links-from url html)
       ;; 从单词序列中建立以单词为key，词频为value的map
       :words (reduce (fn [m word]
                        ;; fnil表示如果key刚添加的话则从1开始计数
                        (update-in m [word] (fnil inc 0)))
                      {}
                      (words-from html))})
    ;; 继续执行下一步动作 
    (finally (run *agent*))))
{% endhighlight %}

handle-results函数更新三种状态数据：往crawled-urls中添加已经抓取网页的链接，往url-queue中添加所有的新连接，往word-freqs合并单词和词频。由图所知，handle-results处理之后需要需要继续从url-queue中获取得到链接来获取网页，然后提取网页和单词，最后又执行此函数，如此循环往复，所以该函数需要返回原始的agent状态：

{% highlight clojure %}
(defn ^::blocking handle-results
  [{:keys [url links words]}]
  (try
    ;; 添加已经抓去过的网页集
    (swap! crawled-urls conj url)
    ;; 往队列中添加所有的链接，其他等待的agent就可以获取一个链接
    ;; 其他的动作就可以王下走了
    (doseq [url links]
      (.put url-queue url))
    ;; 更新单词和词频，词频用+来合并
    (swap! word-freqs (partial merge-with +) words)
    
    ;; 最终返回原始的状态
    {::t #'get-url :queue url-queue}

    ;;继续执行下一步动作
    (finally (run *agent*))))
{% endhighlight %}

注意到每个执行动作最后都有`(run *agent*)`，其实*agent*在其他线程中是没有定义的，因为这里所有的动作都在thread pool的运行线程中执行，所以*agent*是执行动作线程下的当前agent。

综上，这就是使用agent来无间断运行的通常做法。在这个网络爬虫程序中，run函数是将每个agent下一步执行的动作排队，既然每个动作都知道了如何去执行下一步动作，为什么还要添加间接调用执行动作的run函数呢，原因有二：

* run函数可以检查各个执行函数的元属性（meta data）的::blocking属性，以此判断该动作是否阻塞，这样就可以判断使用agent的send和send-off函数来执行agent的下一步动作。

* run还可以检查某个agent是否停掉的状态，通过往agent追加元属性(meta data)的::paused原属性的值达到该目标。

以下是主程序：

{% highlight clojure %}
;; 检查agent是否被停掉
(defn paused? [agent] (::paused (meta agent)))

(defn run
  ([] (doseq [a agents] (run a)))
  ([a]
     (when (agents a)
       (send a (fn [{transition ::t :as state}]
                 ;; 检查当前agent是否处于停止状态
                 (when-not (paused? *agent*)
                   ;; 根据执行动作的::blocking属性来调用send-off还是send
                   (let [dispatch-fn (if (-> transition meta ::blocking)
                                       send-off
                                       send)]
                     ;; 在threading pool线程执行的send或send-off操作
                     (dispatch-fn *agent* transition)))
                 state)))))
{% endhighlight %}

可以停止掉agent的执行相当重要，因为我们不可能让它们无间断的运行下去。因为我们可以改变agent的pause属性来停止执行动作或者清除掉agent的pause属性来重启执行：

{% highlight clojure %}
(defn pause
  ([] (doseq [a agents] (pause a)))
  ([a] (alter-meta! a assoc ::paused true)))

(defn restart
  ([] (doseq [a agents] (restart a)))
  ([a]
     (alter-meta! a dissoc ::paused)
     (run a)))

{% endhighlight %}

目前为止所有的基础建设已经搭建好了，现在可以写个测试程序来测试它们了，测试程序如下所示：

{% highlight clojure %}
(defn test-crawler
  [agent-count starting-url]
  ;; 建立所有的agent，用来测试用的
  (def agents (set (repeatedly agent-count 
                               #(agent {::t #'get-url :queue url-queue}))))
  ;; 清除url队列
  (.clear url-queue)

  ;; 清空状态
  (swap! crawled-urls empty)
  (swap! word-freqs empty)

  (.add url-queue starting-url)
  (run)
  (Thread/sleep 60000)
  (pause)
  [(count @crawled-urls) (count url-queue)])
{% endhighlight %}

运行的结果如下：
![运行结果](http://likon.github.com/images/web-crawler-agent-result.png)
这是用一个agent来执行和用25个agent来执行的效果，结果用25个agent执行的话在1分钟内获取的链接和单词数远远超过用一个agent执行的结果，这就是concurrent的好处，最大化使用系统资源。

最后代码可以从[这里](https://github.com/likon/clojure-exercise)获得。
