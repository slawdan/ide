# 配置HDFS Reader {#concept_azr_4qj_p2b .concept}

HDFS Reader提供了读取分布式文件系统数据存储的能力。在底层实现上，HDFS Reader获取分布式文件系统上文件的数据，并转换为数据集成传输协议传递给Writer。

HDFS Reader实现了从Hadoop分布式文件系统HDFS中，读取文件数据并转为数据集成协议的功能 。

示例如下：

TextFile是Hive建表时默认使用的存储格式，数据不进行压缩。本质上TextFile是以文本的形式将数据存放在HDFS中，对于数据集成而言，HDFS Reader在实现上与OSS Reader有很多相似之处。

ORCFile的全名是Optimized Row Columnar file，是对RCFile的优化，这种文件格式可以提供一种高效的方法来存储Hive数据。HDFS Reader利用Hive提供的OrcSerde类，读取解析ORCFile文件的数据。

**说明：** 

-   由于打通默认资源组到HDFS的网络链路比较复杂，建议您使用自定义资源组完成数据同步任务。您需要确保您的自定义资源组具备HDFS的namenode和datanode的网络访问能力。
-   HDFS默认情况下，使用网络白名单进行数据安全。基于此种情况，建议您使用自定义资源组完成针对HDFS的数据同步任务。
-   您通过脚本模式配置HDFS同步作业，并不依赖HDFS数据源网络连通性测试通过，针对此类错误您可以临时忽略。
-   数据集成同步进程以admin账号启动，您需要确保操作系统的admin账号具备访问相应HDFS文件的读写权限。

## 支持的功能 {#section_fbh_pqj_p2b .section}

目前HDFS Reader支持的功能如下所示：

-   支持TextFile、ORCFile、rcfile、sequence file、csv和parquet格式的文件，且要求文件内容存放的是一张逻辑意义上的二维表。
-   支持多种类型数据读取（使用String表示），支持列裁剪，支持列常量。
-   支持递归读取、支持正则表达式\*和?。
-   支持ORCFile数据压缩，目前支持SNAPPY和ZLIB两种压缩方式。
-   支持sequence file数据压缩，目前支持lzo压缩方式。
-   多个File可以支持并发读取。
-   csv类型支持压缩格式有gzip、bz2、zip、lzo、lzo\_deflate和snappy。
-   目前插件中Hive版本为1.1.1，Hadoop版本为2.7.1（Apache适配JDK1.6］，在Hadoop 2.5.0、Hadoop 2.6.0和Hive 1.2.0测试环境中写入正常。

**说明：** HDFS Reader暂不支持单个File支持多线程并发读取，这里涉及到单个File内部切分算法。

## 支持的数据类型 {#section_mtj_dck_p2b .section}

由于这些文件表的元数据信息由Hive维护，并存放在Hive自己维护的元数据库（如MySQL）中。目前HDFS Reader不支持对Hive元数据的数据库进行访问查询，因此您在进行类型转换时，必须指定数据类型。

RCFile、ParquetFile、ORCFile、TextFile和SequenceFile中的类型，会默认转为数据集成支持的内部类型，如下表所示。

|类型分类|数据集成column配置类型|Hive数据类型|
|----|--------------|--------|
|整数类|long|tinyint、smallint、int和bigint|
|浮点类|double|float和double|
|字符串类|string|string、char、varchar、struct、map、array、union和binary|
|日期时间类|date|date和timestamp|
|布尔类|boolean|boolean|

说明如下：

-   long：HDFS文件中的整型类型数据，例如123456789。
-   double：HDFS文件中的浮点类型数据，例如3.1415。
-   bool：HDFS文件中的布尔类型数据，例如true、false，不区分大小写。
-   date：HDFS文件中的时间类型数据，例如2014-12-31 00:00:00。

**说明：** Hive支持的数据类型TIMESTAMP可以精确到纳秒级别，所以TextFile、ORCFile中TIMESTAMP存放的数据类似于`2015-08-21 22:40:47.397898389`。如果转换的类型配置为数据集成的DATE，转换之后会导致纳秒部分丢失。所以如果需要保留纳秒部分的数据，请配置转换类型为数据集成的字符串类型。

## 参数说明 {#section_jn2_gqh_p2b .section}

