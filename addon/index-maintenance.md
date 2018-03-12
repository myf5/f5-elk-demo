

# Add on

以下内容由**华夏银行苏云涛**友情提供

#### ELK index清理脚本

```
[root@F5-ELK-2 ~]# cat elasticsearch-clear.sh 
#/bin/bash

#10 day ago
DATA=`date -d "10 day ago" +%Y.%m.%d`

#now
time=`date`

#delete log
curl -XDELETE http://127.0.0.1:9200/f5-request-${DATA}

if [ $? -eq 0 ];then
echo $time"-->del $DATA log success.." >> /root/elasticsearch-clear.log
else
echo $time"-->del $DATA log fail.." >> /root/elasticsearch-clear.log
fi
```



####ELK index 列表

```
[root@F5-ELK-2 ~]# cat elasticsearch-list.sh 
curl -XGET 'http://127.0.0.1:9200/_cat/indices/?v'
[root@F5-ELK-2 ~]# ./elasticsearch-list.sh 
health status index                 uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   f5-request-2018.03.09 ViQDL_phQYCB5r239vNQRg   5   1   53604141            0     40.6gb         40.6gb
yellow open   f5-request-2018.03.08 eQXJXAcCSlaNtc9Az1Mu2Q   5   1   46468322            0     35.2gb         35.2gb
yellow open   f5-request-2018.03.12 wCkqGXPbQ1isMqKm5JPAlw   5   1   15942560            0     12.5gb         12.5gb
yellow open   f5-request-2018.03.11 ps0XxcblQ3KGmQZXLGI30A   5   1   22247076            0     16.4gb         16.4gb
yellow open   f5-request-2018.03.05 SqNfvPR-SVipdcS8P10hSw   5   1   57641963            0     43.5gb         43.5gb
yellow open   f5-request-2018.03.10 hnl8pq1pQGyZh7oY-UIVxw   5   1   26219755            0     19.5gb         19.5gb
yellow open   f5-request-2018.03.04 ynXEAHoaRmmqNEZQrH3hdg   5   1   23222925            0       17gb           17gb
yellow open   f5-request-2018.03.07 Nm-eFBxXSk6GalPdH9vkWg   5   1   49241738            0     37.1gb         37.1gb
yellow open   f5-request-2018.03.06 H_6JbwRpTSmGk0hMtiYktA   5   1   50833978            0     38.4gb         38.4gb
yellow open   f5-request-2018.03.03 _ic71i6ATgijBvAnSL2N0Q   5   1   25016316            0     18.4gb         18.4gb
yellow open   .kibana               eaNmFraPQfq3hm5UIgOo-Q   1   1         53           25    147.5kb        147.5kb
```

