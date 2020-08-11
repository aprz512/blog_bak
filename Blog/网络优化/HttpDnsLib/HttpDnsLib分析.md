---
title: HttpDnsLib分析
date: 2020-6-30
categories: 网络优化
---

框架地址：

https://github.com/CNSRE/HTTPDNSLib

先从，框架的几个模块开始介绍。

### 测速模块

对于一个大型app来说，在全国会有多个服务器， 每个服务器的ip地址肯定也是不一致的，所以对于不同地区的用户，一般情况下与之最近的服务器肯定访问最快，测速模块就是找到访问最快的服务器。

测速有两种方法，第一种是使用 ping 命令，第二种是使用 socket。下面分别介绍：

#### Ping

```java
public static class Ping {
    // ping -c1 -s1 -w1 www.baidu.com //-w 超时单位是s
    private static final String TAG_BYTES_FROM = "bytes from ";

    public static int runcmd(String cmd) throws Exception {
        Runtime runtime = Runtime.getRuntime();
        Process proc = null;

        final String command = cmd.trim();
        // 这里计算的是命令跑完的时间，而不是ping输出的时间
        long startTime = System.currentTimeMillis();
        proc = runtime.exec(command);
        proc.waitFor();
        long endTime = System.currentTimeMillis();
        InputStream inputStream = proc.getInputStream();
        String result = "unknown ip";

        BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
        StringBuilder resultBuilder = new StringBuilder();
        String line = "";
        while (null != (line = reader.readLine())) {
            resultBuilder.append(line);
        }
        reader.close();
        String responseStr = resultBuilder.toString();
        result = responseStr.toLowerCase().trim();
        if (isValidResult(result)) {
            return (int) (endTime - startTime);
        }
        return SpeedtestManager.OCUR_ERROR;
    }

    private static boolean isValidResult(String result) {
        if (!TextUtils.isEmpty(result)) {
            if (result.indexOf(TAG_BYTES_FROM) > 0) {
                return true;
            }
        }
        return false;
    }
}
```

这段代码其实就是使用 Runtime 执行了一个 ping 命令，但是它测的时间是这个命令同步执行耗费的时间，而不是命令返回的结果里面的时间。

#### Socket

```java
public int speedTest(String ip, String host) {
    Socket socket = null;
    try {
        // 计算的是 socket 的 connect 连接时间
        long begin = System.currentTimeMillis();
        Socket s1 = new Socket();
        s1.connect(new InetSocketAddress(ip, 80), TIMEOUT);
        long end = System.currentTimeMillis();
        int rtt = (int) (end - begin);
        return rtt;
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            if (null != socket) {
                socket.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return SpeedtestManager.OCUR_ERROR;
}
```

使用的 Socket 的 connect 方法来测量访问速度。

在进行测速的时候，会优先使用 Socket 来测速，如果 Socket 测通了，那么就不会再使用 Ping 来测速了。



### 评分模块

框架里面虽然缓存了各个域名的IP地址，但是有效期只有1分钟，所以开了一个任务来不断的更新过期的域名IP。

更新了IP之后，也会对新IP进行测速，在测速后会统计该IP测试通过-- 成功的次数，失败的次数，最后成功的时间，以及RTT值，另外还有一个HttpDns服务器返回IP的时候会带一个优先级，这个也计入评分项。

> RTT 的概念可以自己搜索一下。

#### 成功的次数

使用上面的测速模块对IP进行测速之后，我们记录一下测速状态：

```java
for (IpModel ipModel : ipArray) {
    int rtt = speedtestManager.speedTest(ipModel.ip, domainModel.domain);
    boolean succ = rtt > SpeedtestManager.OCUR_ERROR;
    if (succ) {
        ipModel.rtt = String.valueOf(rtt);
        ipModel.success_num = String.valueOf((Integer.parseInt(ipModel.success_num) + 1));
        ipModel.finally_success_time = String.valueOf(System.currentTimeMillis());
    } else {
        ipModel.rtt = String.valueOf(SpeedtestManager.MAX_OVERTIME_RTT);
        ipModel.err_num = String.valueOf((Integer.valueOf(ipModel.err_num) + 1));
        ipModel.finally_fail_time = String.valueOf(System.currentTimeMillis());
    }
}
```

