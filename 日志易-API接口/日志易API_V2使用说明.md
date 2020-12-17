### 基本配置

> 测试环境

- ip：192.20.103.196/197/206

- port: 8090

- Token：b8f16edcc0f339b85faae57c77aa4b76

- username：api

- password：Xtyxz@czj2

- operator：api

  与username一致

**PS：**账号密码只做测试使用(可用)，实际账号取决于手上分配的账号

### API接口

> 以下提供了俩种查询接口，非流式与流式接口，俩者区别为
>
> 非流式：提交任务直接等待接口返回结果
>
> 流式接口：提交任务，接口返回sid号，再通过sid去查询接口，为异步实现

**注意：** query参数需进行一次urlencode

1. 提交非流式搜索

   这里以【提交非流式搜索任务】为例，直接等待rizhiyi返回结果，搜索query参数为

   ```
   index=monitor alert_id:* NOT issue_alert:false alert_level:* AND result.result.extend_hits.info:*
   |table timestamp,alert_name,alert_id,alert_history_id,alert_level,result.result.extend_result.sheets.rows.info,result.is_alert_recovery
   |append [[
   	index=monitor alert_id:* NOT issue_alert:false alert_level:* NOT result.result.extend_hits.info:*
   	|table timestamp,alert_name,alert_id,alert_history_id,alert_level,result.result.extend_result.sheets.rows.info,result.is_alert_recovery
   ]]
   |append [[
   	index=monitor result.is_alert_recovery:1
   	|table timestamp,alert_name,alert_id ,result.is_alert_recovery,alert_history_id
   ]]
   |eval alert_level =if(empty(alert_level ),"null",alert_level )
   |sort by timestamp
   |fields timestamp,alert_name,alert_id,alert_history_id,alert_level,result.result.extend_result.sheets.rows.info,result.is_alert_recovery|eval timestamp=formatdate(timestamp)
   ```

   示例

   > 账号密码按照 username:passwd 格式进行base64加密
   >
   > time_range参数为搜索限定时间，可支持相对时间和绝对时间，绝对时间支持unix时间
   >
   > 建议使用unix时间，为13位unix时间戳

   ```shell
   curl --location --request GET 'http://192.20.103.196:8090/api/v2/search/sheets/?time_range=now/m-5m,now/m&query=index%3dmonitor+alert_id%3a*+NOT+issue_alert%3afalse+alert_level%3a*+AND+result.result.extend_hits.info%3a*%0A%7cstats+count()+by+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.strategy.description%2cresult.is_alert_recovery%0A%7cappend+%5b%5b%0A%09index%3dmonitor+alert_id%3a*+NOT+issue_alert%3afalse+alert_level%3a*+NOT+result.result.extend_hits.info%3a*%0A%09%7cstats+count()+by+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.strategy.description%2cresult.is_alert_recovery%0A%5d%5d%0A%7cappend+%5b%5b%0A%09index%3dmonitor+result.is_alert_recovery%3a1%0A%09%7ctable+timestamp%2calert_name%2calert_id+%2cresult.is_alert_recovery%2calert_history_id%0A%5d%5d%0A%7ceval+alert_level+%3dif(empty(alert_level+)%2c%22null%22%2calert_level+)%0A%7csort+by+%2btimestamp%0A%7cfields+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.is_alert_recovery&category=search&fields=false&statsevents=false&size=1000&timeline=false' \
   --header 'Authorization: Basic YXBpOlh0eXh6QGN6ajI='
   ```

   返回结果

   ```json
   {
       "rc": 0,
       "traceid": "c49c910d7a4a4e79b4a78e016e7e4bb0",
       "results": {
           "loseDataCommands": null,
           "starttime": 1601343360000,
           "sheets": {
               "rows": [
                   {
                       "timestamp": 1601343780000,
                       "alert_level": "low",
                       "alert_name": "ces",
                       "alert_id": 13,
                       "result.is_alert_recovery": 0,
                       "alert_history_id": "13_1601343780000_0"
                   }
               ],
               "group_by_num": 0,
               "total": 8,
               "version": 1,
               "_field_infos_": [
                   {
                       "type": "unknown",
                       "name": "timestamp",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_name",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_id",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_history_id",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_level",
                       "groupby": false
                   },
                   {
                       "type": "unknown",
                       "name": "result.is_alert_recovery",
                       "groupby": true
                   }
               ]
           },
           "total_hits": 0,
           "endtime": 1601343960000,
           "type": "stats"
       },
       "result": true
   }
   ```