|参数|描述|必选|默认值|
|:-|:-|:-|:--|
|path|要读取的文件路径，如果要读取多个文件，可以使用简单正则表达式匹配，例如/hadoop/data\_201704\*。 -   当指定单个HDFS文件时，HDFS Reader暂时只能使用单线程进行数据抽取。
-   当指定多个HDFS文件时，HDFS Reader支持使用多线程进行数据抽取，线程并发数通过作业速度mbps指定。

**说明：** 实际启动的并发数是您的HDFS待读取文件数量和您配置作业速度两者中的小者。

-   当指定通配符，HDFS Reader尝试遍历出多个文件信息。例如指定/代表读取/目录下所有的文件，指定/bazhen/代表读取bazhen目录下游所有的文件。HDFS Reader目前只支持\*和?作为文件通配符，语法类似于通常的Linux命令行文件通配符。

 **说明：** 

-   数据集成会将一个同步作业所有待读取文件视作同一张数据表。您必须自己保证所有的File能够适配同一套schema信息，并且提供给数据集成权限可读。
-   **注意分区读取：** Hive在建表时，可以指定分区 （partition），例如创建分区partition\(day="20150820", hour="09"\)，对应的HDFS文件系统中，相应的表的目录下则会多出/20150820和/09两个目录且/20150820是/09的父目录。

分区会列成相应的目录结构，在按照某个分区读取某个表所有数据时，则只需配置好JSON中path的值即可。例如需要读取表名叫mytable01下分区day为20150820这一天的所有数据，则配置如下：

    ``` {#codeblock_pdm_roe_ozv}
"path": "/user/hive/warehouse/mytable01/20150820/*"
    ```


 |是|无|
