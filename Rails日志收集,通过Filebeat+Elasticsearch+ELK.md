# 查询日志格式
通过命令查看tail -f /var/www/readio_server/shared/log/production.log 日志格式如下:
```
method=GET path=/admin/cms/pages/5ce51365b2cdd355261ac66f format=html controller=Admin::Cms::PageController action=show status=200 duration=74.20 params={}
method=GET path=/admin/message_sessions format=json controller=Admin::MessageSessionsController action=index status=200 duration=7.26 params={}
method=GET path=/admin/message_templates/common format=json controller=Admin::MessageTemplatesController action=show status=200 duration=15.92 params={}
method=GET path=/admin/packages/5c63bc81b2cdd3763c1bd37c/homework_shares/workbench format=*/* controller=Admin::Packages::HomeworkSharesController action=workbench status=200 duration=698.08 params={"package_id"=>"5c63bc81b2cdd3763c1bd37c"}
method=GET path=/admin/students/5cefe28d1c94e517a4f3bc4e/profile format=json controller=Admin::Students::ProfilesController action=show status=200 duration=141.03 params={"student_id"=>"5cefe28d1c94e517a4f3bc4e"}
method=GET path=/admin/students/5cefe28d1c94e517a4f3bc4e/families format=json controller=Admin::Students::FamiliesController action=index status=200 duration=173.66 params={"student_id"=>"5cefe28d1c94e517a4f3bc4e"}
method=GET path=/admin/packages/5c8f53b01c94e517aef71b55/marvins format=json controller=Admin::Packages::MarvinsController action=index status=200 duration=169.80 params={"package_id"=>"5c8f53b01c94e517aef71b55"}
method=GET path=/admin/packages/5c63bc81b2cdd3763c1bd37c/students_messages/claim_message format=html controller=Admin::Packages::StudentsMessagesController action=claim_message status=302 duration=192.18 location=http://www.xiguacity.cn/admin params={"package_id"=>"5c63bc81b2cdd3763c1bd37c"}
method=GET path=/admin format=html controller=Admin::DashesController action=show status=200 duration=370.45 params={}
method=GET path=/admin/students/5c6b6b9a1c94e555f5557c78 format=js controller=Admin::StudentsController action=show status=200 duration=653.20 params={}
method=GET path=/admin/students/5c6a53461c94e571d25338b7/transfer/new format=js controller=Admin::Students::TransfersController action=new status=200 duration=511.78 params={"enrollment_id"=>"5ceccfeab2cdd3600d485e9e", "package_id"=>"5c8b3f101c94e5129e13bf2e", "student_id"=>"5c6a53461c94e571d25338b7"}
method=GET path=/admin/cms/pages/5ce62d92b2cdd311f011f4d4 format=html controller=Admin::Cms::PageController action=show status=200 duration=25.47 params={}
method=GET path=/admin format=html controller=Admin::DashesController action=show status=200 duration=362.86 params={}
method=GET path=/admin/cms/pages/5ce3d42eb2cdd323400ee0d1 format=html controller=Admin::Cms::PageController action=show status=200 duration=29.78 params={}
```
# 通过ELK中的Dev Tools进行日志解析
![](http://m.caixinglong.cn/blog/2019-08-01-060140.jpg)

##  输入原始数据和正则表达式进行日志解析
![](http://m.caixinglong.cn/blog/2019-08-01-060219.jpg)

## 正则表达式规则
https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns

# 根据解析的正则表达式配置Elasticsearch的pipeline
## 在服务器上创建一个pipeline的json格式文件,如:readio-server-pipeline.json

注意:根据正则表达式生产pipeline时需要进行适当转义
```
{
  "description": "readio-server-pipeline",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "\\w+\\S%{WORD:method}\\s+\\w+\\S%{URIPATHPARAM:path}\\s+\\w+\\S%{GREEDYDATA:format}\\s+\\w+\\S%{NOTSPACE:controller}\\s+\\w+\\S%{WORD:action}\\s+\\w+\\S%{WORD:status}\\s+\\w+\\S%{NOTSPACE:duration}\\s+\\w+\\S%{GREEDYDATA:params}","%{GREEDYDATA:log}"
        ]
      }
    }
  ]
}
```
## 把定义的pipeline写入Elasticsearch的规则中
```
curl -H 'Content-Type: application/json' -XPUT http://localhost:9200/_ingest/pipeline/readio-server-pipeline -d@readio-server-pipeline.json
```
# 定义Filebeat如何收集和输出日志到Elasticsearch中
## 定义Filebeat的日志输入源
```yml
- type: log
 
  # Change to true to enable this input configuration.
  enabled: true
 
  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /var/www/readio_server/shared/log/access.log
 
  fields:
      service: nginx-log
 
- type: log
  enable: true
  paths:
    - /var/www/readio_server/shared/log/production.log
  fields:
      service: readio-server-log
  scan_frequency: 10s
```
## 定义Filebeat的日志输出源(Elasticsearch)
```yml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["172.17.191.9:9200"]
  indices:
    - index: "nginx-access-%{+yyy.MM.dd}"
      when.equals:
        fields.service: "nginx-log"
    - index: "readio-server-%{+yyy.MM.dd}"
      when.equals:
        fields.service: "readio-server-log"
  pipelines:
    - pipeline: "filebeat-6.4.0-nginx-access-default"
      when.equals:
        fields.service: "nginx-log"
    - pipeline: "readio-server-pipeline"
      when.equals:
        fields.service: "readio-server-log"
```
## 停止filebeat使用命令行观察是否报错
停止filebeat:   
```
sudo service filebeat stop
```
命令行观察日志解析是否报错:  
```
sudo filebeat -e -c filebeat.yml -d "*"    
或者 
sudo filebeat -e -c filebeat.yml -d "publish"
```
# ELK配置查看日志
## 确认日志是否收集到Elasticsearch
![](http://m.caixinglong.cn/blog/2019-08-01-061211.jpg)

## ELK配置查询索引
![](http://m.caixinglong.cn/blog/2019-08-01-061234.jpg)

点击创建索引
![](http://m.caixinglong.cn/blog/2019-08-01-061300.jpg)

# 根据创建的索引查询日志
![](http://m.caixinglong.cn/blog/2019-08-01-061325.jpg)





# 正则表达式例子
```shell
method=GET path=/admin/user_infos/58e4ca765f94a64566240e63 format=html controller=Admin::UserInfosController action=show status=200 duration=908.33 params={"situation"=>"coupon”}

\w+\S%{WORD:method}\s+\w+\S%{URIPATHPARAM:path}\s+\w+\S%{WORD:format}\s+\w+\S%{NOTSPACE:controller}\s+\w+\S%{WORD:action}\s+\w+\S%{WORD:status}\s+\w+\S%{NOTSPACE:duration}\s+\w+\S%{NOTSPACE:params}
```