代码逻辑很简单：

如果成功了，记录RTT值，成功次数加一，记录成功时间。

如果失败了，RTT记为9999，失败次数加一，记录失败时间。

我们看一下成功次数这一项对评分的影响：

```java
public void run(ArrayList<IpModel> list) {
    // 查找到最大历史成功次数
    float MAX_SUCCESSNUM = 0;
    for (IpModel temp : list) {
        if (temp.success_num == null || temp.success_num.equals(""))
            continue;
        float successNum = Float.parseFloat(temp.success_num);
        MAX_SUCCESSNUM = Math.max(MAX_SUCCESSNUM, successNum);
    }
    // 计算比值
    if (MAX_SUCCESSNUM == 0) {
        return;
    }
    float bi = getWeight() / MAX_SUCCESSNUM;
    // 计算得分
    for (IpModel temp : list) {
        if (temp.success_num == null || temp.success_num.equals("")){
            continue;
        }
        float successNum = Float.parseFloat(temp.success_num);
        temp.grade += (successNum * bi);
    }

}


public float getWeight() {
    return PlugInManager.SuccessNumPluginNum;
}

```

逻辑不负责，使用的是比较简单的评分算法，找到所有IP的最大成功次数，然后按照自己的成功次数计算出一个比例，在诚意一个固定的值。

这种算法最神奇的地方就是参数了，没人知道为什么是这个参数，但是它就是可以工作。所以调参是个技术活。

其他项的评分算法其实是一样的逻辑，就不过多的介绍了，再拿RTT举个例子吧。

#### RTT

```java
@Override
public void run(ArrayList<IpModel> list) {
    // 查找到最大速度
    float MAX_SPEED = 0;
    for (IpModel temp : list) {
        if (temp.rtt == null || temp.rtt.equals(""))
            continue;
        float finallySpeed = Float.parseFloat(temp.rtt);
        MAX_SPEED = Math.max(MAX_SPEED, finallySpeed);
    }
    // 计算比值
    if (MAX_SPEED == 0) {
        return;
    }
    float bi = getWeight() / MAX_SPEED;
    // 计算得分
    for (IpModel temp : list) {
        if (temp.rtt == null || temp.rtt.equals("")){
            continue;
        }
        float finallySpeed = Float.parseFloat(temp.rtt);
        temp.grade += (getWeight() - (finallySpeed * bi));
    }
}

@Override
public float getWeight() {
    return PlugInManager.SpeedTestPluginNum;
}
```

唯一不同点就是，RTT越小，评分应该越高，所以采用的是减法。

### DNS服务器模块

我一直以为HttpDns服务器要一个就够用了，没想到真是开了眼界了。这个库里面有4个DNS服务器，其中有一个是本地服务器。

#### PodDns

这个是一个免费的HttpDns。

```java
public HttpDnsPack requestDns(String domain) {
    // 如果新浪自家的服务器没有拿到数据，或者数据有问题，则使用 dnspod 提供的接口获取数据
    String jsonDataStr = null;
    HttpDnsPack dnsPack = null;

    String dnspod_httpdns_api_url = DnsConfig.DNSPOD_SERVER_API + DNSPodCipher.Encryption(domain);
    jsonDataStr = netWork.requests(dnspod_httpdns_api_url);
    if (jsonDataStr == null || jsonDataStr.equals(""))
        return null; // 如果dnspod 也没提取到数据 则返回空

    jsonDataStr = DNSPodCipher.Decryption(jsonDataStr);

    dnsPack = new HttpDnsPack();
    try {
        String IP_TTL[] = jsonDataStr.split(",");
        String IPArr[] = IP_TTL[0].split(";");
        String TTL = IP_TTL[1];
        dnsPack.rawResult = jsonDataStr;
        dnsPack.domain = domain;
        dnsPack.device_ip = NetworkManager.Util.getLocalIpAddress();
        dnsPack.device_sp = NetworkManager.getInstance().getSPID() ; 

        dnsPack.dns = new HttpDnsPack.IP[IPArr.length];
        for (int i = 0; i < IPArr.length; i++) {
            dnsPack.dns[i] = new HttpDnsPack.IP();
            dnsPack.dns[i].ip = IPArr[i];
            dnsPack.dns[i].ttl = TTL;
            dnsPack.dns[i].priority = "0";
        }
    } catch (Exception e) {
        dnsPack = null;
    }
    return dnsPack;
}
```

