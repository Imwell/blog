---
title: Java并发工具包-Semaphore
date: 2021-03-08 10:05:01
tags:
    - Java
    - 并发
categories:
    - 并发
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/java.png
---
# Semaphore
Semaphore(信号量)是一个线程同步工具，主要用于在一个时刻允许多个线程对共享内容进行并行操作的场景，比如限流。它需要多个线程获取访问共享资源的许可证。
## Semaphore内部处理逻辑

- 创建Semaphore，规定许可证数量
- 线程申请许可证，判断是否有许可证，没有则阻塞，有则进行执行（tryAcquire方法不会阻塞）
- 许可证数量减少，线程操作共享资源，操作完释放许可证
- 许可证数量增加，回到第二部

{%  codeblock lang:java %}
    
    public static void main(String[] args) {
        
        LoginService loginService = new LoginService(MAX_PLAYER);
        // 最多只允许10个人登录，开启20个线程测试
        for (int i = 0; i < 20; i++) {
            new Thread(() -> {
                if (loginService.login(Thread.currentThread().getName())) {
                    try {
                        TimeUnit.SECONDS.sleep(5);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    loginService.logout(Thread.currentThread().getName());
                }
            }).start();
        }

    }

    class LoginService {
    
        Semaphore semaphore;
    
        public LoginService(int max) {
            this.semaphore = new Semaphore(max, true);
        }
    
        public boolean login(String name) {
    
            if (semaphore.tryAcquire()) { // 非阻塞，第一次进不来，就永远进不来
                System.out.println(name + " login success");
                return true;
            }
            System.out.println(name + " login failed");
            return false;
        }
    
        public void maxLogin(String name) throws InterruptedException {
            semaphore.acquire(); // 阻塞，永远都可以进来
            System.out.println(name + " login success");
        }
    
        public void logout(String name) {
            semaphore.release();
            System.out.println(name + " login failed");
        }
    }
{% endcodeblock %}

## Semaphore的主要方法

`public Semaphore(int permits)`：构造方法，传入许可证数量
`public Semaphore(int permits, boolean fair)`：构造方法，传入许可证数量，并规定是否为公平同步器
`tryAcquire()`：尝试获取许可证，并立即返回结果，非阻塞方法
`tryAcquire(long timeout, Timeunit unit)`：增加超时参数，阻塞方法
`tryAcquire(int permits)`：获取对应数量的许可证，可用数量小于或者总数量小于获取数量都返回false
`acquire()`：获取一个许可证，该方法会阻塞线程，直到获取到许可证
`acquire(int permits)`：获取对应数量的许可证，并阻塞，一定要很小心使用这个方法
`release()`：释放许可证。该方法使用也要注意，只有保证了正确并非抛出异常的情况下获取到了许可证，才能释放，
`release(int permits)`：释放指定数量的许可证
`int availablePermits()`：当前Semaphore还有多少许可证

## 总结
Semaphore并发工具类提供了一个很方便的操作共享资源的方法，如果许可证数量为1时，可以将它当成一个锁来使用。但是，该方法只能保证对共享资源的并发访问控制，对于共享资源的临界区以及线程安全性，Semaphore并不会提供任何保证。

