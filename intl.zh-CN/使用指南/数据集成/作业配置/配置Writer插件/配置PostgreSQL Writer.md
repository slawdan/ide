# 配置PostgreSQL Writer {#concept_pyy_trn_q2b .concept}

本文为您介绍PostgreSQL Writer支持的数据类型、写入方式、字段映射和数据源等参数及配置举例。

PostgreSQL Writer插件实现了向PostgreSQL写入数据。在底层实现上，PostgreSQL Writer通过JDBC连接远程PostgreSQL数据库，并执行相应的SQL语句将数据从PostgreSQL库中SELECT出来，公共云上RDS提供PostgreSQL存储引擎。

**说明：** 在开始配置PostgreSql Writer插件前，请首先配置好数据源，详情请参见[配置PostgreSQL数据源](intl.zh-CN/使用指南/数据集成/数据源配置/配置PostgreSQL数据源.md#)。

简而言之，PostgreSQL Writer通过JDBC连接器连接到远程的PostgreSQL数据库，根据您配置的信息生成查询SELECT SQL语句并发送到远程PostgreSQL数据库，并将该SQL执行返回结果使用CDP自定义的数据类型拼装为抽象的数据集，并传递给下游Writer处理。

-   对于您配置的table、column和where等信息，PostgreSQL Writer将其拼接为SQL语句发送到PostgreSQL数据库。
-   对于您配置的querySql信息，PostgreSQL直接将其发送到PostgreSQL数据库。

## 类型转换列表 {#section_vcg_fvn_q2b .section}

PostgreSQL Writer支持大部分PostgreSQL类型，但也存在部分类型没有支持的情况，请注意检查您的类型。

PostgreSQL Writer针对PostgreSQL的类型转换列表，如下所示。

|数据集成内部类型|PostgreSQL数据类型|
|:-------|:-------------|
|LONG|BIGINT、BIGSERIAL、INTEGER、SMALLINT和SERIAL。|
|DOUBLE|DOUBLE、PRECISION、MONEY、NUMERIC和REAL。|
|STRING|VARCHAR、CHAR、TEXT、BIT和INET|
|DATE|DATE、TIME和TIMESTAMP|
|BOOLEAN|BOOL|
|BYTES|BYTEA|

**说明：** 

-   除上述罗列字段类型外，其他类型均不支持。
-   MONEY、INET和BIT需要您使用`a_inet::varchar`类似的语法进行转换。

## 参数说明 {#section_jn2_gqh_p2b .section}

|参数|描述|是否必选|默认值|
|:-|:-|:---|:--|
|datasource|数据源名称，脚本模式支持添加数据源，此配置项填写的内容必须要与添加的数据源名称保持一致。|是|无|
|table|选取的需要同步的表名称。|是|无|
|writeMode|选择导入模式，可以支持insert和copy方式。 -   insert：执行PostgreSQL的`insert into...values...`语句，将数据写出到PostgreSQL中。当数据出现主键/唯一性索引冲突时，待同步的数据行写入PostgreSQL失败，当前记录行成为脏数据。建议您优先选择insert模式。
-   copy：PostgreSQL提供copy命令，用于表与文件（标准输出，标准输入）之间的相互复制。数据集成支持使用`copy from`，将数据加载到表中。建议您在遇到性能问题时再尝试使用该模式。

 |否|insert|
|column|目标表需要写入数据的字段，字段之间用英文逗号分隔。例如`"column":["id","name","age"]`。如果要依次写入全部列，使用（\*）表示，例如`"column":["*"]`。|是|无|
|preSql|执行数据同步任务之前率先执行的SQL语句。目前向导模式仅允许执行一条SQL语句，脚本模式可以支持多条SQL语句，例如清除旧数据。|否|无|
|postSql|执行数据同步任务之后执行的SQL语句。目前向导模式仅允许执行一条SQL语句，脚本模式可以支持多条SQL语句，例如加上某一个时间戳。|否|无|
|batchSize|一次性批量提交的记录数大小，该值可以极大减少数据集成与PostgreSQL的网络交互次数，并提升整体吞吐量。但是该值设置过大可能会造成数据集成运行进程OOM情况。|否|1024|
|pgType|PostgreSQL特有类型的转化配置，支持bigint\[\]、double\[\]、text\[\]、jsonb和json类型。配置示例如下： ```
{
    "job": {
        "content": [{
            "reader": {...},
            "writer": {
                "parameter": {
                    "column": [
                        // 目标表字段列表
                        "bigint_arr",
                        "double_arr",
                        "text_arr",
                        "jsonb_obj",
                        "json_obj"
                    ],
                    "pgType": {
                        // 特殊的类型设置，key为目标表的字段名，value为字段类型。
                        "bigint_arr": "bigint[]",
                        "double_arr": "double[]",
                        "text_arr": "text[]",
                        "jsonb_obj": "jsonb",
                        "json_obj": "json"
                    }]
                }
            }
        }]
    }
}
```

 |否|无|

## 向导开发介绍 {#section_bp2_wsh_p2b .section}

1.  选择数据源

    配置同步任务的数据来源和数据去向。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16253/15590952288214_zh-CN.png)

    |配置|说明|
    |:-|:-|
    |**数据源**|即上述参数说明中的datasource，一般填写您配置的数据源名称。|
    |**表**|即上述参数说明中的table。|
    |**导入前准备语句**|即上述参数说明中的preSql，输入执行数据同步任务之前率先执行的SQL语句。|
    |**导入后完成语句**|即上述参数说明中的postSql，输入执行数据同步任务之后执行的SQL语句。|
    |**导入模式**|即上述参数说明中的writeMode，包括**insert**和**copy**两种模式。|

