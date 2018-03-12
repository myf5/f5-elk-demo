### F5 L4 VS 监控pool member延迟数据

在一些场景下，尽管VS 类型是L4 VS，但依旧希望对服务器的延迟进行监控，例如针对SSL offload池中的SSL卸载设备的性能进行监控，防止连接可以建立，但是SSL卸载处理已经变得很慢情况，及时隔离有问题的SSL卸载设备。对于出向链路的VS则可以用于观察链路延迟情况参考。

<img src="https://github.com/myf5/f5-elk-demo/blob/master/l4/Screenshot-2018-3-11%20L4_Dashboard%20-%20Kibana.png?raw=true">

在本例中，irule采集了握手延迟，首包响应延迟以及连续响应包情形下两包之间的延迟（采样）

* 握手延迟：TCP三次握手中，服务器SYN+ACK的响应延迟
* 首包响应延迟：服务器对客户端每个发送数据的首包响应延迟，该数据持续跟踪在一个tcp链接中，每次客户端发送tcp数据后，服务器的第一个响应数据到达F5的延迟，通过统计该数据可以有效发现后端服务器在对一个连接的整个响应过程中延迟状态。
* 连续响应包之间延迟: 如果服务器连续返回数据，则计算两个响应包之间的延迟，irule容许设置采样间隔，即每连续响应多少个包采样一次间隔



#### irule：

```tcl
when RULE_INIT {
# V1.1. 
#Pls check https://github.com/myf5/f5-elk-demo/l4 for updated version
#This irule work ONLY FOR L4 TYPE VS,DON NOT USE FOR STD VS
#Any question pls create issue on the github,Will be best effort support. Welcome PR if you had enhenced/fixed it.
#The irule will caculates the initial response packet delay for each tcp client data request
#The irule also caculates the delay between continuous response packets, but sampling for below interval
#Which means, get a delay every 5 continuous tcp response packets. Bigger number cause less sampling.
#If you think there are too many logs were sent to ELK, pls adjust it bigger.
set static::interval 5
#You need create a LTM pool named as below pool name, here is logstash-pool_8515
set static::hslpool "logstash-pool_8515"
# set it to 1 in case you need debug
set static::debug 1
}
when CLIENT_ACCEPTED {
    set cd 0
    set sd 0
    set server_data_time_last 0
    set skip 0
    set hsf 1
    set hsl [HSL::open -proto TCP -pool $static::hslpool]
    set client_remote "[IP::client_addr] [TCP::client_port]"
    set client_local  "[IP::local_addr clientside] [TCP::local_port clientside]"
    if {($static::debug == 1)} {
        log local0. "$client_remote $client_local SYN is coming. [clock clicks -milliseconds]"
    }
}

when CLIENT_DATA {
    incr cd
    set cdf 1
    set sdf 0
    set server_data_time_last 0
    set client_data_time [clock clicks -milliseconds]
    if {($static::debug == 1)} {
        log local0. "$client_remote $client_local cd number: $cd, Client data fired,client data time is $client_data_time"
    }
    if {($hsf == 1)}{
        set cd 0
    }
}


when SERVER_DATA {
    incr sd
    incr sdf
    set server_data_time [clock clicks -milliseconds]
    set server_local  "[IP::local_addr] [TCP::local_port]"
    set server_remote "[IP::server_addr] [TCP::server_port]"
    if {($hsf == 1)}{
        if {($static::debug == 1)} {
            log local0. "$client_remote $client_local $server_local $server_remote sd number: $sd,Server data fired,This is TCP 3HS! server data time is $server_data_time"
            log local0. "The 3HS delay is [expr {$server_data_time-$client_data_time}]"
        }
        set elkdata "$client_remote $client_local $server_local $server_remote HS-delay [expr {$server_data_time-$client_data_time}]\n"
        HSL::send $hsl $elkdata
        set hsf 0
        set cd 0
        set sd 0
        set cdf 0
    } else {
        if {($static::debug == 1)} {
            log local0. "$client_remote $client_local $server_local $server_remote sd number: $sd, Server data fired,server data time is $server_data_time"
        }
        if {($cd >= 1) and ($cdf == 1) }{
        # If client sends several continuous data, then server respond. The delay will be the time between the server data and the last client data
        # If it is a big client data, this may cause false positive. You can check client data number as reference.
        # If the cd number is bigger (like > 10), then you can safely ignore it if the delay value is bigger.
            if {($static::debug == 1)} {
                log local0. "$client_remote $client_local $server_local $server_remote sd number: $sd, ========The delay of initial response packet: [expr {$server_data_time-$client_data_time}]"
            }
            set elkdata "$client_remote $client_local $server_local $server_remote cdnumber $cd init-delay [expr {$server_data_time-$client_data_time}]\n"
            HSL::send $hsl $elkdata
            set cd 0
            set sd 0
            set cdf 0
        }
        if {($sdf > 1) and ($cdf == 0) and ([expr {$sd%$static::interval}] == 0)}{
            set sd_latency [expr {$server_data_time-$server_data_time_last}]
            if {($static::debug == 1)} {
                log local0. "$client_remote $client_local $server_local $server_remote .========server data delay from the last server data:$sd_latency"
            }
            set elkdata "$client_remote $client_local $server_local $server_remote svr-pkts-delay $sd_latency\n"
            HSL::send $hsl $elkdata
         }
        set skip 0
        set server_data_time_last $server_data_time
    }
}
```



