---
layout: blog
title: clojure之旅：读写文件总结
---

从clojure中读取文件的最简单方式是使用slurp和spit函数：

``` clojure
(spit "file.txt" "hello, there!")
(slurp "file.txt")
"hello there!"
```

slurp函数可以从网络地址中获取网页：

``` clojure
(slurp "http://www.google.com")
```

注意slurp和spit只适合读写文本格式，并且文件要比较小，因为他们是一次性地把所有内容读入内存和从内存写入文件中的，这样就不太适合大文件的操作了。

如果想要获得更理想的读取文件性能，使用reader可以提升效率：

``` clojure
(require '[clojure.java.io :as io])
(with-open [r (io/reader "file.txt")]
 (doseq [liene (line-seq r)]
   (println line)))

line1
line2
line3
... ...
```
with-open宏保证在它的作用域范围外资源得到释放，因此在此例中文件不用手动关闭；line-seq从文件流中构建惰性序列（文件中的每一行内容），doseq对文件的每一行进行打印操作。

以下是使用writer对一个文件进行写操作:

``` clojure
(with-open [w (io/writer "file.txt" :append true)]
 (.write w "line4\n"))
```

with-open也可以同时对多个文件进行监管，即是在退出时自动释放资源：

``` clojure
(with-open [r (io/reader "file.txt")
            w (io/writer "file1.txt")]
  (doseq [line (line-seq r)]
    (.write w (str line "\n"))))
```

读取二进制数据可以直接用Java自带的io库,以下是读取一个图片文件并以字节数组返回:

``` clojure
(with-open [input (new java.io.FileInputStream "image.png")
            output (new java.io.ByteArrayOutputStream)]
    (io/copy input output)
    (.toByteArray output))
```

在此例中，我们可以用clojure的input-stream，形如(io/input-stream "image.png")来代替java的(new java.io.FileInputStream "image.png")。

下例中往文件写二进制数据：

``` clojure
(with-open [output (new java.io.FileOutputStream "file.bin")]
   (.write output (.getBytes "hello world.")))
```

再者，我们可以用clojure的output-stream，形如(io/output-stream "file.bin")来代替java的(new java.io.FileOutputStream "file.bin")。


下例从文件读取二进制数据，放到字节数组中，并最终转换成字符串：

``` clojure
(let [file "file.bin"
      output (byte-array (.length (io/as-file file)))]
  (with-open [input (io/input-stream file)]
    (.read input output)
    (apply str (map char output))))

"hello world."
```