2.  字段映射，即上述参数说明中的column。

    左侧的源头表字段和右侧的目标表字段为一一对应的关系，单击**添加一行**可增加单个字段，鼠标放至需要删除的字段上，即可单击**删除**图标进行删除。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16253/15590952288215_zh-CN.png)

    -   同行映射：单击**同行映射**可以在同行建立相应的映射关系，请注意匹配数据类型。
    -   自动排版：可以根据相应的规律自动排版。
3.  通道控制

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/16221/15590952297675_zh-CN.png)

    |配置项|说明|
    |:--|:-|
    |**DMU**|数据集成消耗资源（包括CPU、内存、网络等资源分配）的度量单位。一个DMU描述了一个数据集成作业最小运行能力，即在限定的CPU、内存、网络等资源情况下对于数据同步的处理能力。|
    |**作业并发数**|可将此属性视为数据同步任务内，可从源并行读取或并行写入数据存储端的最大线程数。向导模式通过界面化配置并发数，指定任务所使用的并行度。|
    |**同步速率**|设置同步速率可保护读取端数据库，以避免抽取速度过大，给源库造成太大的压力。同步速率建议限流，结合源库的配置，请合理配置抽取速率。|
    |**错误记录数**|错误记录数，表示脏数据的最大容忍条数。|
    |**任务资源组**|任务运行的机器，如果任务数比较多，使用默认资源组出现等待资源的情况，建议添加自定义资源组（目前只有华东1，华东2支持添加自定义资源组），详情请参见[新增任务资源](intl.zh-CN/使用指南/数据集成/常见配置/新增任务资源.md#)。|


## 脚本开发介绍 {#section_cp2_wsh_p2b .section}

脚本配置样例如下，具体参数填写请参见参数说明。

```
{
    "type":"job",
    "version":"2.0",//版本号
    "steps":[ //Reader的模板如下所示，您可以查看相应数据源的Reader文档。
        {
            "stepType":"stream",
            "parameter":{},
            "name":"Reader",
            "category":"reader"
        },
        {
            "stepType":"postgresql",//插件名
            "parameter":{
                "postSql":[],//执行数据同步任务之后率先执行的SQL语句。
                "datasource":"//数据源
                    "col1",
                    "col2"
                ],
                "table":"",//表名
                "preSql":[]//执行数据同步任务之前率先执行的SQL语句。
            },
            "name":"Writer",
            "category":"writer"
        }
    ],
    "setting":{
        "errorLimit":{
            "record":"0"//错误记录数
        },
        "speed":{
            "throttle":false,//false代表不限流，下面的限流的速度不生效，true代表限流。
            "concurrent":1,//作业并发数
            "dmu":1//DMU值
        }
    },
    "order":{
        "hops":[
            {
                "from":"Reader",
                "to":"Writer"
            }
        ]
    }
}
```

