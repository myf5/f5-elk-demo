### iRule DNS可视化方法：

方法有关的说明文档请访问[查看文章](https://www.myf5.net/post/2581.htm)

iRule

将以下iRule应用在DNS的listener VS上，GTM/DNS系统上配置一个logstash-pool的 LTM pool（如果你发现没有地方配置LTM Pool，联系F5工程师）：

````tcl
when DNS_REQUEST {
    set clientip [IP::client_addr]
    set clientport [UDP::client_port]
    set listenerip [IP::local_addr]
    set requestid [DNS::header id]
    set qname [DNS::question name]
    set qtype [DNS::question type]
}
when DNS_RESPONSE {
    set hsl [HSL::open -proto UDP -pool logstash-pool]
    set status [DNS::header rcode]
    set origin [DNS::origin]
    set elkdata "$static::tcl_platform(machine) $clientip $clientport $listenerip $requestid $qname $qtype $status $origin"
    #log local0. $elkdata
    
    HSL::send $hsl $elkdata
}
````



Logstash配置文件, 注意修改相关IP为你的环境：

```
[root@F5ELK-NODE01 conf.d]# cat dns-log-to-es-irule.conf
input {
  udp {
    port => 8514
    type => 'f5-dns'
  }
}

filter {
 if [type] == 'f5-dns' {
    grok {
          match => { "message" => "%{HOSTNAME:F5hostname} %{IP:clientip} %{POSINT:clientport} %{IP:svrip} %{NUMBER:qid} %{HOSTNAME:qname} %{GREEDYDATA:qtype} %{GREEDYDATA:status} %{GREEDYDATA:origin}" }
    }
    geoip {
         source => "clientip"
         target => "geoip"
    }
    }
}

output {
   #stdout{ codec => rubydebug }
   elasticsearch {
    hosts => ["192.168.214.130:9200"]
    index => "f5-dns-%{+YYYY.MM.dd}"
    template_name => "f5-dns"
  }
}
```



Elasticsearch的template设置，请参考以下连接里的内容

请访问https://www.myf5.net/post/2504.htm

### DNS logging profile可视化方法：

请参考https://www.myf5.net/post/2504.htm