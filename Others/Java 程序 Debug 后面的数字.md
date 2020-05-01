# Debug Java 程序 对象后面的数字是什么？

在 Debug Java 程序的时候，在 IDEA 的 Variables 窗口中经常可以看到类似下图形式的变量：

![Debug 图像](https://i.loli.net/2019/03/29/5c9e11f82febc.png)

`{ConcurrentHashMap$Segment@873}` 中的 873 是 JVM 在调试的时候提供的唯一的 ObjectId，在程序内部是无法访问的。可以参考 Stack Overflow上的一个问答： [Deciphering variable information while debugging Java](https://stackoverflow.com/questions/2322903/deciphering-variable-information-while-debugging-java)

后半部分：`java.util.concurrent.ConcurrentHashMap$Segment@58886ad0[Locked by thread main]` 中的 `58886ad0` 则是当前 Object 的 hashCode 的16禁止表示。

this 的 hashCode:

![this 的 hashCode](https://i.loli.net/2019/03/29/5c9e11f839fd0.png)


this 的 16进制表示:

![this 的16进制表示](https://i.loli.net/2019/03/29/5c9e11f839fd0.png)