2. 提交流式搜索

   这里以【提交流式搜索任务】为例，提交任务获取sid，以下是parms中的query参数

   ```
   index=monitor alert_id:* NOT issue_alert:false alert_level:* AND result.result.extend_hits.info:*
   |stats count() by timestamp,alert_name,alert_id,alert_history_id,alert_level,result.strategy.description,result.is_alert_recovery
   |append [[
   	index=monitor alert_id:* NOT issue_alert:false alert_level:* NOT result.result.extend_hits.info:*
   	|stats count() by timestamp,alert_name,alert_id,alert_history_id,alert_level,result.strategy.description,result.is_alert_recovery
   ]]
   |append [[
   	index=monitor result.is_alert_recovery:1
   	|table timestamp,alert_name,alert_id ,result.is_alert_recovery,alert_history_id
   ]]
   |eval alert_level =if(empty(alert_level ),"null",alert_level )
   |sort by +timestamp
   |fields timestamp,alert_name,alert_id,alert_history_id,alert_level,result.is_alert_recovery
   ```

   示例

   > 账号密码按照 username:passwd 格式进行base64加密
   >
   > time_range参数为搜索限定时间，可支持相对时间和绝对时间，绝对时间支持unix时间
   >
   > 建议使用unix时间，为13位unix时间戳

   ```shell
   curl --location --request GET 'http://192.20.103.196:8090/api/v2/search/submit/?time_range=now/m-10m,now/m&query=index%3dmonitor+alert_id%3a*+NOT+issue_alert%3afalse+alert_level%3a*+AND+result.result.extend_hits.info%3a*%0A%7cstats+count()+by+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.strategy.description%2cresult.is_alert_recovery%0A%7cappend+%5b%5b%0A%09index%3dmonitor+alert_id%3a*+NOT+issue_alert%3afalse+alert_level%3a*+NOT+result.result.extend_hits.info%3a*%0A%09%7cstats+count()+by+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.strategy.description%2cresult.is_alert_recovery%0A%5d%5d%0A%7cappend+%5b%5b%0A%09index%3dmonitor+result.is_alert_recovery%3a1%0A%09%7ctable+timestamp%2calert_name%2calert_id+%2cresult.is_alert_recovery%2calert_history_id%0A%5d%5d%0A%7ceval+alert_level+%3dif(empty(alert_level+)%2c%22null%22%2calert_level+)%0A%7csort+by+%2btimestamp%0A%7cfields+timestamp%2calert_name%2calert_id%2calert_history_id%2calert_level%2cresult.is_alert_recovery&category=search&fields=false&statsevents=false&size=1000&timeline=false' \
   --header 'Authorization: Basic YXBpOlh0eXh6QGN6ajI='
   ```

   返回结果

   ```json
   {
       "sid": "5ece0ad649e748f0f781c47d623e5c11c01467c5",
       "traceid": "14548cce3a0c4ef781e343df4f0efcbc",
       "result": true,
       "rc": 0
   }
   ```

3. 从sid中preview结果

   上面已经提交了一个任务，并返回了sid值，通过这个值可以那会统计结果

   示例

   > 账号密码按照 username:passwd 格式进行base64加密

   ```shell
   curl --location --request GET 'http://192.20.103.196:8090/api/v2/search/fetch/?sid=5ece0ad649e748f0f781c47d623e5c11c01467c5&category=sheets&size=1000' \
   --header 'Authorization: Basic YXBpOlh0eXh6QGN6ajI='
   ```

   返回结果

   > 其中result.sheets.row为统计结果

   ```json
   {
       "job_status": "COMPLETED",
       "traceid": "172b73dfb1484d2abb3f02ef9b17c534",
       "preview_is_null": false,
       "results": {
           "total_hits": 0,
           "endtime": 1601344260000,
           "starttime": 1601343660000,
           "sheets": {
               "rows": [
                   {
                       "timestamp": 1601343660000,
                       "alert_level": "low",
                       "alert_name": "ces",
                       "alert_id": 13,
                       "result.is_alert_recovery": 0,
                       "alert_history_id": "13_1601343660000_0"
                   }
               ],
               "group_by_num": 0,
               "total": 3,
               "version": 1,
               "_field_infos_": [
                   {
                       "type": "unknown",
                       "name": "timestamp",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_name",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_id",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_history_id",
                       "groupby": true
                   },
                   {
                       "type": "unknown",
                       "name": "alert_level",
                       "groupby": false
                   },
                   {
                       "type": "unknown",
                       "name": "result.is_alert_recovery",
                       "groupby": true
                   }
               ]
           },
           "hint_message": null
       },
       "result": true,
       "rc": 0,
       "progress": 100,
       "type": "stats"
   }
   ```