#### 使用方法：

* 创建一个HSL pool，该pool为日志接收端地址，例如本例ELK环境下，设置为logstash的服务器地址及端口。该pool名称将被irule引用

  ```shell
  ltm pool logstash-pool_8515 {
      members {
          172.16.199.131:8515 {
              address 172.16.199.131
          }
      }
  }
  ```

  注意： 如果你已经在使用本仓库下提供的L7层 ELK配置，请注意这里设置的是一个不同与之前采集L7 VS数据时配置的logstash端口，这里是8515.

* 拷贝以上irule，并关联到相关VS

* 拷贝以下elasticsearch template文件，并执行命令更新：

  ```json
  [root@F5ELK-NODE01 confbak]# vi f5-l4_template.json

  {
  "template": "f5-l4-*",
  "mappings": {
  "_default_": {
  "_all": {
  "enabled": true,
  "norms": false
  },
  "dynamic_templates": [
  {
  "message_field": {
  "path_match": "message",
  "match_mapping_type": "string",
  "mapping": {
  "type": "text",
  "norms": false
  }
  }
  }
  ,
  {
  "string_fields": {
  "match": "*",
  "match_mapping_type": "string",
  "mapping": {
  "type": "text",
  "norms": false,
  "fields": {
  "keyword": {
  "type": "keyword",
  "ignore_above": 256
  }
  }
  }
  }
  }
  ],
  "properties": {
  "@timestamp": {
  "type": "date",
  "include_in_all": false
  },
  "@version": {
  "type": "keyword",
  "include_in_all": false
  },
  "geoip": {
  "dynamic": true,
  "properties": {
  "ip": {
  "type": "ip"

  ```

  更新命令:

  ```shell
  curl -XPUT http://localhost:9200/_template/f5-l4?pretty -d @f5-l4_template.json 
  ```

  确认获得正确的确认返回信息

* 拷贝以下logstash配置文件到/etc/logstash/conf.d 目录下

  ```
  input {
    tcp {
      port => 8515
      type => 'f5-l4'
    }
  }


  filter {
    if [message]=~ 'init-delay' {
        grok {
          match => { "message" => "%{IP:clientip} %{NUMBER:clientport} %{IP:vsip} %{NUMBER:vsport} %{IP:snatip} %{NUMBER:snatport} %{IP:memberip} %{NUMBER:memberport} cdnumber %{NUMBER:cdnumber:int} init-delay %{NUMBER:init-delay:int}" }
      } 

    } 
    
    if [message]=~ 'HS-delay' { 

        grok {
          match => { "message" => "%{IP:clientip} %{NUMBER:clientport} %{IP:vsip} %{NUMBER:vsport} %{IP:snatip} %{NUMBER:snatport} %{IP:memberip} %{NUMBER:memberport} HS-delay %{NUMBER:hs-delay:int}" }
      }


   } 

   if [message]=~ 'svr-pkts-delay'  {
        grok {
          match => { "message" => "%{IP:clientip} %{NUMBER:clientport} %{IP:vsip} %{NUMBER:vsport} %{IP:snatip} %{NUMBER:snatport} %{IP:memberip} %{NUMBER:memberport} svr-pkts-delay %{NUMBER:svr-pkts-delay:int}" }
      }
     }
    geoip {
          source => "clientip"
          target => "geoip"
      }
  }



  output {
   if [type] == "f5-l4" {
     elasticsearch {
       hosts => ["192.168.214.130:9200"]
       index => "f5-l4-%{+YYYY.MM.dd}"
    }
   }
  }
  ```

  注意：如果你已经在使用其它logstash的配置文件，即同目录下还有其它配置文件，请确保配置文件中的output部分配置了正确的type条件判断，否则可能出现数据被发送到不同的index里。在本例中包含以下两个文件：

  ```shell
  [root@F5ELK-NODE01 conf.d]# pwd
  /etc/logstash/conf.d
  [root@F5ELK-NODE01 conf.d]# ll
  total 8
  -rw-r--r--. 1 root root 788 Mar 11 20:43 f5-l4-logging.conf
  -rwxr-xr-x. 1 root root 925 Mar 11 19:31 f5-request-logging-xff.conf
  ```

  f5-request-logging-xff.conf的内容为：

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
   if [type] == "f5-request" {
     elasticsearch {
      hosts => ["192.168.214.130:9200"]
      index => "f5-request-%{+YYYY.MM.dd}"
    }
   }
  }
  ```

  注意两个文件output部分的判断条件

  另：为了防止大量的连接连接到logstash上，上述两个logstash配置了不同的监听端口

* 重启logstash 服务

* 检查elasticsearch已经收到并创建f5-l4开头的index

* 进入kibana-management-创建新index partern:` f5-l4-*` ，获取一定量数据后，刷新该index

* 创建相关可视化试图，并建立一个独立的dashboard。首包响应延迟关键字段为`init-delay`. 服务器连续响应包延迟采样字段为`svr-pkts-delay`

#### 注意： 

* 以上irule等内容仅在本repro内支持，有任何问题请在这里提交issue
* 如果你修复或增强了irule功能，欢迎提交PR
* **在生产系统使用时，请务必提前做好测试，观察irule可能对性能产生的影响，并观察logstash服务器性能状态（连接，磁盘存储）**.上述示例并未执行性能测试。