这个就没啥好说的了，就是按照文档里面的来就好了。值得一提的是，因为这个免费的返回数据里面并没有提供Model里面的各个字段（比自身的HttpDns服务器返回的字段少了一些，所以有些字段需要自己拼一下）。

比如，设备的ip与运营商。关于运营商还有个小知识点需要说一下：

> 在客户端内如果是手机网络可以知道网络类型（2G、3G、4G）也可以知道当前SP（移动、联通、电信）.
>
> 如果是Wifi网络环境可以知道SSID，但是无法知道当前SP.

#### LocalDns

这个是本地的Dns，使用的是 InetAddress 来获取对应的域名 IP。

```java
public HttpDnsPack requestDns(String domain) {
    try {
        InetAddress[] addresses = InetAddress.getAllByName(domain);
        String[] ipList = new String[addresses.length];
        for (int i = 0; i < addresses.length; i++) {
            ipList[i] = addresses[i].getHostAddress();
        }
        if (null != ipList && ipList.length > 0) {
            HttpDnsPack dnsPack = new HttpDnsPack();
            String IPArr[] = ipList;
            String TTL = "60";
            dnsPack.domain = domain;
            dnsPack.device_ip = NetworkManager.Util.getLocalIpAddress();
            dnsPack.device_sp = NetworkManager.getInstance().getSPID() ;
            dnsPack.rawResult = "domain:" + domain + ";\nipArray:";
            dnsPack.dns = new HttpDnsPack.IP[IPArr.length];
            for (int i = 0; i < IPArr.length; i++) {
                String ip = IPArr[i];
                if (i == IPArr.length - 1) {
                    //去掉最后的逗号
                    dnsPack.rawResult += (ip);
                } else {
                    dnsPack.rawResult += (ip + ",");
                }
                dnsPack.dns[i] = new HttpDnsPack.IP();
                dnsPack.dns[i].ip = ip;
                dnsPack.dns[i].ttl = TTL;
                dnsPack.dns[i].priority = "0";
            }
            return dnsPack;
        }
    } catch (Exception e) {
        e.printStackTrace();
    }

    return null;
}
```

这个也很简单，就是利用一些信息，在本地自己拼出来一个HttpDnsPack对象。

#### SinaHttpDns

```java
public HttpDnsPack requestDns(String domain) {
    String jsonDataStr = null;
    HttpDnsPack dnsPack = null;
    ArrayList<String> serverApis = new ArrayList<String>();
    serverApis.addAll(DnsConfig.SINA_HTTPDNS_SERVER_API);
    while (null == dnsPack && serverApis.size() > 0) {
        try {
            String api = "";
            int index = serverApis.indexOf(usingServerApi);
            if (index != -1) {
                api = serverApis.remove(index);
            } else {
                api = serverApis.remove(0);
            }
            String sina_httpdns_api_url = api + domain;
            jsonDataStr = netWork.requests(sina_httpdns_api_url);
            dnsPack = jsonObj.JsonStrToObj(jsonDataStr);
            usingServerApi = api;
        } catch (Exception e) {
            e.printStackTrace();
            usingServerApi = "";
        }
    }
    return dnsPack;
}
```

自身的服务器，返回的数据直接使用 Json 转对象了，没啥好说的。

里面的逻辑有做一个循环，是因为它的请求地址有多个，一个不行再试另外一个。

#### UdpDns

这个应该是访问的公共的DNS服务器。

