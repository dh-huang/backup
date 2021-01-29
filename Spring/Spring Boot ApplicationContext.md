# Spring Boot ApplicationContext

IOC容器

##### AnnotationConfigApplicationContext

##### ServletWebServerApplicationContext

为什么SpringBoot中main方法执行完毕后程序一直在运行？

**JVM进程退出**：

所有**非Daemon线程完全终止**，那么只要保证SpringBoot进程中包含1个以上的**非Daemon线程**就可以保证进程不会终止，比如`embed tomcat`启动的时候，会调用到`TomcatWebServer.initialize()`方法，该方法里会调用`startDaemonAwaitThread()`，

```java
private void startDaemonAwaitThread() {
  Thread awaitThread = new Thread("container-" + (containerCounter.get())) {

    @Override
    public void run() {
      // 调用org.apache.catalina.core.StandardServer#await，保证该线程一直运行
      TomcatWebServer.this.tomcat.getServer().await();
    }

  };
  awaitThread.setContextClassLoader(getClass().getClassLoader());
  awaitThread.setDaemon(false);
  awaitThread.start();
}


```



