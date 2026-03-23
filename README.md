---
date: 2021-10-15 00:00:00
permalink: /pages/4caf10/
author: 
  name: Northboat
  link: https://github.com/Northboat
title: 远程关机 - Sockets + Redis
---

## 基于 Java Sockets 实现

> Servlet / JSP + Java Sockets

设计思路

- Socket 网络通信
- Runtime 类

将 Socket 服务端部署在云服务器上；控制客户端部署在网页，负责给服务端发送命令指示；被控制客户端部署在本地，时刻监听服务端给自己传达的信息

### Socket 服务器

SocketThread 类，通信主要功能实现

~~~java
public class ServerThread extends Thread{

    private InputStream in = null;
    private OutputStream out = null;

    private Socket socket = null;

    private String command = null;

    public ServerThread(Socket socket, String command){ this.socket = socket; this.command = command; }

    public String getCommand(){
        return command;
    }

    public void getMessage() throws IOException {
        in = socket.getInputStream();
        ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
        byte[] buffer = new byte[2084];
        int len = 0;
        while((len = in.read(buffer)) != -1){
            byteOut.write(buffer, 0, len);
        }
        command = byteOut.toString();
        System.out.println("这里是服务器，接收到命令：" + command);
        byteOut.close();
        socket.shutdownInput();
    }

    public void sendMessage() throws IOException {
        out = socket.getOutputStream();
        out.write(command.getBytes("GBK"));
        socket.shutdownOutput();
    }