|defaultFS|Hadoop HDFS文件系统namenode节点地址。默认资源组不支持Hadoop高级参数HA的配置，请新增自定义资源，详情请参见[新增任务资源](intl.zh-CN/使用指南/数据集成/常见配置/新增任务资源.md#)。|是|无|
|fileType|文件的类型，目前只支持您配置为text、orc、rc、seq、csv和parquet。HDFS Reader能够自动识别文件的类型，并使用对应文件类型的读取策略。HDFS Reader在做数据同步前，会检查您配置的路径下所有需要同步的文件格式是否和fileType一致，如果不一致任务会失败。 fileType可以配置的参数值列表如下所示。

-   text：表示TextFile文件格式。
-   orc：表示ORCFile文件格式。
-   rc：表示rcfile文件格式。
-   seq：表示sequence file文件格式。
-   csv：表示普通HDFS文件格式（逻辑二维表）。
-   parquet：表示普通parquet file文件格式。

**说明：** 由于TextFile和ORCFile是两种不同的文件格式，所以HDFS Reader对这两种文件的解析方式也存在差异，这种差异导致Hive支持的复杂复合类型（例如map、array、struct和union）在转换为数据集成支持的String类型时，转换的结果格式略有差异，以map类型为例。

-   ORCFile map类型经HDFS eader 析转换成数据集成支持的 Sring类型后，结果为\{job=80, team=60, person=70\}。
-   TextFile map类型经HDFS Reader解析转换成数据集成支持的 string 类型后，结果为job:80, team:60, person:70。

从上面的转换结果可以看出，数据本身没有变化，但是表示的格式略有差异，所以如果您配置的文件路径中要同步的字段在Hive中是复合类型的话，建议配置统一的文件格式 。

 **最佳实践建议：** 

-   如果需要统一复合类型解析出来的格式，建议您在Hive客户端将TextFile格式的表导成ORCFile格式的表。
-   如果是Parquet文件格式，后面的parquetSchema则必填，此属性用来说明要读取的Parquet格式文件的格式。

 对于您指定的column信息，type必须填写，index/value必须选择其一。

 |是|无|
|column|读取字段列表，type指定源数据的类型，index指定当前列来自于文本第几列（以0开始），value指定当前类型为常量。不从源头文件读取数据，而是根据value值自动生成对应的列。默认情况下，您可以全部按照STRING类型读取数据，配置为`"column": ["*"]`。 您也可以指定column字段信息（文件数据列和常量列配置二选一），配置如下：

``` {#codeblock_y1s_nwt_4kd}
{
  "type": "long",
  "index": 0
  //从本地文件文本第一列（下标索引从0开始计数）获取INT字段，index表示从数据文件中获取列数据。
},
{
  "type": "string",
  "value": "alibaba"
  //HDFS Reader内部生成alibaba的字符串字段作为当前字段，value表示常量列。
}
```

 **说明：** 建议您指定待读取的每一列数据的下标和类型，避免配置column \*通配符。

 |是|无|
|fieldDelimiter|读取的字段分隔符，HDFS Reader在读取TextFile数据时，需要指定字段分割符，如果不指定默认为','，HDFS Reader在读取ORCFile时，您无需指定字段分割符，Hive本身的默认分隔符为\\u0001。 -   如果您想将每一行作为目的端的一列，分隔符请使用行内容不存在的字符，例如不可见字符\\u0001。
-   分隔符不能使用\\n。

 |否|,|
|encoding|读取文件的编码配置。|否|utf-8|
|nullFormat|文本文件中无法使用标准字符串定义null（空指针），数据集成提供nullFormat定义哪些字符串可以表示为null。 例如您配置`nullFormat:"null"`，如果源头数据是null，数据集成会将其视作null字段。

 **说明：** 字符串的null（n、u、l、l四个字符）和实际的null不同。

 |否|无|
|compress|当fileType（文件类型）为csv下的文件压缩方式，目前仅支持gzip、bz2、zip、lzo、lzo\_deflate、hadoop-snappy和framing-snappy压缩。 **说明：** 

-   lzo存在lzo和lzo\_deflate两种压缩格式，您在配置时需要留心，不要配错。
-   由于snappy目前没有统一的stream format，数据集成目前仅支持最主流的hadoop-snappy（hadoop上的snappy stream format）和framing-snappy（google建议的snappy stream format）。
-   rc表示rcfile文件格式。
-   orc文件类型下无需填写。

 |否|无|
|parquetSchema| 如果您的文件格式类型为Parquet，在配置column配置项的基础上，您还需配置parquetSchema，具体表示parquet存储的类型说明。您需要确保填写parquetSchem后，整体配置符合JSON语法。parquetScema的配置格式说明如下：

``` {#codeblock_y05_tc5_fmj}
message MessageType名 {
是否必填, 数据类型, 列名;
......................;
}
```

-   MessageType名：填写名称。
-   是否必填：required表示非空，optional表示可为空。推荐全填optional。
-   数据类型：Parquet文件支持BOOLEAN、Int32、Int64、Int96、FLOAT、DOUBLE、BINARY（如果是字符串类型，请填BINARY）和fixed\_len\_byte\_array等类型。
-   每行列设置必须以分号结尾，最后一行也要写上分号。

 配置示例如下所示。

``` {#codeblock_iep_yg4_iif}
"parquetSchema": "message m { optional int32 minute_id; optional int32 dsp_id; optional int32 adx_pid; optional int64 req; optional int64 res; optional int64 suc; optional int64 imp; optional double revenue; }"
```

 |否|无|
|csvReaderConfig|读取CSV类型文件参数配置，Map类型。读取CSV类型文件使用的CsvReader进行读取，会有很多配置，不配置则使用默认值。 常见配置如下所示。

``` {#codeblock_kio_mss_8v3}
"csvReaderConfig":{
  "safetySwitch": false,
  "skipEmptyRecords": false,
  "useTextQualifier": false
}
```

 所有配置项及默认值，配置时csvReaderConfig的map中请严格按照以下字段名字进行配置。

``` {#codeblock_j9p_ohf_0aj}
boolean caseSensitive = true;
char textQualifier = 34;
boolean trimWhitespace = true;
boolean useTextQualifier = true;//是否使用csv转义字符。
char delimiter = 44;//分隔符
char recordDelimiter = 0;
char comment = 35;
boolean useComments = false;
int escapeMode = 1;
boolean safetySwitch = true;//单列长度是否限制100,000字符。
boolean skipEmptyRecords = true;//是否跳过空行。
boolean captureRawRecord = true;
```

 |否|无|
|hadoopConfig|hadoopConfig中可以配置与Hadoop相关的一些高级参数，例如HA的配置。默认资源组不支持Hadoop高级参数HA的配置，请新增自定义资源，详情请参见[新增任务资源](intl.zh-CN/使用指南/数据集成/常见配置/新增任务资源.md#)。 ``` {#codeblock_43q_j23_3pt}
"hadoopConfig":{
"dfs.nameservices": "testDfs",
"dfs.ha.namenodes.testDfs": "namenode1,namenode2",
"dfs.namenode.rpc-address.youkuDfs.namenode1": "",
"dfs.namenode.rpc-address.youkuDfs.namenode2": "",
"dfs.client.failover.proxy.provider.testDfs": "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider"
}
```

 |否|无|
|haveKerberos|是否有Kerberos认证，默认为false。例如用户配置为true，则配置项kerberosKeytabFilePath和kerberosPrincipal为必填。|否|false|
|kerberosKeytabFilePath|Kerberos认证keytab文件的绝对路径。如果haveKerberos为true，则必选。|否|无|
|kerberosPrincipal|Kerberos认证Principal名，如\*\*\*\*/hadoopclient@\*\*.\*\*\* 。如果haveKerberos为true，则必选。 **说明：** 由于Kerberos需要配置keytab认证文件的绝对路径，您需要在自定义资源组上使用此功能。配置示例如下：

``` {#codeblock_dyb_crb_ycv}
"haveKerberos":true,
"kerberosKeytabFilePath":"/opt/datax/**.keytab",
"kerberosPrincipal":"**/hadoopclient@**.**"
```

 |否|无|

## 向导开发介绍 {#section_bp2_wsh_p2b .section}

暂不支持向导开发模式开发。

## 脚本开发介绍 {#section_cp2_wsh_p2b .section}

配置一个从HDFS抽取数据到本地的作业，详情请参见上述参数说明。

``` {#codeblock_s47_9vb_pd0}
{
    "type": "job",
    "version": "2.0",
    "steps": [
        {
            "stepType": "hdfs",//插件名
            "parameter": {
                "path": "",//要读取的文件路径
                "datasource": "",//数据源
                "column": [
                    {
                        "index": 0,//序列号
                        "type": "string"//字段类型
                    },
                    {
                        "index": 1,
                        "type": "long"
                    },
                    {
                        "index": 2,
                        "type": "double"
                    },
                    {
                        "index": 3,
                        "type": "boolean"
                    },
                    {
                        "format": "yyyy-MM-dd HH:mm:ss", //日期格式
                        "index": 4,
                        "type": "date"
                    }
                ],
                "fieldDelimiter": ","//列分隔符
                "encoding": "UTF-8",//编码格式
                "fileType": ""//文本类型
            },
            "name": "Reader",
            "category": "reader"
        },
        { //下面是关于Writer的模板，您可以查找相应的写插件文档。
            "stepType": "stream",
            "parameter": {},
            "name": "Writer",
            "category": "writer"
        }
    ],
    "setting": {
        "errorLimit": {
            "record": ""//错误记录数
        },
        "speed": {
            "concurrent": 3,//作业并发数
            "throttle": false,//false代表不限流，下面的限流的速度不生效，true代表限流。
            "dmu": 1//DMU值
        }
    },
    "order": {
        "hops": [
            {
                "from": "Reader",
                "to": "Writer"
            }
        ]
    }
}
```

parquetSchema的HDFS Reader配置样例如下。

**说明：** 

-   fileType配置项必须设置为parquet。
-   如果您要读取parquet文件中的部分列，需在parquetSchema配置项中，指定完整schema结构信息，并在column中根据下标，筛选需要的同步列进行列映射。

``` {#codeblock_c3c_m3x_ce3}
"reader":  {
    "name": "hdfsreader",
    "parameter": {
        "path": "/user/hive/warehouse/addata.db/dw_ads_rtb_monitor_minute/thedate=20170103/hour_id=22/*",
        "defaultFS": "h10s010.07100.149:8020",
        "column": [
            {
                "index": 0,
                "type": "string"
            },
            {
                "index": 1,
                "type": "long"
            },
            {
                "index": 2,
                "type": "double"
            }
        ],
        "fileType": "parquet",
        "encoding": "UTF-8",
        "parquetSchema": "message m { optional int32 minute_id; optional int32 dsp_id; optional int32 adx_pid; optional int64 req; optional int64 res; optional int64 suc; optional int64 imp; optional double revenue; }"
    }
}
```

