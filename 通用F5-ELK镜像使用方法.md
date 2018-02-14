# 通用F5-ELK镜像使用方法

## 镜像下载地址

存储地址可能发生变化，因此如需要请email我，说明需通用版ELK镜像即可。

## 镜像基本信息

1. 镜像基础为Centos7，用户名密码root/root

2. 默认配置了三个网卡，多个网卡主要是为了适应本地测试，在实际使用时，可以只使用一个网卡

3. 镜像默认包含三个硬盘，一个系统盘，两个ELK数据盘，全部为lvm，方便扩展。系统盘默认20G，两个数据盘分别为50G。两个数据盘分别挂载到了/elkbigdata以及/elkbigdata2目录

   ```
   [root@F5ELK-NODE01 ~]# df -h
   Filesystem                         Size  Used Avail Use% Mounted on
   /dev/mapper/centos-root             18G  2.7G   15G  15% /
   devtmpfs                           2.9G     0  2.9G   0% /dev
   tmpfs                              3.0G     0  3.0G   0% /dev/shm
   tmpfs                              3.0G  8.7M  2.9G   1% /run
   tmpfs                              3.0G     0  3.0G   0% /sys/fs/cgroup
   /dev/mapper/bigdatavg2-bigdatalv2   50G   33M   50G   1% /elkbigdata2
   /dev/mapper/bigdatavg1-bigdatalv1   50G   33M   50G   1% /elkbigdata
   /dev/sda1                          497M  167M  331M  34% /boot
   tmpfs                              596M     0  596M   0% /run/user/0
   ```

   ```
   [root@F5ELK-NODE01 ~]# lvdisplay 
     --- Logical volume ---
     LV Path                /dev/bigdatavg2/bigdatalv2
     LV Name                bigdatalv2
     VG Name                bigdatavg2
     LV UUID                gSSWHq-df57-RpM5-ox8S-PfIu-kNmp-x9ZrCM
     LV Write Access        read/write
     LV Creation host, time F5ELK-NODE01.myf5.net, 2018-02-14 13:03:58 +0800
     LV Status              available
     # open                 1
     LV Size                <50.00 GiB
     Current LE             12799
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:3
      
     --- Logical volume ---
     LV Path                /dev/centos/swap
     LV Name                swap
     VG Name                centos
     LV UUID                LH7KSi-o87B-aMDz-2GBW-T53R-73Dk-6QDZPg
     LV Write Access        read/write
     LV Creation host, time localhost.localdomain, 2016-07-26 12:00:02 +0800
     LV Status              available
     # open                 0
     LV Size                2.00 GiB
     Current LE             512
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:1
      
     --- Logical volume ---
     LV Path                /dev/centos/root
     LV Name                root
     VG Name                centos
     LV UUID                Fdnm2m-xZIJ-hJyB-t57l-5hBX-3qwv-gaWro6
     LV Write Access        read/write
     LV Creation host, time localhost.localdomain, 2016-07-26 12:00:02 +0800
     LV Status              available
     # open                 1
     LV Size                <17.47 GiB
     Current LE             4472
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:0
      
     --- Logical volume ---
     LV Path                /dev/bigdatavg1/bigdatalv1
     LV Name                bigdatalv1
     VG Name                bigdatavg1
     LV UUID                UvXCBL-QqAd-KzTX-vjn7-30eU-SAfU-qe9c61
     LV Write Access        read/write
     LV Creation host, time F5ELK-NODE01.myf5.net, 2018-02-14 12:35:50 +0800
     LV Status              available
     # open                 1
     LV Size                <50.00 GiB
     Current LE             12799
     Segments               1
     Allocation             inherit
     Read ahead sectors     auto
     - currently set to     8192
     Block device           253:2
   ```

4. 默认虚机内存6G

5. ELK版本为5.6

6. 虚机镜像硬件兼容性为V11版本，支持，fusion7,fusion8,workstation11.x,workstation12.x,ESXI6.0



## 镜像使用方法

1. 导入镜像ova文件，根据自己网络环境配置网络
2. 磁盘，建议SSD硬盘。ELK对数据存储容量要求较高，根据接收数据量的不同，很可能多达每日数百G乃至10余T，如果在生产环境使用，请注意观察评估磁盘使用量。根据实际需要扩容已有的两个ELK数据存储盘内容，或增加更多硬盘（注意增加新硬盘并挂载后，需要修改elasticsearch配置文件）。扩容现有硬盘办法参考<http://www.mamicode.com/info-detail-1456892.html> ，增加新硬盘并挂载参考https://github.com/myf5/f5-elk-demo/blob/master/%E5%A2%9E%E5%8A%A0%E8%99%9A%E6%9C%BA%E7%A1%AC%E7%9B%98%E5%B9%B6%E6%8C%82%E8%BD%BD%E4%BD%9C%E4%B8%BAelk%E6%95%B0%E6%8D%AE%E7%9B%98
3. 虚机内存建议在16G以上
4. 虚机CPU建议在4核以上
5. 如果实际使用中，发现硬盘容量无法满足要求，需及时扩容。或使用该链接中的方法配置elasticsearch对硬盘的使用阀值限制https://github.com/myf5/f5-elk-demo/blob/master/4.%E6%8E%A7%E5%88%B6%20elasticsearch%E5%AF%B9%E7%A3%81%E7%9B%98%E7%9A%84%E4%BD%BF%E7%94%A8%E9%87%8F.md



