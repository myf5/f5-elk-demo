### 旁路LTM模式下对接N-Compass

* 按照F5 passive 模式配置基本配置（镜像，网络，vs等）

* 关联以下irule

  ```tcl
  when CLIENT_ACCEPTED {
    set CLIENT_IP [IP::client_addr]
    set CLIENT_PORT [TCP::client_port]
    set DATE_NCSA [clock format [clock seconds] -format "%d/%m/%y %H:%M:%S %z"]
    set VIRTUAL_IP [IP::local_addr]
    set VIRTUAL_PORT [TCP::local_port]
    set VIRTUAL_NAME [virtual name]
    set VIRTUAL_POOL_NAME [LB::server pool]
    set BIGIP_HOSTNAME $static::tcl_platform(machine)
    set VIRTUAL_SNATPOOL_NAME [LB::snat]
  }

  when HTTP_REQUEST {
    set HTTP_PATH [HTTP::uri]
  #  set HTTP_REQUEST "[HTTP::method] [HTTP::uri] HTTP/[HTTP::version]" 
    set Referer [HTTP::header "Referer"]
    set Host [HTTP::header "Host"]
    set HTTP_METHOD [HTTP::method]
    set HTTP_VERSION [HTTP::version]
    set HTTP_KEEPALIVE [HTTP::is_keepalive]

    set Useragent [HTTP::header "User-agent"]

  }


  when HTTP_REQUEST_SEND {
    set send_start_time [clock clicks -milliseconds]
  }

  when HTTP_RESPONSE {
    set hsl [HSL::open -proto UDP -pool pool_request_log]
    set SERVER_IP [IP::server_addr]
    set SERVER_PORT [TCP::server_port]
    set HTTP_STATCODE [HTTP::status]
    set RESPONSE_SIZE [HTTP::header "Content-Length"]
    set RESPONSE_MSECS [ expr  [clock clicks -milliseconds] -  $send_start_time] 
    set SNAT_IP [IP::local_addr]
    set SNAT_PORT [TCP::local_port]
    
  HSL::send $hsl "http||$BIGIP_HOSTNAME||$Host||$CLIENT_IP||$CLIENT_PORT||$DATE_NCSA||$SERVER_IP||$SERVER_PORT||$VIRTUAL_IP||$VIRTUAL_PORT||$VIRTUAL_NAME||$VIRTUAL_POOL_NAME||$VIRTUAL_SNATPOOL_NAME||$Referer||$Useragent||$HTTP_KEEPALIVE||$HTTP_METHOD||$HTTP_PATH||$HTTP_VERSION||$RESPONSE_SIZE||$RESPONSE_MSECS||$SNAT_IP||$SNAT_PORT||$HTTP_STATCODE||[HTTP::header "Content-Type"]||$DATE_NCSA"

  }

  ```

  示例输出日志:

  ```
  May 11 11:13:45 172.16.17.33 http||ltm12.f5.com||10.10.17.63||10.10.17.8||50111||29/09/17 15:48:04 +0800||172.16.20.1||80||10.10.17.63||80||/Common/vs_http_80_63||/Common/pool_http_80||snat automap||http://10.10.17.63/mp3/paomo.mp3||Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36||1||GET||/mp3/paomo.mp3||1.1||4143955||9||172.16.17.33||50111||200||audio/mpeg||29/09/17 15:48:04 +0800
  ```

  ​

与N-compass更多对接细节或支持，请联系N-Compass技术支持