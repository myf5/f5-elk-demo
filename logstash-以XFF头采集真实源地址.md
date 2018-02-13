### VS 处于SNAT设备之后，通过XFF执行GEOIP

1. request logging profile 设置类似如下，增加x-forwarded-for变量

```
$CLIENT_IP ${X-Forwarded-For} $DATE_NCSA $VIRTUAL_IP $VIRTUAL_NAME $VIRTUAL_POOL_NAME $SERVER_IP $SERVER_PORT "$HTTP_PATH" "$HTTP_REQUEST" $HTTP_STATCODE $RESPONSE_SIZE $RESPONSE_MSECS "$Referer" "${User-agent}"
```

2. logstash 参考以下处理

   ```
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

       mutate{
           gsub => [
               "xff", ",.*", ""
           ]
       }

     geoip {
           source => "xff"
           target => "geoip"
       }
   }



   output {
     elasticsearch {
       hosts => ["192.168.214.130:9200"]
       index => "f5-request-%{+YYYY.MM.dd}"
     }
   }
   ```

   以上gsub处理 xff值内容类似, 202.202.202.202,10.1.1.1,192.1.1.1这样格式，第一个IP是用于执行geoip的