```java
public HttpDnsPack requestDns(String domain) {
    try {
        UdpDnsInfo info = UdnDnsClient.query(DnsConfig.UDPDNS_SERVER_API, domain);
        if (null != info && info.ips.length > 0) {
            HttpDnsPack dnsPack = new HttpDnsPack();
            String IPArr[] = info.ips;
            String TTL = String.valueOf(info.ttl);
            dnsPack.rawResult = "domain : " + domain + "\n" + info.toString();
            dnsPack.domain = domain;
            dnsPack.device_ip = NetworkManager.Util.getLocalIpAddress();
            dnsPack.device_sp = NetworkManager.getInstance().getSPID();

            dnsPack.dns = new HttpDnsPack.IP[IPArr.length];
            for (int i = 0; i < IPArr.length; i++) {
                dnsPack.dns[i] = new HttpDnsPack.IP();
                dnsPack.dns[i].ip = IPArr[i];
                dnsPack.dns[i].ttl = TTL;
                dnsPack.dns[i].priority = "0";
            }
            return dnsPack;
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
    return null;
}
```

一共有4个服务器，他们也是有优先级的，与测速一样，优先级高的跑通了，后面的就不再跑了。

我们总结一下上面的3个模块：

DNS模块是获取域名对应的IP地址

测速模块是对IP地址进行测速

评分模块是对IP地址的访问状况做评分，选出最好的

基本上，经过这3个模块后，我们就可以拿到表现最好的IP地址了。但是这还不够，接下来还有几个模块需要介绍。

### 缓存模块

我们从 HttpDns 服务器里面请求下来了那么多数据，而且我们还设置了IP的有效期为一分钟，会不断的更新这些数据，所以将这些数据进行缓存是非常有必要的。

缓存分为两部分，内存与数据库。

内存就比较简单，就是一个ConcurrentHashMap。

```java
/**
 * 缓存链表，域名与 DomainModel 的键值对。
 */
private ConcurrentHashMap<String, DomainModel> data = new ConcurrentHashMap<String, DomainModel>(INIT_SIZE, MAX_CACHE_SIZE);
```

数据库是分为了三个表：

```
		db.execSQL(CREATE_DOMAIN_TABLE_SQL);
		db.execSQL(CREATE_IP_TEBLE_SQL);
		db.execSQL(CREATE_CONNECT_FAIL_TABLE_SQL);
```

DOMAIN表 是域名。

IP表 是域名对应的IP集合。

DOMAIN 与 IP 是关联的。

CONNECT_FAIL表 是用于上报，这里不分析。

我们先来看一下数据模型：

```java
public class DomainModel {
	
	/**
	 * 自增id <br>
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants } DOMAIN_COLUMN_ID 字段 <br>
	 */
	public long id = -1 ;
	
	/**
	 * 域名 <br>
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants } DOMAIN_COLUMN_DOMAIN 字段 <br>
	 */
	public String domain = "" ; 
	
	/**
	 * 运营商 <br>
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants } DOMAIN_COLUMN_SP 字段 <br>
	 */
	public String sp = "" ; 
	
	/**
	 * 域名过期时间 <br>
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants } DOMAIN_COLUMN_TTL 字段 <br>
	 */
	public String ttl = "0" ;

    /**
     * 域名最后查询时间 <br>
     *
     * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants } DOMAIN_COLUMN_TIME 字段 <br>
     */
    public String time = "0" ;




    /**
	 * 域名关联的ip数组 <br>
	 */
	public ArrayList<IpModel> ipModelArr = null ; 

}

public class IpModel {
	
	
	/**
	 * 自增id <br>
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_ID }字段 <br>
	 */
	public long id = -1 ; 
	
	/**
	 * domain id 关联id
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_DOMAIN_ID }字段 <br>
	 */
	public long d_id = -1 ; 
	
	/**
	 * 服务器ip地址
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_PORT }字段 <br>
	 */
	public String ip = "" ;
	
	/**
	 * ip服务器对应的端口
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_PORT }字段 <br>
	 */
	public int port = -1 ; 
	
	/**
	 * ip服务器对应的sp运营商
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_SP }字段 <br>
	 */
	public String sp = "" ; 
	
	/**
	 * ip过期时间
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_TTL }字段 <br>
	 */
	public String ttl = "0" ; 
	
	/**
	 * ip服务器优先级-排序算法策略使用
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_PRIORITY }字段 <br>
	 */
	public String priority = "0" ; 
	/**
	 * 访问ip服务器的往返时延
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_PRIORITY }}字段 <br>
	 */
	public String rtt = "0" ; 
	
	/**
	 * ip服务器链接产生的成功数
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_SUCCESS_NUM }字段 <br>
	 */
	public String success_num = "0" ; 
	
	/**
	 * ip服务器链接产生的错误数
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_ERR_NUM }字段 <br>
	 */
	public String err_num = "0" ; 
	
	/**
	 * ip服务器最后成功链接时间
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_FINALLY_SUCCESS_TIME }字段 <br>
	 */
	public String finally_success_time = "0" ; 
	
	/**
	 * ip服务器最后失败链接时间
	 * 
	 * 该字段映射类 {@link com.sina.util.dnscache.cache.DBConstants#IP_COLUMN_FINALLY_FAIL_TIME }字段 <br>
	 */
	public String finally_fail_time = "0" ; 


	
	/**
	 * 评估体系 评分分值
	 */
	public float grade = 0 ; 
}
```