## ELK相关配置调整

虚机设定好内存，硬盘并启动后，请检查并确认以下相关配置文件:

1. /etc/elasticsearch/elasticsearch.yml 文件中的path.data部分，如果有新增加硬盘并挂载到新的数据目录，需要在这里增加新目录。现有硬盘扩容或未变化，则保持此部分不动
2. 确认elasticsearch jvm配置已根据物理内存大小调整好，修改虚机内存后，参考该连接修改相应elasticsearch jvm配置：https://github.com/myf5/f5-elk-demo/blob/master/3.%E8%B0%83%E6%95%B4elasticsearch%20heap%20size.txt
3. logstash配置，/etc/logstash/conf.d/ 目录下均为有效配置，此目录不能放置无用文件。默认已配置f5-request-logging-xff.conf ，默认该配置能够根据是否有XFF头做GEOIP，因此如果请求源地址是私有地址但包含XFF（XFF格式为 publicip,privateip1,privateip2，否则无法处理），或请求是公网地址的都可以正确使用地图显示。’没有XFF且F5看到的源地址为私有地址的，无法使用地图。

```
[root@F5ELK-NODE01 conf.d]# cat f5-request-logging-xff.conf 
input {
  tcp {
    port => 8514
    type => 'f5-request'
  }
}

filter {
 if [type] == "f5-request" {
      grok {
        match => { "message" => "%{IP:clientip} %{DATA:xff} \[%{HTTPDATE:timestamp}\] %{IP:virtual_ip} %{DATA:virtual_name} %{DATA:virtual_pool_name} %{DATA:server} %{NUMBER:server_port} \"%{DATA:path}\" \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\" %{NUMBER:response:int} %{NUMBER:bytes:int} %{NUMBER:response_ms:int} %{QS:referrer} %{QS:agent}"}
    }
  }

 if [xff] {
    mutate {
        gsub => [
            "xff", ",.*", ""
        ]
      }

    geoip {
        source => "xff"
        target => "geoip"
      }
 } else {

    geoip  {
        source => "clientip"
        target => "geoip"
      }
 }

}

output {
  elasticsearch {
    hosts => ["192.168.214.130:9200"]
    index => "f5-request-%{+YYYY.MM.dd}"
  }
}
```

4. 根据实际网卡地址，需改上述logstash配置中的最后IP部分，修改为实际地址



## ELK使用

1. ELK三个应用都已注册为系统服务，开机自动启动
2. 应用访问： http://yourip:9100  图形化查看elasticsearch状态，在打开的界面中的connect按钮之前输入框中输入http://yourip:9200/并点击connect按钮，应可以看到页面ELKnode1后面出现绿色方框。否则请检查elasticsearch服务状态及其日志排错。
3. http://yourip:5601为Kibana
4. Kibana默认已经提供了一个较为通用的可视化设置，点击Kibana左侧dashboard按钮，并点击F5_Request_Logging 可查看（前提系统已开始接收数据）
5. Kibana中部分需要自定义的部分可视图已在dashboard界面中提示，请自行根据需要调整
6. 访问kibana的机器浏览器需要连接互联网，否则无法出现地图。ELK服务器本身不需要互联网
7. 当前内置的dashboard是基于一个应用（所有VS）视角来配置，多个应用混合到一个dashboard中也可以，但某些可视图需要调整（这类图一般因为检索会和具体VS强耦合）
8. 缺省试图之外的需求，需自行定义

## F5配置

1. 参考https://www.myf5.net/post/2491.htm中关于F5配置的提示，其中template配置填入以下内容（不换行）

```
$CLIENT_IP ${X-Forwarded-For} $DATE_NCSA $VIRTUAL_IP $VIRTUAL_NAME $VIRTUAL_POOL_NAME $SERVER_IP $SERVER_PORT "$HTTP_PATH" "$HTTP_REQUEST" $HTTP_STATCODE $RESPONSE_SIZE $RESPONSE_MSECS "$Referer" "${User-agent}"
```

2. logstash, 监听接口为 **TCP 8514**
3. 应使得数据发送使用TMM业务口，不要使用管理口

## 其它注意事项

1. ELK机器为单节点，未配置cluster
2. 图形界面查看elasticsearch时候，请切勿删除   ***.kibana***  这个index
3. 如需删除某个index，点击对应index上部的 Actions 下拉按钮，并点击删除。请切勿删除   ***.kibana***  这个index
4. 其它未尽事项，请参考<https://gitlab.es.f5net.com/jlin/elk-image/blob/master/README.MD>以及https://github.com/myf5/f5-elk-demo/
5. 配置多台ELK cluster，多个logstash作为HSL服务器情形，需自行研究
6. 镜像挂远端存储方式，需自行设置