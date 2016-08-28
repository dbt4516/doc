##1 关于TransportClient
TransportClient自带一个连接池并按需打开，单例地使用它是正确的。<br>
来源：<br>
https://discuss.elastic.co/t/memory-leak-related-issue-with-threadlocals-in-elasticsearch/20905/4<br>

##2 Tomcat关闭时报内存泄漏warning
问题描述：<br>Tomcat在关闭时报ThreadLocal未正常移除，提示很可能内存泄漏风险。<br>
问题解答：<br>ES没有使用Tomcat管理的线程池，导致存在内存泄漏风险。<br>
问题链接：<br>http://stackoverflow.com/questions/27090676/memory-leak-related-issue-with-threadlocals-in-elasticsearch<br>

##3 热部署网页应用导致TransportClient内存泄漏
这是上面的warning对应的实例。网页应用关闭时TransportClient没有被释放（原因是TransportClient不归tomcat管理），无法GC，最终导致内存永久代OOM。<br>
注意改进点：<br>在ContextListener的初始化中初始化client，在context关闭时关闭client。<br>
原文链接（图文并茂，推荐阅读）：<br>http://wpcertification.blogspot.jp/2014/12/how-to-use-elasticsearch-from-web.html



