### 如果存在XFF头则用xff作为源地址进行geoip，否则使用client_ip

1. request logging profile中的template 设置类似如下

   更多可用变量请参考 <https://support.f5.com/kb/en-us/products/big-ip-aam/manuals/product/aam-concepts-12-1-0/29.html>

   F5 logging profile配置参考 <https://www.myf5.net/post/2491.htm>

```
$CLIENT_IP ${X-Forwarded-For} $DATE_NCSA $VIRTUAL_IP $VIRTUAL_NAME $VIRTUAL_POOL_NAME $SERVER_IP $SERVER_PORT "$HTTP_PATH" "$HTTP_REQUEST" $HTTP_STATCODE $RESPONSE_SIZE $RESPONSE_MSECS "$Referer" "${User-agent}"
```

2. logstash 供参考

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

   