可以看出。数据库就是储存了这些对象的信息。

我们看一下缓存对外暴露的接口：

```java
    /**
     * 获取 domain 缓存
     *
     * @param sp
     * @param domain
     * @return
     */
    public DomainModel getDnsCache(String sp, String domain);


    /**
     * 插入一条缓存记录
     *
     * @param dnsPack
     * @return
     */
    public DomainModel insertDnsCache(HttpDnsPack dnsPack);

    /**
     * 获取缓存中全部的 DomainModel数据
     *
     * @return
     */
    public ArrayList<DomainModel> getAllMemoryCache();
```

这里我只列出了3个，还有很多，但是这3个可以说明缓存的大致作用，就是将服务器请求的数据缓存起来，然后用时可以查找取出。

#### 定时器

定时器是属于缓存模块的，但是我个人是觉得很重要的，因为只有开了定时器，才会不断的去 HttpDns 服务器中去同步IP地址。

```java
    private TimerTask task = new TimerTask() {

        @Override
        public void run() {
            timerTaskOldRunTime = System.currentTimeMillis();
            //无网络情况下不执行任何后台任务操作
            if (NetworkManager.Util.getNetworkType() == Constants.NETWORK_TYPE_UNCONNECTED || NetworkManager.Util.getNetworkType() == Constants.MOBILE_UNKNOWN) {
                return;
            }
            /************************* 更新过期数据 ********************************/
            Thread.currentThread().setName("HTTP DNS TimerTask");
            final ArrayList<DomainModel> list = dnsCacheManager.getExpireDnsCache();
            for (DomainModel model : list) {
                checkUpdates(model.domain, false);
            }

            long now = System.currentTimeMillis();
            /************************* 测速逻辑 ********************************/
            if (now - lastSpeedTime > SpeedtestManager.time_interval - 3) {
                lastSpeedTime = now;
                RealTimeThreadPool.getInstance().execute(new SpeedTestTask());
            }

            /************************* 日志上报相关 ********************************/
            now = System.currentTimeMillis();
            if (HttpDnsLogManager.LOG_UPLOAD_SWITCH && now - lastLogTime > HttpDnsLogManager.time_interval) {
                lastLogTime = now;
                // 判断当前是wifi网络才能上传
                if (NetworkManager.Util.getNetworkType() == Constants.NETWORK_TYPE_WIFI) {
                    RealTimeThreadPool.getInstance().execute(new LogUpLoadTask());
                }
            }
        }
    };
```

这个任务逻辑很清晰：

先更新缓存中过期的数据，然后对缓存中所有的IP，重新进行测速。

这样，一个完整的库就差不多写好了。