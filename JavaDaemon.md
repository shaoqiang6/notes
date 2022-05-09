# Java Daemon

```java
public class DaemonTest {
    public static void main(String[] args) {
        Runnable run = () -> {
            try {
                TimeUnit.DAYS.sleep(1L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
        Thread thread = new Thread(run);
        // 守护线程, 默认是false, true时jvm会在执行完打印语句之后退出, false时jvm不会退出
        thread.setDaemon(true);
        thread.start();
        System.out.println("main 结束");
    }
}
```

Java 里面 Daemon 线程的定义：如果虚拟机里面只有 Daemon 线程运行时，虚拟机就会退出。

- 通常守护线程是有用户线程创建或者 JVM 创建，用来辅助工作

- 主线程不是守护线程（main 线程）

- 示例中 main 线程结束后仅剩守护线程在运行，进程退出
