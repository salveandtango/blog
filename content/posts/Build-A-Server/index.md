---
title: 动手搭一台自己的服务器
date: 2021-11-10 10:58:00
categories: ["technical"]
---

## 前言
家里有一台大学时期的笔记本电脑，因为硬件有些小问题，一直放在柜子里吃灰。有一天突发奇想，既然我在阿里云买的服务器也不怎么用，这个笔记本也不怎么用，那为什么不把笔记本做成个人服务器呢，这样既能废物利用，也能省下买服务器的钱。于是在花了一个星期查询了各种资料、询问了身边的运维朋友、打爆了电信客服电话之后终于搭建成功，正常使用了两三天没发现什么问题，于是来给大伙分享一下搭建的过程。
## 公网IP
如果要搭建一台私人服务器，你最好有一个公网 IP ，这里的 IP 指的是 IPV4 。那么如何查询自己的 IP 地址是否是公网 IP 呢，你可以在浏览器中直接访问 [这个链接](https://ip.cn/)，如果它显示的 IP 地址与你在`ipconfig`中看到的一致，那么你的地址就已经是公网 IP 了。可惜目前国内的 IPV4 资源已经比较有限或者已经完全没有了，你获得的 IP 很可能是运营商通过多层 NAT 之后分配给你的内网 IP ，可以理解为运营商给每个片区的网络都套上了一个巨大的路由器，而你通过`ipconfig`获取的信息是这个路由器分配的。那么这就有一个问题，这个 IP 地址是别人无法 ping 通的，也就是说你无法把自己的电脑暴露在公网下，别人也无法正常访问你电脑上的服务。
这个时候有两种解决办法：

 1. 向运营商申请公网 IP
 2. 使用内网穿透技术

我是用的是第一种办法——打电话向运营商申请公网 IP 。我是电信客户，在直接致电电信人工客服说明情况之后，他们同意了我的申请，给我分配一个公网 IP，只是这个 IP 地址不是固定的。

而如果你购买了域名的话，那么就需要在 IP 地址改变的时候重新进行域名解析。这看上去是很麻烦的一件事，其实可以通过DDNS解决。

## DDNS

> DDNS（Dynamic Domain Name Server，动态域名服务）是将用户的[动态IP地址](https://baike.baidu.com/item/动态IP地址)映射到一个固定的[域名解析](https://baike.baidu.com/item/域名解析/574285)服务上，用户每次连接网络的时候客户端程序就会通过信息传递把该[主机](https://baike.baidu.com/item/主机/455151)的[动态IP地址](https://baike.baidu.com/item/动态IP地址/10688515)传送给位于服务商主机上的[服务器](https://baike.baidu.com/item/服务器)程序，服务器程序负责提供DNS服务并实现[动态域名解析](https://baike.baidu.com/item/动态域名解析/98200)。
>
> ——百度百科

我是用的是阿里云的域名服务，在阿里云的 [API 文档](https://help.aliyun.com/document_detail/29774.html)中我们可以找到一项修改解析记录的功能，当然要使用阿里的 SDK 并且申请一个 Access Key 。具体的实现方法可以参考 @duapple 老哥的 [博客](https://blog.csdn.net/duapple/article/details/109481236)。他使用的是 GO 语言进行 DDNS 操作，我也贴出自己的 Java 版本供大家参考 。

``` java
import com.aliyun.credentials.utils.StringUtils;
import com.aliyuncs.DefaultAcsClient;
import com.aliyuncs.alidns.model.v20150109.DescribeSubDomainRecordsRequest;
import com.aliyuncs.alidns.model.v20150109.DescribeSubDomainRecordsResponse;
import com.aliyuncs.alidns.model.v20150109.UpdateDomainRecordRequest;
import com.aliyuncs.exceptions.ClientException;
import com.aliyuncs.profile.DefaultProfile;
import com.aliyuncs.profile.IClientProfile;
import com.google.gson.Gson;
import lombok.Data;
import org.apache.http.HttpEntity;
import org.apache.http.ParseException;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.util.EntityUtils;

import java.io.IOException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;


/**
 * @author unboiledwater
 */
public class Application {
	
    // 这里你可以选择自己的服务器地址，我的是深圳
    static String regionId = "cn-shenzhen";
    static String accessKeyId = "xxx";
    static String accessKeySecret = "xxx";

    @Data
    static class IPResponse {
        private Integer rs;
        private Integer code;
        private String ip;
        private Integer isDomain;
    }

    public static String getV4Ip() {
        String ip = null;
        String chinaz = "https://ip.cn/api/index?ip=&type=0";
        CloseableHttpClient httpClient = HttpClientBuilder.create().build();
        HttpGet httpGet = new HttpGet(chinaz);
        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpGet);
            HttpEntity responseEntity = response.getEntity();
            if (responseEntity != null) {
                String content = EntityUtils.toString(responseEntity);
                Gson gson = new Gson();
                IPResponse j = gson.fromJson(content, IPResponse.class);
                ip = j.getIp();
            }
        } catch (ParseException | IOException e) {
            System.out.println("连接失败" + e.getMessage());
        } finally {
            try {
                if (httpClient != null) {
                    httpClient.close();
                }
                if (response != null) {
                    response.close();
                }
            } catch (IOException e) {
                System.out.println("IOException" + e.getMessage());
                e.printStackTrace();
            }
        }
        if (ip != null && StringUtils.isEmpty(ip)) {
            throw new RuntimeException("未找到IP");
        } else {
            return ip;
        }
    }

    public static void main(String[] args) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        IClientProfile profile = DefaultProfile.getProfile(regionId, accessKeyId, accessKeySecret);
        DefaultAcsClient client = new DefaultAcsClient(profile);
        DescribeSubDomainRecordsRequest recordsRequest = new DescribeSubDomainRecordsRequest();
        recordsRequest.setSubDomain("你的域名或二级域名");
        DescribeSubDomainRecordsResponse recordsResponse;
        try {
            recordsResponse = client.getAcsResponse(recordsRequest);
            List<DescribeSubDomainRecordsResponse.Record> recordList = recordsResponse.getDomainRecords();
            String openIp = getV4Ip();
            for (DescribeSubDomainRecordsResponse.Record record : recordList) {
                String oldIp = record.getValue();
                if (!oldIp.equals(openIp)) {
                    System.out.println(LocalDateTime.now().format(formatter) + " ---- 公网IP变化，准备进行DDNS");
                    System.out.println("ip>>>" + oldIp
                            + "  DomainName>>>" + record.getDomainName()
                            + "  Type>>>" + record.getType()
                            + "  RR>>>" + record.getRR()
                            + "  Priority>>>" + record.getPriority());
                    UpdateDomainRecordRequest udrReq = new UpdateDomainRecordRequest();
                    udrReq.setRecordId(record.getRecordId());
                    udrReq.setRR(record.getRR());
                    udrReq.setValue(openIp);
                    udrReq.setType(record.getType());
                    udrReq.setTTL(record.getTTL());
                    udrReq.setPriority(record.getPriority());
                    udrReq.setLine(record.getLine());
                    client.getAcsResponse(udrReq);
                    System.out.println(LocalDateTime.now().format(formatter) + " ---- 重新解析域名完成");
                } else {
                    System.out.println(LocalDateTime.now().format(formatter) + " ---- 公网IP未变化");
                }
            }
        } catch (ClientException e) {
            System.out.println("============ 阿里SDK报错 ============");
            e.printStackTrace();
        }
    }
}
```

pom 文件如下：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>ddns-java</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <repositories>
        <repository>
            <id>sonatype-nexus-staging</id>
            <name>Sonatype Nexus Staging</name>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
    <dependencies>
        <!-- https://mvnrepository.com/artifact/com.aliyun/aliyun-java-sdk-alidns -->
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>aliyun-java-sdk-alidns</artifactId>
            <version>2.6.32</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/com.aliyun/aliyun-java-sdk-core-v5 -->
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>aliyun-java-sdk-core</artifactId>
            <version>4.5.29</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/com.google.code.gson/gson -->
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.8.9</version>
        </dependency>

        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>alidns20150109</artifactId>
            <version>2.0.1</version>
        </dependency>
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>tea-openapi</artifactId>
            <version>0.1.1</version>
        </dependency>
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>tea-console</artifactId>
            <version>0.0.1</version>
        </dependency>
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>tea-util</artifactId>
            <version>0.2.13</version>
        </dependency>
        <dependency>
            <groupId>com.aliyun</groupId>
            <artifactId>tea</artifactId>
            <version>[1.1.13, 2.0.0)</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.5</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.22</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>
    <build>
        <finalName>ddns-java</finalName>
        <plugins>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <configuration>
                <appendAssemblyId>false</appendAssemblyId>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
                <archive>
                    <manifest>
                        <!-- 此处指定main方法入口的class -->
                        <mainClass>Application</mainClass>
                    </manifest>
                </archive>
              </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>assembly</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        </plugins>
    </build>
</project>
```

执行`maven install`命令之后，会在项目的`target`文件夹下生成`ddns-java`的可执行 jar 包。将这个包上传到服务器后，在当前文件夹下执行`java -jar ddns.java`命令就可以自动检测 IP 地址是否变换并修改 DNS 解析记录。之后就是编写一个定时任务循环执行该 jar 包就可以了。我这里使用的是 [crontab](https://www.runoob.com/linux/linux-comm-crontab.html) ，你也可以在代码中编写定时任务。

## DMZ或端口映射

> DMZ是英文“demilitarized zone”的缩写，中文名称为“隔离区”，也称“非军事化区”。它是为了解决安装[防火墙](https://baike.baidu.com/item/防火墙/52767)后外部网络不能访问内部[网络服务器](https://baike.baidu.com/item/网络服务器/99096)的问题，而设立的一个非[安全系统](https://baike.baidu.com/item/安全系统/3131501)与安全系统之间的[缓冲区](https://baike.baidu.com/item/缓冲区/4830555)，这个缓冲区位于企业内部网络和外部网络之间的小网络区域内，在这个小网络区域内可以放置一些必须公开的服务器设施，如企业[Web服务器](https://baike.baidu.com/item/Web服务器/8390210)、[FTP服务器](https://baike.baidu.com/item/FTP服务器)和论坛等。另一方面，通过这样一个DMZ区域，更加有效地保护了内部网络，因为这种网络部署，比起一般的防火墙方案，对攻击者来说又多了一道关卡。
>
> 端口映射是NAT的一种，功能是把在公网的地址转翻译成[私有地址](https://baike.baidu.com/item/私有地址/727338)， 采用[路由方式](https://baike.baidu.com/item/路由方式/22306021)的ADSL宽带路由器拥有一个动态或固定的[公网IP](https://baike.baidu.com/item/公网IP/8881123)，[ADSL](https://baike.baidu.com/item/ADSL/96520)直接接在[HUB](https://baike.baidu.com/item/HUB/703984)或[交换机](https://baike.baidu.com/item/交换机/103532)上，所有的电脑[共享上网](https://baike.baidu.com/item/共享上网/933900) 。
>
> ——百度百科

配置好 DDNS 之后，还需要设置 DMZ 主机或者配置端口映射才能真正访问到电脑，这些需要到光猫的网关中去配置。拿电信举例，电信的天翼网关必须有超级管理员权限才可以进行 DMZ 主机和端口映射的配置，而这个超级管理员的密码我是没有的，联系客服之后才拿到，进入光猫网关的超级管理员界面后，在【应用】一栏可以找到【高级NAT配置】，在这里就可以设置自己的 DMZ 主机和端口映射了。

简单来说，将电脑设置为 DMZ 主机之后，这台主机是**完全暴露在公网下的**，所有的开放端口都可以从公网链接，比较危险，使用时一定要开启系统防火墙。而端口映射则是只向公网暴露你设置好的端口号，风险降低了，不过需要多配置一下。

注意：如果电脑接的路由器的话，不建议直接将路由器配置成 DMZ 主机，这样连接这台路由器的**所有设备很可能都将暴露在公网下**，我的做法是将电脑直接和光猫连接，开启防火墙并手动修改应用端口，这样也能保证一定的安全性。如果你不得不连接路由器的话，那可能就需要做两层端口转发才能生效。（我还没有尝试过，不过应该是这样的）

## 其他问题

到这里服务器基本已经可以正常使用了，总的看下来整个配置过程其实并不复杂，来看看别的。

### CentOs 8 装机

因为习惯使用 CentOs 而且笔记本也带的动，所以在笔记本安装了 CentOs 8 系统。简单分享一下装机过程：

1. 搞一个 USB 3.1接口的 U 盘，真的很必要！不然 9 个多 G 的系统要花大概一个半小时的时间才能拷贝到 U 盘里。
2. 在[这里](https://www.centos.org/download/)下载好 CentOs 8 的 ISO 文件。
3. 在[这里](http://rufus.ie/zh/)下载好 Rufus 装机软件。
4. 打开 Rufus，将【设备】一栏选中自己的 U 盘，【引导类型选择】一栏选中下载好的 ISO 文件，点击【开始】按钮，按照提示操作。
5. 给电脑插上 U 盘，重启进入 BIOS，选择 U 盘启动，再次开机就可以开始安装系统了。

详细过程参考[这里](https://www.jianshu.com/p/236554fe5ab7)。

### 修改 SSH 的 22 端口

我用的是 DMZ 方式，所以有必要将开放的端口都重新自定义一下避免不必要的麻烦。结果发现直接修改配置文件好像 sshd 服务直接不工作了，查了一下好像是需要用到`semanage`口令。

1. 在`/etc/ssh/sshd_config`配置文件中新增一个端口，例如`Port 11`
2. 输入`semanage port -a -t ssh_port_t -p tcp 11`新增 SSH 端口
3. 测试（可能需要重启 sshd 服务）
4. 成功之后输入`semanage port -d -t ssh_port_t -p tcp 22`删除原本的 22 端口
5. 注释`/etc/ssh/sshd_config`配置文件中的 22 端口

### 80端口被禁

电信普通线路的 80 端口是被封禁的，在咨询所在片区客服经理时得到的回复是：只有城域网线路才可以开通 80 和 8080 端口。那就说明除非我在域名后加上自定义端口，否则我无法开启 Web 服务 :sweat_smile:。在查询了一些资料以后，发现这其实也是有一个不是办法的解决办法：配置隐性 URL 转发。

我购买的是阿里云的域名，在 DNS 解析服务中，可以添加一条记录类型为【隐性 URL】的解析，记录值则是你需要转发到的地址 + 端口号。举个例子——

假如

- 当前有一个域名`a.xx`，自定义的Web 服务端口是 8181，浏览器访问 Web 服务需要在网址栏中输入`a.xx:8181`

那么

- 新建解析的记录类型是`隐性URL`，主机记录是`b.xx`，记录值为`http://a.xx:8181`，这样你在访问`b.xx`域名时浏览器返回的其实是`a.xx:8181`的结果

遗憾的是在这个隐性 URL 转发下无法实现 HTTPS 访问，网址栏左边的警示标志还是很不好看的。
