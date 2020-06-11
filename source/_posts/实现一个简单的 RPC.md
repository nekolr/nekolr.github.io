---
title: 实现一个简单的 RPC
date: 2020/6/11 16:16:0
tags: [Java,RPC]
categories: [其他]
---

一个最简单的 RPC 需要满足几个基本的要求。首先是**通信**，一般可选的有 HTTP 和 TCP，这里选择 TCP，直接使用 Java Socket 处理通信。然后就是**寻址**，也就是如何找到要调用的方法。这里根据服务消费者提供的基本调用信息，然后利用 Java 的反射机制进行调用。服务消费者在进行远程调用时就像调用本地方法一样的效果则依靠 Java 的动态代理机制来实现。最后是**参数序列化和反序列化**，这里使用最简单的 Java 原生的序列化机制。

<!--more-->

首先我们需要一个注册中心，所有的服务提供者都需要先向注册中心进行服务注册，这样服务消费者才能“发现”并进行消费。

```java
public interface Registry {

    void stop();

    void start();

    void register(Class<?> serviceInterface, Class<?> impl);
}
```

```java
@Getter
public class RegistryCenter implements Registry {

    private ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    private static Map<String, Class> serviceRegistry = new HashMap<>();

    private int port;

    private ServerSocket serverSocket;

    public RegistryCenter(int port) {
        this.port = port;
    }

    @Override
    public void stop() {
        this.executor.shutdown();
        if (this.serverSocket != null) {
            try {
                this.serverSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public void start() {
        try {
            this.serverSocket = new ServerSocket(this.port);
            System.out.println("Registry Center Start...");
            // 服务端需要在 accept 方法上自旋
            while (true) {
                // 有一个请求就创建一个线程去处理
                this.executor.execute(new ServiceTask(this.serverSocket.accept()));
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (this.serverSocket != null) this.serverSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public void register(Class<?> serviceInterface, Class<?> impl) {
        this.serviceRegistry.put(serviceInterface.getName(), impl);
    }

    static class ServiceTask implements Runnable {

        private Socket socket;

        public ServiceTask(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try (ObjectInputStream input = new ObjectInputStream(this.socket.getInputStream())) {
                // 反序列化
                String serviceName = input.readUTF();
                String methodName = input.readUTF();
                Class<?>[] paramTypes = (Class<?>[]) input.readObject();
                Object[] arguments = (Object[]) input.readObject();
                Class<?> serviceClass = serviceRegistry.get(serviceName);
                if (serviceClass == null) {
                    throw new ClassNotFoundException(serviceName + " not found.");
                }
                // 调用指定类的方法
                Method method = serviceClass.getMethod(methodName, paramTypes);
                Object result = method.invoke(serviceClass.newInstance(), arguments);
                // 将结果传输给客户端
                try (ObjectOutputStream output = new ObjectOutputStream(this.socket.getOutputStream())) {
                    output.writeObject(result);
                    output.flush();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

注册中心应该是一个服务端，需要长期在某个端口进行监听，在有客户端连接过来时就分出一个线程去处理。将消费者提供的序列化后的服务名称、方法名称、方法参数类型和参数进行反序列化，然后利用反射机制调用对应的服务，并将调用结果序列化后传输给消费者（客户端）。

在客户端，当我们调用接口的一个方法时，接口的实现类需要将我们的调用信息通过网络传递给注册中心，然后等待调用结束后再接收调用结果返回，如果为每个接口都创建这么一个类，显然是不合适的，因此最好的方式就是使用动态代理机制。我们创建一个工具类，提供一个获取远程对象的方法，在该方法中实现 InvocationHandler 接口，将实现类的上述逻辑放在其中。

```java
public class ProxyUtil {

    @SuppressWarnings("unchecked")
    public static <T> T getRemoteProxyInstance(Class<T> serviceInterface, InetSocketAddress address) {

        return (T) Proxy.newProxyInstance(serviceInterface.getClassLoader(), new Class<?>[]{serviceInterface},
                ((proxy, method, args) -> {
                    try (Socket socket = new Socket()) {
                        socket.connect(address);
                        try (ObjectOutputStream output = new ObjectOutputStream(socket.getOutputStream())) {
                            // 序列化
                            output.writeUTF(serviceInterface.getName());
                            output.writeUTF(method.getName());
                            output.writeObject(method.getParameterTypes());
                            output.writeObject(args);
                            output.flush();
                            try (ObjectInputStream input = new ObjectInputStream(socket.getInputStream())) {
                                // 阻塞读
                                return input.readObject();
                            } catch (Exception e) {
                                e.printStackTrace();
                            }
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    return null;
                }));
    }
}
```

接下来就可以开始使用了，使用时有一点要特别注意，由于我们使用的是 Java 原生的序列化和反序列化机制，因此我们方法的参数和返回值都需要实现 `java.io.Serializable` 接口。

```java
@Getter
@Setter
@ToString
public class User implements Serializable {

    private String id;
    private String username;

    public User(String id, String username) {
        this.id = id;
        this.username = username;
    }
}
```

```java
public interface UserService {
    User getUserById(String id);
}
```

```java
public class UserServiceImpl implements UserService {
    @Override
    public User getUserById(String id) {
        return new User(id, "saber");
    }
}
```

```java
public class Application {
    public static void main(String[] args) throws InterruptedException {
        Registry registry = new RegistryCenter(8888);
        // 注册服务
        registry.register(UserService.class, UserServiceImpl.class);
        // 另起一个线程运行注册中心
        new Thread(() -> registry.start(), "server").start();

        UserService userService = ProxyUtil.getRemoteProxyInstance(UserService.class, new InetSocketAddress("localhost", 8888));
        User user = userService.getUserById("12");
        System.out.println(user);

        // 关闭注册中心
        TimeUnit.SECONDS.sleep(2);
        registry.stop();
    }
}
```