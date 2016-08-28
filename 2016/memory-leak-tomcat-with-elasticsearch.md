##1 Tomcat关闭时报错
问题描述：<br>Tomcat在关闭时报ThreadLocal未正常移除，提示很可能内存泄漏风险。<br>
问题解答：<br>ES没有使用Tomcat管理的线程池，导致Tomcat认为存在内存泄漏风险。可以运行profiler进行检验。一般认为无碍。<br>
问题链接：<br>http://stackoverflow.com/questions/27090676/memory-leak-related-issue-with-threadlocals-in-elasticsearch<br>