    @Override
    public void run() {
        try {
            //读取客户端信息
            if(command == null){
                getMessage();
            }else{
                sendMessage();
            }
        } catch (Exception e) {
            // TODO: handle exception
            System.out.println("该程序靠此bug运行");
            //e.printStackTrace();
        } finally{
            //关闭资源
            try {
                if(out != null)
                    out.close();
                if(in != null)
                    in.close();
                if(socket != null)
                    socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}
~~~

执行代码

其中count很重要，用于记录接收到命令的次数，当上个任务以及完成后重置命令为空，等待下次命令送达

~~~java
public class Server {

    private static int count = 0;
    //防止无限循环警告
    @SuppressWarnings("InfiniteLoopStatement")
    public static void main(String[] args) {
        try {

            // 创建服务端socket
            ServerSocket serverSocket = new ServerSocket(8088);
            // 创建客户端socket
            Socket socket;
            //储存命令
            String command = null;
            //循环监听等待客户端的连接
            while(true){
                if(count % 2 == 0){
                    command = null;
                    count = 0;
                }
                // 监听客户端
                socket = serverSocket.accept();

                ServerThread thread = new ServerThread(socket, command);
                thread.start();
                System.out.println(count);
                TimeUnit.SECONDS.sleep(2);
                if(thread.getCommand() != null){
                    count++;
                    command = thread.getCommand();
                    System.out.println(command);
                }
                
                //System.out.println(command);
                InetAddress address = socket.getInetAddress();
                System.out.println("当前客户端的IP：" + address.getHostAddress() + ":" + socket.getPort());
            }
        } catch (Exception e) {
            // TODO: handle exception
            e.printStackTrace();
        }
    }
}
~~~

### Web 服务

依赖，pom.xml

~~~xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.11</version>
        <scope>test</scope>
    </dependency>

    <!-- https://mvnrepository.com/artifact/javax.servlet/javax.servlet-api -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
~~~

xml 配置，web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="3.0" xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
	http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd">
  <display-name>Archetype Created Web Application</display-name>


  <servlet>
    <servlet-name>Controller</servlet-name>
    <servlet-class>com.Controller</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>Controller</servlet-name>
    <url-pattern>/exec</url-pattern>
  </servlet-mapping>

</web-app>
```

简单的 servlet，Conroller.java

~~~java
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.net.Socket;

public class Controller extends HttpServlet{
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        req.setCharacterEncoding("utf-8");
        String pwd = req.getParameter("pwd");
        if(pwd.equals("011026")){
            try{
                //向服务端发送信息，想办法把这个字符串存起来
                Socket socket = new Socket("39.106.160.174", 8088);
                //Socket socket = new Socket("localhost", 8088);
                socket.getOutputStream().write("shutdown".getBytes());
                socket.shutdownOutput();
                resp.sendRedirect("/Remote-Controller/ShutdownSuccessfully.jsp");
            }catch (Exception e){
                e.printStackTrace();
            }
        }else{
            System.out.println("密码错误");
            resp.sendRedirect("/Remote-Controller");
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
~~~

index.jsp

~~~html
<html>
<body style="text-align: center" marginheight="300px">
<title>Remote Controller</title>
<form action="${pageContext.request.contextPath}/exec" method="post">
    <h1>Welcome To NorthernBoat's Remote-Controller</h1><br><br>

    <br><h4>ShutDown U PC</h4><input type="password" name="pwd"><br><br>

    <input type="submit" value="exec">
</form>
</body>
</html>
~~~

ShutdownSuccessfully.jsp

~~~html
<%--
  Created by IntelliJ IDEA.
  User: NorthBoat
  Date: 2021/10/5
  Time: 15:47
  To change this template use File | Settings | File Templates.
--%>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>ShutdownSuccessfully</title>
</head>
<body style="text-align: center" marginheight="350px">
<h1>Shutdown Successfully!</h1>
<a href="/Remote-Controller">Go Back</a>
</body>
</html>
~~~

### 客户端 Client

主函数，就是死循环监听 socket 服务器端口

~~~java
public class Client {
    public static Socket socket;
    public static InputStream is;
    public static BufferedReader reader;

    public static void exec() throws IOException {
        try{
            // 和服务器创建连接并获取命令
            while(true) {
                System.out.println("hahaha");
                socket = new Socket("39.106.160.174", 8088);
                //socket = new Socket("localhost", 8088);
                socket.setSoTimeout(3000);
                /*socket.getOutputStream().write("Client请求指示".getBytes());
                socket.shutdownOutput();*/
                // 从服务器接收的信息
                is = socket.getInputStream();
                reader = new BufferedReader(new InputStreamReader(is));
                String info = null;
                if ((info = reader.readLine()) != null) {
                    if (info.equals("shutdown")) {
                        break;
                    }
                }
            }
            reader.close();
            is.close();
            socket.close();
        }catch (SocketTimeoutException e){
            exec();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        try {
            exec();
        }  catch (IOException e) {
            e.printStackTrace();
        }  finally {
            //exec.shutdown();
            System.out.println("shutdown");
        }
    }
}
~~~

关机实现

~~~java
public class exec {
    public static void shutdown(){
        try{
            Runtime.getRuntime().exec("shutdown -s -t 0");
        } catch (IOException e){
            e.printStackTrace();
        } finally {
            System.out.println("bye");
        }
    }
}
~~~

### Issues

网页部署：同样使用docker部署

~~~bash
docker run -it -d --name controller -p 8082:8080 tomcat
~~~

其中 -p 指令，第一个是宿主机端口，第二个是容器端口，该命令将二者映射

Linux 后台运行 jar 包：后台运行 jar 包并把控制台消息输出到 log.txt 文件中

~~~bash
nohup java -jar Server.jar & > log.txt 
~~~

查找后台运行的jar包

~~~bash
jps -l
~~~

安全退出（pid为进程号）

~~~bash
kill pid
~~~

打包 jar ：使用idea集成的功能对代码进行打包

~~~
File ——> Project Structure ——> Artifacts
~~~

注意配置Main函数入口

jar 转 exe：exe4j & innosetup 打包 jar 为 exe 文件

## 基于 Redis 共享内存实现

> SpringBoot 2 + Redis

### 客户端监听 Listener

> 基于Jedis实现消息共享

依赖

~~~xml
<dependencies>
    <!--jedis依赖-->
    <!-- https://mvnrepository.com/artifact/redis.clients/jedis -->
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>4.0.1</version>
    </dependency>

    <!--一部分slf4日志，jedis依赖于此-->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-nop</artifactId>
        <version>1.7.6</version>
    </dependency>

</dependencies>
~~~

listener：与远端`Redis`通信，设置每4s循环一次

- getStatus(String token)/getStatus()：获取token的值(String类型)，没有返回null
- setToken()：设置Token以及作用时间，用Scanner获取用户输入，用一个while循环捕捉不合规范的输入，并要求重新输入，返回值为`int`，用于判断是否设置成功
- close()：关闭jedis连接
- flush()：刷新Redis数据库
- listening()：一个while循环，首次进入打印当前状态，状态即Redis中当前Token对应的值，当收到不同于`alive`的值或值为`null`时退出循环，返回值`status`，交由`Executor`处理

~~~java
public class Listener {

    private String token;
    private Jedis jedis;
    private String url = "39.106.160.174";
    private int port = 6379;


    public void setHost(String url, int port){
        this.url = url;
        this.port = port;
    }

    private int initRedis(){
        try{
            jedis = new Jedis(url, port);
            jedis.auth("011026");
        } catch (redis.clients.jedis.exceptions.JedisConnectionException e){
            return -1;
        }
        return 200;
    }



    // 获取当前用户Token状态
    public String getStatus(){
        return jedis.get(this.token);
    }
    // 获取传入的Token状态，在设置Token时使用，用于避免重复设置
    public String getStatus(String token){
        return jedis.get(token);
    }



    // 监听当前Token状态，间隔5s
    public String listening() throws InterruptedException{
        boolean firstEnter = true;
        String status;
        while(true) {
            // getStatus获取Token的value
            status = getStatus();
            if(firstEnter){
                System.out.println("当前状态: " + status);
                firstEnter = false;
            }
            // 若不为alive，返回当前状态，交由Executor处理
            if(status==null || !status.equals("\"alive\"")){
                break;
            }
            // 测试用，模拟收到命令
            //jedis.set(token, "ipconfig");
            TimeUnit.SECONDS.sleep(4);
        }
        return status;
    }


    // 将查询信息返回到Redis中，给用户去读
    public void response(String resp){
        jedis.set(token, resp);
    }


    // 设置Token
    public int setToken() {
        // 初始化redis连接，并检查状态，主动catch网络异常
        int init = initRedis();
        if(init != 200){ return init; }

        // 准备获取用户输入
        Scanner scanner = new Scanner(System.in);
        String token;

        // 设置Token
        System.out.print("请设置你的Token: ");
        while(true) {
            token = scanner.nextLine();
            String status = getStatus(token);
            if(status == null){
                break;
            }
            System.out.print("该Token已被占用，请重试: ");
        }
        this.token = token;
        String set = jedis.set(this.token, "\"alive\"");
        // 若设置失败，直接返回，退出程序
        if(!set.equals("OK")){
            return 501;
        }
        //设置默认时间为1min，防止用户突然退出一直占用Token
        jedis.expire(this.token, 60);

        // 设置生效时间，循环处理输入格式异常
        int seconds;
        System.out.print("请设置Token生效时间(单位: 分钟)(请在一分钟之内完成设置): ");
        while(true){
            String sec = scanner.nextLine();
            try{
                seconds = Integer.parseInt(sec);
            }catch (NumberFormatException e){
                System.out.print("输入格式有误，请重新输入: ");
                continue;
            }
            break;
        }

        long expired = jedis.expire(this.token, seconds*60);
        // 若设置时间返回不为1，即失败，返回502，交给Checker处理
        if(expired != 1){
            return 502;
        }

        System.out.println("设置成功!");
        return 200;
    }
    
    // 关闭连接
    public void close(){
        jedis.close();
    }

    // 刷新数据库
    public void flush(){
        jedis.flushDB();
    }
}
~~~

Executor：执行类，处理`Listener`返回的不同状态

- shutdown()：关机`Runtime.getRuntime().exec("shutdown -s -t 10")`
- clean()/interrupted()：提示用户信息，然后`System.exit()`
- ipConfig()：执行`ipconfig`，获取控制台输出，截取字符串ip并返回
- getIp(String config)：截取字符串
- exec(String status)：集合以上功能，处理不同`status`

~~~java
public class Executor {

    // 关机
    private static void shutdown(){
        try{
            System.out.println("当前状态: \"shutdown\"");
            System.out.println("收到关机指令，已清除Token，十秒后将关闭计算机...");
            Runtime.getRuntime().exec("shutdown -s -t 10");
        } catch (IOException e){
            e.printStackTrace();
        }
        System.out.println("bye");
        System.exit(1);
    }

    // 异常中断
    private static void interrupted(){
        System.out.println("消息丢失，或Token已失效，即将退出程序...");
        System.exit(0);
    }

    // 获取ip地址
    private static String ipConfig(){
        try{
            System.out.println("当前状态: \"sending Ipv4\"");
            Process pro = Runtime.getRuntime().exec("ipconfig");
            InputStream in = pro.getInputStream();

            ByteArrayOutputStream bos = new ByteArrayOutputStream();

            //读取缓存
            byte[] buffer = new byte[2084];
            int length = 0;
            while((length = in.read(buffer)) != -1) {
                bos.write(buffer, 0, length);//写入输出流
            }
            in.close();//读取完毕，关闭输入流
            String config = new String(bos.toByteArray());

            //获取ipv4地址字符串并返回
            return getIp(config);
        } catch (IOException e){
            e.printStackTrace();
        }
        return null;
    }
    // 从ip信息中截取ipv4地址
    public static String getIp(String config){
        int index = config.indexOf("IPv4")+7;
        StringBuilder ip = new StringBuilder();
        boolean found = false;
        while(true){
            char c = config.charAt(index++);
            //System.out.print(c);

            //找到ip地址的起点，开始录入字符串
            if(c==':' && !found){
                found = true;
                continue;
            }

            //当在起点后，既不是数字，也不是'.'，也不是空格，说明已经过了结尾，直接退出
            if(found && !Character.isDigit(c) && c!='.' && c!=' ') {
                //System.out.println(c);
                break;
            }

            //录入ip地址
            if(found){
                ip.append(c);
            }

        }
        return "\"" + ip.toString().trim() + "\"";
    }

    // 机主清除Token
    private static void clean(){
        System.out.println("当前状态: \"Token has been cleaned\"");
        System.out.println("Token已被人为清除，即将退出程序");
        System.exit(1);
    }


    // 这里只告知服务端收到命令，重置操作给服务端执行
    public static void exec(String status, Listener listener){
        if(status == null){
            listener.close();
            interrupted();
        }else if(status.equals("\"shutdown\"")){
            //告知服务器已收到，继续执行之后操作
            listener.response("\"received\"");
            listener.close();
            shutdown();
        }else if(status.equals("\"ipconfig\"")){
            listener.response(ipConfig());
        }else if(status.equals("\"clean\"")){
            listener.response("\"received\"");
            listener.close();
            clean();
        }

        //System.out.print("即将退出程序...");
    }

}
~~~

Checker：这个类很简单，就是根据set函数的返回值判断一些设置是否成功并作出相应提示，若为成功返回`false`并退出程序

~~~java
public class Checker {

    public static boolean checkSet(int set){
        if(set == 200){
            return true;
        }
        if(set == 501){
            System.out.println("Token设置失败，即将退出程序...");
        } else if(set == 502){
            System.out.println("生效期设置失败，或未在一分钟内完成操作，即将退出程序...");
        } else if(set == -1){
            System.out.println("请检查网络连接，即将退出程序...");
        }

        return false;
    }
}
~~~

Main 函数

- tip()：打印一些提示信息
- buffer()：线程休眠几秒，main函数中是一层while循环，listening函数中也是一层while循环，当listening收到指令退出循环后，main调用exec函数处理返回状态，这个时候要令外层循环等待一下，因为重置状态的操作在服务端执行，若直接继续执行，很有可能状态没有复原，重复执行上一条指令，这个地方设计有问题
- main()：用一层while套住listening，意在重复监听多条消息，而不是执行一次就退出程序，listenting是阻塞的，只有收到指令才会退出循环

~~~java
public class Main {

    private static void tip(){
        try{
            System.out.print("即将开始监听: 3 ");
            TimeUnit.SECONDS.sleep(1);
            System.out.print("2 ");
            TimeUnit.SECONDS.sleep(1);
            System.out.println("1...");
            TimeUnit.SECONDS.sleep(1);
            System.out.println("正在监听，请勿关闭程序，或造成消息丢失");
        }catch (InterruptedException e){
            e.printStackTrace();
        }
    }

    private static void buffer(){
        try{
            TimeUnit.SECONDS.sleep(12);
        }catch (InterruptedException e){
            e.printStackTrace();
        }
    }



    public static void main(String[] args) throws InterruptedException {
        Listener listener = new Listener();

        int set = listener.setToken();
        if(!Checker.checkSet(set)){ return; }

        // 打印提示信息
        tip();

        // 开始监听
        //noinspection InfiniteLoopStatement
        while(true){
            // listening是阻塞的，只有收到命令或Token失效才会继续执行
            String status = listener.listening();
            Executor.exec(status, listener);

            // 防止主程序继续循环，等一等服务端信息
            buffer();
        }
    }
}

~~~

### RedisConfig 配置

SpringBoot 集成 Redis

依赖

~~~xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
~~~

配置自定义`RedisTemplate`，定义序列化规则，注入`Bean`

RedisConfig.java

~~~java
@Configuration
public class RedisConfig {

    //一个固定的模板，在企业中可以直接使用，几乎包含了所有场景
    //编写我们自己的RedisTemplate
    @Bean
    @SuppressWarnings("all")
    public RedisTemplate<String, Object> myRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);

        //Json序列化配置
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper objectMapper = new ObjectMapper();
        objectMapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        //objectMapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        objectMapper.activateDefaultTyping(LaissezFaireSubTypeValidator.instance, ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(objectMapper);

        //String序列化配置
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        // key和Hash的key使用String序列化
        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);

        // value和Hash的value使用Jackson序列化
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
~~~

### Redis 工具类

注入自定义的`RedisTemplate`，封装`Redis`的一些基本功能，有待完善

`RedisUtil.java`

~~~java
// 待完善
@Component
@SuppressWarnings("all")
public class RedisUtil {

    private RedisTemplate myRedisTemplate;
    @Autowired
    public void setMyRedisTemplate(RedisTemplate myRedisTemplate){
        this.myRedisTemplate = myRedisTemplate;
    }


    //设置有效时间，单位秒
    public boolean expire(String key, long time){
        try{
            if(time > 0){
                myRedisTemplate.expire(key, time, TimeUnit.SECONDS);
            }
            return true;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    //获取剩余有效时间
    public long getExpire(String key){
        return myRedisTemplate.getExpire(key);
    }

    //判断键是否存在
    public boolean hasKey(String key){
        try{
            return myRedisTemplate.hasKey(key);
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    //批量删除键
    public void del(String... key){
        if(key != null && key.length > 0){
            if(key.length == 1){
                myRedisTemplate.delete(key[0]);
            } else {
                myRedisTemplate.delete(CollectionUtils.arrayToList(key));
            }
        }
    }

    //获取普通值
    public Object get(String key){
        return key == null ? null : myRedisTemplate.opsForValue().get(key);
    }

    //放入普通值
    public boolean set(String key, Object val){
        try{
            myRedisTemplate.opsForValue().set(key, val);
            return true;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }

    //放入普通缓存并设置时间
    public boolean set(String key, Object val, long time){
        try{
            if(time > 0){
                myRedisTemplate.opsForValue().set(key, val, time, TimeUnit.SECONDS);
            } else { // 若时间小于零直接调用普通设置的方法放入
                this.set(key, val);
            }
            return true;
        }catch (Exception e){
            e.printStackTrace();
            return false;
        }
    }


    //值增
    public long incr(String key, long delta){
        if(delta < 0){
            throw new RuntimeException("递增因子必须大于零");
        }
        return myRedisTemplate.opsForValue().increment(key, delta);
    }

    //值减
    public long decr(String key, long delta){
        if(delta < 0){
            throw new RuntimeException("递增因子必须大于零");
        }
        return myRedisTemplate.opsForValue().decrement(key, delta);
    }
~~~

### Service & Controller

在 Service 层注入`RedisUtil`，使用`Redis`的一些基本api实现用户功能

CommandService.java

~~~java
public interface CommandService {
    String shutdown(String token);
    String clean(String token);
    String ipconfig(String token);
}
~~~

`CommandServiceImpl.java`

1. 所有命令进来，首先要判断`Token`是否存在，若不存在，直接返回不存在或已失效信息(String)
2. 第二，若`Token`存在，需要判断其值，也就是状态是否为`alive`，若不为`alive`，说明有其他命令正在执行过程中，直接返回冲突信息，注意这里再加一层是否为空的判断，防止是`shutdown/clean`命令执行结束清空了`Token`
3. 两层判断后再进行业务处理，使用一个while循环去等待客户端反应，用一个很原始的计时器做简单的超时判断

```java
long before = System.currentTimeMillis();
while(true){
    long cur = System.currentTimeMillis();
    if(cur-before > 12000){
        return "超时";
    }
}
```

具体的通信设计

1. shutdown：网页端输入Token并通过判断后，设置`{Token: "shutdown"}`，开始等待客户端反应；（客户端每4s监听一次，会有适当延迟）客户端`get(token)`收到`shutdown`指令后，执行`shutdown`，再设置`{Token: "received"}`；服务端收到`received`，说明客户端已经执行了`shutdown`，删除`Token`，返回关机成功信息
2. clean：与shutdown类似，通过客户端发送的`received`判断是否接收到信息，收到才删除`Token`
3. ipconfig：服务端设置`{Token: "ipconfig"}`，等待相应；客户端收到`ipconfig`后查询ip地址，将ip作为值，即设置`{Token: IPv4}`，等待（buffer）服务端接收ip地址；当值不为`ipconfig`时说明客户端已经相应，接收`IPv4`地址并用`String`记录下来，重置`{Token: "alive}"`，继续下一轮监听

~~~java
@SuppressWarnings("all")
@Service
public class CommandServiceImpl implements CommandService {


    //注入RedisUtil，用@component修饰过
    private RedisUtil redisUtil;
    @Autowired
    public void setRedisUtil(RedisUtil redisUtil){
        this.redisUtil = redisUtil;
    }

    @Override
    public String shutdown(String token) {
        // 先判断Token是否存在
        if(!redisUtil.hasKey(token)){
            return "Token不存在或已失效";
        }

        // 先判断是否为alive，若不为alive说明有其他指令作用，防止冲突直接返回
        if(!redisUtil.hasKey(token) || !redisUtil.get(token).equals("alive")){
            return "其他命令执行中，请稍后再试";
        }

        // 设置命令，若发送失败返回错误信息
        if(!redisUtil.set(token, "shutdown")){
            return "消息发送失败，服务器错误";
        }
        //等待监听器收到命令并回应
        //简单设置一个计时器，当超过十二秒，处理异常并返回超时信息
        long before = System.currentTimeMillis();
        while(!redisUtil.get(token).equals("received")){
            long cur = System.currentTimeMillis();
            if(cur-before > 12000){
                redisUtil.del(token);
                return "长时间未收到监听器反馈，或监听器停止运行，已自动清除本次Token";
            }
        }
        redisUtil.del(token);
        return "成功接收消息，你的计算机将在10后关闭";
    }

    @Override
    public String clean(String token) {
        // 先判断Token是否存在
        if(!redisUtil.hasKey(token)){
            return "Token不存在或已失效";
        }

        // 先判断是否为alive，若不为alive说明有其他指令作用，防止冲突直接返回
        if(!redisUtil.hasKey(token) || !redisUtil.get(token).equals("alive")){
            return "其他命令执行中，请稍后再试";
        }

        if(!redisUtil.set(token, "clean")){
            return "消息发送失败，服务器错误";
        }

        //等待监听器收到命令并回应
        //简单设置一个计时器，当超过十二秒，处理异常并返回超时信息
        long before = System.currentTimeMillis();
        while(!redisUtil.get(token).equals("received")){
            long cur = System.currentTimeMillis();
            if(cur-before > 12000){
                redisUtil.del(token);
                return "长时间未收到监听器反馈，或监听器已停止运行，已清除本次Token";
            }
        }
        redisUtil.del(token);
        return "Token已清除，计算机上的Listener即将关闭";
    }

    @Override
    public String ipconfig(String token) {
        // 先判断Token是否存在
        if(!redisUtil.hasKey(token)){
            return "Token不存在或已失效";
        }

        // 先判断是否为alive，若不为alive说明有其他指令作用，防止冲突直接返回
        if(!redisUtil.hasKey(token) || !redisUtil.get(token).equals("alive")){
            return "其他命令执行中，请稍后再试";
        }

        // System.out.println(redisUtil.get(token));
        // 消息发送失败，返回错误信息
        if(!redisUtil.set(token, "ipconfig")){
            return "消息发送失败，服务器错误";
        }

        long before = System.currentTimeMillis();
        while(redisUtil.get(token).equals("ipconfig")){
            long cur = System.currentTimeMillis();
            if(cur-before > 12000){
                redisUtil.del(token);
                return "长时间未收到监听器反馈，或监听器停止运行，已清除本次Token";
            }
        }
        String ip = redisUtil.get(token).toString();
        redisUtil.set(token, "alive");
        return ip;
    }
}
~~~

Controller 层注入`CommandServiceImpl`，直接调用方法，返回对应字符串即可

~~~java
package com.northboat.remotecontrollerserver.controller;

import com.northboat.remotecontrollerserver.service.impl.CommandServiceImpl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Controller
@RequestMapping("/exec")
public class CommandController {

    private CommandServiceImpl commandService;
    @Autowired
    public void setCommandServiceImpl(CommandServiceImpl commandServiceImpl){
        this.commandService = commandServiceImpl;
    }


    @RequestMapping("/shutdown")
    public String shutdown(Model model, String token){
        String result = commandService.shutdown(token);
        System.out.println(token + "执行关机命令\t执行结果:" + result);
        model.addAttribute("result", result);
        return "index";
    }


    @RequestMapping("/ipconfig")
    public String ipConfig(Model model, String token){
        String result = commandService.ipconfig(token);
        System.out.println(token + "执行ipconfig命令\t执行结果:" + result);
        model.addAttribute("result", result);
        return "index";
    }

    @RequestMapping("/clean")
    public String clean(Model model, String token){
        String result = commandService.clean(token);
        System.out.println(token + "执行清除命令\t执行结果:" + result);
        model.addAttribute("result", result);
        return "index";
    }

}
~~~

### 前端及交互

> 前端套用的该网站模板，我只能说确实是好人
>
> [HTML5 UP! Responsive HTML5 and CSS3 Site Templates](https://html5up.net/)

单页面应用，交互很少

- 只有一个表单需要提交，即用户`Token`，用`thymeleaf`提交 post 请求到相应接口即可，如`th:action="@{/exec/shutdown}"`和`th:action="@{/exec/ipconfig}"`
- 只有一个信息需要展示，即命令执行结果`result`，通过`@Controller`返回到`"index"`即可

另外附上`listener.jar`的下载链接，jar 包放在`static`目录下即可

~~~html
<a href="/program/remote-controller-listener.jar" download="listener.jar">listener</a>
~~~

main.html

~~~html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">

	<head>
		<title>Remote-Controller-Ⅱ</title>
		<meta charset="utf-8"/>
		<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no"/>
		<link rel="stylesheet" href="/assets/css/main.css"/>
	</head>
	<body class="is-preload">

        <!-- Header -->
        <div id="header">

            <a href="#result"><span class="logo icon fa-paper-plane"></span></a>
            <h1>Hi. This is My Remote Controller.</h1>
            <p>
                一个适用于windows的远程控制程序 <br/>Help control your PC with a
                <a href="/program/remote-controller-listener.jar" download>listener</a>
            </p>

        </div>

        <!-- Main -->
        <div id="main">

            <header class="major container medium">
                <h2>下载并运行Listener后
                <br />
                在本页面使用Token
                <br />
                控制你的PC</h2>

                <p>using u own token to control u pc<br />
                after running the listener.</p>

            </header>

            <div class="box alt container">
                <section class="feature left">

                    <a href="#result" class="image icon solid fa-signal"><img src="/images/pic01.jpg" alt="查看ip地址" /></a>
                    <div class="content">
                        <h3>get ip config</h3>
                        <p>获取计算机当前的IPv4地址</p>
                        <form th:action="@{/exec/ipconfig}" method="post">
                            <input name="token" type="text" placeholder="输入你设置的Token"><br>
                            <input type="submit" value="查询">
                        </form>
                    </div>

                </section>
                <section class="feature right">
                    <a href="#result" class="image icon solid fa-code"><img src="/images/pic02.jpg" alt="查看关闭状态"/></a>
                    <div class="content">
                        <h3>shutdown the pc</h3>
                        <p>关闭你的计算机</p>
                        <form th:action="@{/exec/shutdown}" method="post">
                            <input name="token" type="text" placeholder="输入你设置的Token"><br>
                            <input type="submit" value="关机">
                        </form>
                    </div>
                </section>
                <section class="feature left">
                    <a href="#result" class="image icon solid fa-mobile-alt"><img src="/images/pic03.jpg" alt="查看清除状态" /></a>
                    <div class="content">
                        <h3>clean the token</h3>
                        <p>清除你的Token</p>
                        <form th:action="@{/exec/clean}" method="post">
                            <input name="token" type="text" placeholder="输入你设置的Token"><br>
                            <input type="submit" value="清除">
                        </form>
                    </div>
                </section>
            </div>

            <footer id="result" class="major container medium">
                <h3>命令返回结果</h3>
                <h4 th:text="${result}"></h4>
            </footer>

        </div>

        <!-- Footer -->
        <div id="footer">
            <div class="container medium">

                <header class="major last">
                    <h2>Based On Redis</h2>
                </header>

                <p>reo~reo~reo~reo~reo~reo~reo~reo~reo~reo~reo~reo~</p>

               

                <ul class="icons">
                    <li><a href="http://39.106.160.174:8082/Remote-Controller/" class="icon brands fa-dribbble"><span class="label">Instagram</span></a></li>
                    <li><a href="https://github.com/NorthBoat" class="icon brands fa-github"><span class="label">Github</span></a></li>
                    <li><a href="https://northboat.github.io/Blog/" class="icon brands fa-instagram"><span class="label">Dribbble</span></a></li>
                </ul>

                <ul class="copyright">
                    <li>&copy; NorthBoat.</li><li>Design: <a href="http://html5up.net">HTML5 UP</a></li>
                </ul>

            </div>
        </div>

        <!-- Scripts -->
        <script src="/assets/js/jquery.min.js"></script>
        <script src="/assets/js/browser.min.js"></script>
        <script src="/assets/js/breakpoints.min.js"></script>
        <script src="/assets/js/util.js"></script>
        <script src="/assets/js/main.js"></script>
	</body>
</html>
~~~

### 服务部署

1️⃣ 打包

1. 打包 jar：对于`listener`，`File - Project Structure - Artifacts`，点击`+`号选择`jar-from mudoles with dependencies `包，选择`main`函数入口点击`OK`，然后与`File`同级，点击`Build-build artifacts`即可
2. maven 打包：对于`Server`，使用`Maven`插件打包即可，若前后多次打包，要先`clean`再`package`
3. jar 转 exe：使用工具`exe4j`和`inno setup`将`listener.jar`转成一个直接可执行的`windows`安装包`listener setup.exe`，安装后无需`jdk`环境可直接运行`listener.exe`

2️⃣ 运行

~~~bash
nohup java -jar Remote-Controller-Ⅱ.jar --server.port=8085 > log/remoteController1.log &
nohup java -jar Remote-Controller-Ⅱ.jar --server.port=8086 > log/remoteController2.log &
~~~

3️⃣ nginx

~~~bash
cd /usr/local/nginx/conf
vim nginx.conf
~~~

设置负载均衡

~~~
# 配置负载均衡
upstream RemoteController{
	# 邮件发送服务器资源
	server 127.0.0.1:8085 weight=1;
	server 127.0.0.1:8086 weight=1; 
}
~~~

配置`location`

~~~
server {
	listen       8084;
	server_name  localhost;
	location / {
		root   html;
		index  index.html index.htm;
		# 配置服务
		proxy_pass http://RemoteController;
	}

	error_page   500 502 503 504  /50x.html;
	location = /50x.html {
		root   html;
	}
}
~~~

开放端口`8084`
