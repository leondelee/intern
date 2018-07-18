# <center>使用Promethus + Grafana + Mysql完成对Ubuntu各项性能实时数据的监控</center>
## 目录
<!-- TOC --> 

- [使用Promethus + Grafana + Mysql完成对Ubuntu各项性能实时数据的监控](#center使用promethus--grafana--mysql完成对ubuntu各项性能实时数据的监控center)
    - [1.数据介绍](#1数据介绍)
    - [2.工具介绍](#2工具介绍)
        - [2.1 Prometheus](#21-prometheus)
            - [2.1.1 简介](#211-简介)
            - [2.1.2 架构](#212-架构)
            - [2.1.3 概念](#213-概念)
                - [数据模型](#数据模型)
                - [指标(Metrics)](#指标metrics)
                - [实例(Instannces)和任务(Jobs)](#实例instannces和任务jobs)
            - [2.1.4 安装](#214-安装)
                - [配置go环境](#配置go环境)
                - [使用预编译的二进制文件](#使用预编译的二进制文件)
            - [2.1.5 配置文件](#215-配置文件)
            - [2.1.6 运行](#216-运行)
            - [2.1.7 与Mysql结合](#217-与mysql结合)
            - [2.1.8 报警机制](#218-报警机制)
        - [2.2 Grafana](#22-grafana)
            - [2.2.1 安装](#221-安装)
                - [Install Stable](#install-stable)
                - [添加APT源](#添加apt源)
                - [更新并安装:](#更新并安装)
                - [启动Grafana服务](#启动grafana服务)
            - [2.2.2 创建DataSource](#222-创建datasource)
            - [2.2.3 创建Dashboard](#223-创建dashboard)
<!-- /TOC -->
## 概述
这篇文章将以Mysql数据库为例,介绍Prometheus+Grafana的具体使用方法.其中提到的所有指令均在Ubuntu16.04运行.
## 1.数据介绍
在这篇文章当中,我将把glances(一种linux系统性能监控工具)采集到的实时数据导入Mysql数据库当中,继而介绍Prometheus,Grafana以及Mysql三者结合起来的用法.
<br>首先打开Glances(没有的朋友可以下载安装,这里不再赘述),并把数据上传到Web端:

    glances -w
然后可以看到下面这样的输出:

    Glances web server started on http://0.0.0.0:61208/
这说明我们可以通过在浏览器访问 http://0.0.0.0:61208/ 这个地址来查看结果.结果如下图:

![glances](https://github.com/leondelee/photos/blob/master/glances_web.png)

通过上述图片,我们可以看到,Glances检测并展示了我电脑的几乎所有常见的性能数据,比如CPU使用率是4.2%,内存使用率是29.9%,CPU温度是55摄氏度等.

而后,我使用PHP脚本抓取http://0.0.0.0:61208/ 这个地址当中的数据,并选取系统当前时间,系统版本,总内存,可用内存,已使用内存,内存使用率,CPU使用率,CPU温度这8个数据录入到Mysql数据库当中.至于这个过程的具体步骤,不是这篇文章的重点,有兴趣的朋友可以到我的另一篇文章当中看[Glances数据导入Mysql](https://github.com/leondelee/intern/blob/master/mysql.md).最终Mysql数据库长这个样子:

![Mysql](https://github.com/leondelee/photos/blob/master/monitor.png)

## 2.工具介绍
### 2.1 Prometheus
#### 2.1.1 简介
Prometheus 是由 SoundCloud 开发的开源监控报警系统和时序列数据库(TSDB).它具有以下几个特点:

  - 灵活而强大的查询语句（PromQL）：在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作.
  - 易于管理： Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
  - 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
  - 使用 pull 模式采集时间序列数据，这样不仅有利于本机测试而且可以避免有问题的服务器推送坏的 metrics。
  - 通过基于HTTP的pull方式采集时序数据

  但是需要注意的是,Prometheus在采集数据的过程当中可能出现错误.因此,如果需要百分之百的数据准确的话,需要慎用.
#### 2.1.2 架构
Prometheus的架构如下图:

![prometheus_arch](https://github.com/leondelee/photos/blob/master/prometheus_arch.png)

从上图可以看出，Prometheus 的主要模块包括：Prometheus server, exporters, Pushgateway, PromQL, Alertmanager 以及图形界面。其大概的工作流程是：

  1. Prometheus server 定期从配置好的 jobs 或者 exporters 中拉 metrics，或者接收来自   Pushgateway 发过来的 metrics，或者从其他的 Prometheus server 中拉 metrics。

  2. Prometheus server 在本地存储收集到的 metrics，并运行已定义好的 alert.rules，记录新的时间序列或者向 Alertmanager 推送警报。
  3. Alertmanager 根据配置文件，对接收到的警报进行处理，发出告警。
  4. 在图形界面中，可视化采集数据。(我们使用Grafana完成可视化界面)

#### 2.1.3 概念
在这里主要介绍几个概念:数据模型,指标(metric),实例(Instance),任务(jobs):
##### 数据模型

Prometheus 中存储的数据为时间序列，是由 metric 的名字和一系列的标签（键值对）唯一标识的，不同的标签则代表不同的时间序列。

  - metric 名字：该名字应该具有语义，一般用于表示 metric 的功能，例如：http_requests_total, 表示 http 请求的总数。其中，metric 名字由 ASCII 字符，数字，下划线，以及冒号组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
  - 标签：使同一个时间序列有了不同维度的识别。例如 http_requests_total{method="Get"} 表示所有 http 请求中的 Get 请求。当 method="post" 时，则为新的一个 metric。标签中的键由 ASCII 字符，数字，以及下划线组成，且必须满足正则表达式 [a-zA-Z_:][a-zA-Z0-9_:]*。
  - 样本：实际的时间序列，每个序列包括一个 float64 的值和一个毫秒级的时间戳。
  - 格式：<metric name>{<label name>=<label value>, …}，例如：http_requests_total{method="POST",endpoint="/api/tracks"}。

##### 指标(Metrics)
Prometheus 客户端库主要提供四种主要的 metric 类型：

  - Counter:一种累加的 metric，典型的应用如：请求的个数，结束的任务数， 出现的错误数等等。例如，查询 http_requests_total{method="get", job="Prometheus", handler="query"} 返回 8，10 秒后，再次查询，则返回 14。
  - Gauge:一种常规的 metric，典型的应用如：温度，运行的 goroutines 的个数。可以任意加减。
  - Histogram:可以理解为柱状图，典型的应用如：请求持续时间，响应大小。可以对观察结果采样，分组及统计。例如，查询 http_request_duration_microseconds_sum{job="Prometheus", handler="query"} 时，返回结果如下：
  
![example1](https://github.com/leondelee/photos/blob/master/histogram.png)

  - Summary:类似于 Histogram, 典型的应用如：请求持续时间，响应大小。提供观测值的 count 和 sum 功能。提供百分位的功能，即可以按百分比划分跟踪结果。

##### 实例(Instannces)和任务(Jobs)
  
  - Instances:一个单独 scrape 的目标， 一般对应于一个进程。
  - Jobs:一组同种类型的 instances（主要用于保证可扩展性和可靠性）

 job 和 instance 的关系:

    job: api-server
 
        instance 1: 1.2.3.4:5670
        instance 2: 1.2.3.4:5671
        instance 3: 5.6.7.8:5670
        instance 4: 5.6.7.8:5671

下面以实际的 metric 为例，对上述概念进行说明:

![example](https://github.com/leondelee/photos/blob/master/example1.png)

如上图所示，这三个 metric 的名字都一样，他们仅凭 handler 不同而被标识为不同的 metrics。这类 metrics 只会向上累加，是属于 Counter 类型的 metric，且 metrics 中都含有 instance 和 job 这两个标签。

#### 2.1.4 安装

##### 配置go环境
首先需要在电脑上安装go语言,具体的可参考 https://golang.org/doc/install

##### 使用预编译的二进制文件
在Prometheus官网[下载](https://prometheus.io/download/)压缩文件,然后进入压缩文件所在的目录,在命令行输入:

    tar xvfz prometheus-*.tar.gz

进行解压缩.
<br>随后进入该目录:

    cd prometheus-*

#### 2.1.5 配置文件

在使用Prometheus之前,还需要配置一个文件,这个文件的名字是prometheus.yml,它就在我们刚刚进入的这个文件当中.使用任何一种编辑器打开这个文件,可以看到类似的配置(官网下载下来的时候是英文版本的,我稍微改编翻译了一下):

    global:
        scrape_interval:     15s # 每15秒抓取一次数据

        scrape_configs:
        # job_name定义了任务名字,在这里我们将完成Prometheus对自己的监控
        - job_name: 'prometheus'

            # 覆盖了原先的全局设置,进行一个局部设置,也就是每5秒抓取一次
            scrape_interval: 5s
            # 默认端口是9090
            static_configs:
            - targets: ['localhost:9090'] #访问这个地址来查看Prometheus的运行情况
保存以上编辑就可以了.
如果想要了解更为详细的配置信息,可以查看[官方文档的相关部分](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

#### 2.1.6 运行
配置完成以后,在这个当前文件路径下运行以下指令:

    ./prometheus --config.file=prometheus.yml

然后如果看到输出:

![prometheus_default_start](https://github.com/leondelee/photos/blob/master/prometheus_default_start.png)

就表明Prometheus安装并配置成功.
<br>接下来就可以通过浏览器访问 http://localhost:9090 来查看运行情况.
访问这个地址,我们可以看到:

![prometheus_default_ui](https://github.com/leondelee/photos/blob/master/prometheus_default_ui.png)

在这个界面上,我们可以看到:一个搜索框.这个搜索框的作用是:我们可以在其中输入我们需要的metric的表达式,点击下方的'Execute',就可以找到我们需要的metric.除此之外,如果我们在浏览器访问 http://localhost:9090/metrics ,那么就可以看到Prometheus采集到的所有metrics,如下图:

  ![prometheus_all_metrics](https://github.com/leondelee/photos/blob/master/prometheus_all_metrics.png)

至此,我们已经实现了Prometheus对自己的监控任务.

#### 2.1.7 与Mysql结合

接下来,我将实现Prmetheus对一开始提到的MySql数据的采集和监控.我在这里使用一个名叫prometheus-mysql-exporter的python3框架,它不是官方提供的框架,而是PyPI上的一个开源小项目.使用这个框架,我们可以自定义我们需要采集的数据,也就是说,我们可以选择Mysql数据库当中的任何一个表当中的任何一种或者多种数值数据进行采集和监控,比如我将会采集内存,可用内存,已使用内存,内存使用率,CPU使用率,CPU温度这几个数据.
<br>promethesu-mysql-exporter是PyPI上面的一个开源项目,项目地址是 https://pypi.org/project/prometheus-mysql-exporter/ ,使用之前你需要保证你的mysql-server是打开的,接下来的过程都默认mysql-server的地址是:localhost:3306.具体使用方法如下:

  - 环境配置:需要python3,pip3,以及libmysqlclient-dev.这三者都可以使用apt-get直接安装.
  - 安装:环境配置好以后,在命令行输入:

        pip install prometheus-mysql-exporter
  
  - 配置文件:在当前文件夹创建一个名为mycnf.cfg的文件,这是配置文件,用来明确如何采集mysql当中的数据.大家可以按照我下面的方式进行配置:

        [query_foo]
        QueryIntervalSecs = 5 
        QueryStatement = SELECT column1 FROM testdb.testtb; 
        QueryValueColumns = column1

    简单说明一下以上配置:
      - 这个文件的section名字[query_foo]表明这个query的名字是foo,之后我们采集到的metric的开头都是foo
      - 第二行定义了两次数据库query的间隔时间,这里是5秒
      - 第三行是mysql query语句,指的是从testdb这个数据库的testtb这个表当中选取column1这一列数据,column1是这列数据的名字.需要注意的是,这列数据必须是纯数值数据,而不能是字符串.
      - 第四行是把column1这一列的数据作为metric暴露给prometheus.
    
    回到我们之前提到的Glances数据这个例子,我想要让prometheus采集总内存,可用内存,已使用内存,内存使用率,CPU使用率,CPU温度这几个纯数值数据,那么我的配置文件应该这么写:

        [query_foo]
        QueryIntervalSecs = 5
        QueryStatement = SELECT Mem_total,Mem_avail,Mem_used,Mem_per,Cpu_core,Cpu_per,Cpu_temp FROM ubuntu_monitor.monitor;
        QueryValueColumns = Mem_total,Mem_avail,Mem_used,Mem_per,Cpu_core,Cpu_per,Cpu_temp
    
  - 运行: 进入该配置文件所在的文件目录,在命令行输入:

        prometheus-mysql-exporter -p 8080 -s localhost:3306 -u 登陆mysql服务器的用户名 -P 你的mysql服务器密码 -c mycnf.cfg -d 你的数据库名字
    上面这个语句的具体语法,可以参看官方文档说明.
    <br>通过运行上述指令,这个exporter将在8080端口运行,并进入你的指定数据库.正常情况下会得到下面这个输出:

          [2018-07-18 10:09:15,981] root.INFO MainThread Starting server...
          [2018-07-18 10:09:15,983] root.INFO MainThread Server started on port 8080

到这里,我们就算完成对指定metric的暴露工作.
完成上述暴露工作以后,我们还需要在之前提到的prometheus的配置文件当中修改配置,也就是建立一个新的job,不然prometheus并不知道需要采集这些信息.就好像我们现在是快递公司把快件送到了地方,但如果我们不联系买家,那么他们并不知道要来收快递.
将原来的配置文件改成下面这个样子:

    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        scrape_interval: 5s
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'mysql'
        scrape_interval: 5s
        static_configs:
          - targets: ['localhost:8080']
            labels:
              instance: db1
然后在命令行重新输入:

    ./prometheus --config.file=prometheus.yml

随后继续用浏览器访问 http://localhost:9090 ,点击上方Status中的targets:

![prometheus_mysql_exporter](https://github.com/leondelee/photos/blob/master/prometheus_mysql_exporter.png)

可以看到:

![prometheus_mysql_exporter_targets](https://github.com/leondelee/photos/blob/master/prometheus_mysql_exporter_targets.png)

这里有两个targets,其中mysql是我们后来添加的.'Up'表明这个target正常运行.
<br>之后我们可以到graph里面验证.点击上访的graph,在搜索框输入foo_column1(在之前mycnf.cfg文件当中配置的),点击execute.在我这个例子当中,我可以输入foo_Mem_per,看到输出如下:

![prometheus_mysql_exporter_output](https://github.com/leondelee/photos/blob/master/prometheus_mysql_exporter_output.png)

'value'的值说明了在这一时刻,我的内存使用率是55.4%.

#### 2.1.8 报警机制
在这里,我将介绍prometheus的报警机制.
<br>prometheus的报警系统包括两部分:报警规则和报警管理器.当发生异常时,系统根据报警规则发送报警信号给报警管理器,然后报警管理器处理这些报警信号,包括silencing, inhibition, aggregation和通过邮件短信等方式发送报警信号.具体的流程如下:

  - 建立并配置报警管理器:
    首先安装alertmanger

        $ GO15VENDOREXPERIMENT=1 go get github.com/prometheus/alertmanager/cmd/...
        # cd $GOPATH/src/github.com/prometheus/alertmanager
    然后编辑alertmanager的配置文件simple.yml,这个文件要和alertmanager这个二进制文件放在同一级文件夹当中,一个简单的版本如下:

            global:
              # The smarthost and SMTP sender used for mail notifications.
              smtp_smarthost: 'localhost:25'
              smtp_from: 'alertmanager@example.org'
              smtp_auth_username: 'alertmanager'
              smtp_auth_password: 'password'

            # The directory from which notification templates are read.
            templates: 
            - '/etc/alertmanager/template/*.tmpl'

            route:
              receiver: 'default-receiver'
              group_wait: 1s #组报警等待时间
              group_interval: 1s  #组报警间隔时间
              repeat_interval: 1s  #重复报警间隔时间
              group_by: [cluster, alertname]
              routes:
              - receiver: test
                group_wait: 1s
                match_re:
                  severity: test

            receivers:      #改成自己的邮箱
            - name: 'default-receiver'
              email_configs:
              - to: 'liliangwei@wecash.net'
                headers: { Subject: " {{ .CommonAnnotations.summary }}" }
            - name: 'test'
              email_configs:
              - to: 'liliangwei@wecash.net'
                headers: { Subject: " {{ 测试}}" }
    更加详细的配置说明请参考 https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml#L87 
    <br>这时,需要运行alertmanager:

        alertmanager --config.file=simple.yml
  - 配置prometheus,使它能够和报警管理器通信:接下来需要配置在一开始提到的prometheus.yml这个文件,需要在原先的基础之上添加几行代码,如下:

        rule_files:
            - rules.yml

        alerting:
          alertmanagers:
            - static_configs:
              - targets: ["localhost:9093"]
    所以,完整的prometheus.yml文件如下:

        global:
          scrape_interval: 15s
        rule_files:
            - rules.yml

        alerting:
          alertmanagers:
            - static_configs:
              - targets: ["localhost:9093"]

          
        scrape_configs:
          - job_name: 'prometheus'
            scrape_interval: 5s
            static_configs:
              - targets: ['localhost:9090']

          - job_name: 'mysql'
            scrape_interval: 5s
            static_configs:
              - targets: ['localhost:8080']
                labels:
                  instance: db1
  - 建立报警规则:最后,需要编写rules.yml文件,这个文件要放在你在刚才prometheus.yml当中指定的路径中,在这里和prometheus.yml在同一级就可以了,一个简单的版本如下:

        groups:
            - name: test-rule
              rules:
              - alert: Mem_per
                expr: foo_Mem_per{db="ubuntu_monitor",instance="db1",job="mysql"}>50
                for: 30s    
                labels:
                  team: node
                annotations:
                  summary: "{{$labels.instance}}: High memory usage detected"
                  description: "{{$labels.instance}}:Memory usage is above 80% (current value is: {{ $value }}"
              - alert: Cpu_temp
                expr: foo_Cpu_temp{db="ubuntu_monitor",instance="db1",job="mysql"}>50
                for: 30s
                labels:
                  team: node
                annotations:
                  summary: "{{$labels.instance}}: High Cpu Temperature detected"
                  description: "{{$labels.instance}}: Cpu temperature is above 45 (current value is: {{ $value }}"
    我在这里设置了当内存占用大于80%或者cpu温度高于45摄氏度时发出警告.

  最后,重新启动prometheus:

    ./prometheus --config.file=prometheus.yml
  重新进入 http://localhost:9090 ,并点击Alerts,我们可以看到异常情况已经被监控出来:

  ![alert](https://github.com/leondelee/photos/blob/master/alert.png)

到这里,就完成了Prometheus和Mysql当中的所有部分,接下来,我们将开始使用Grafana将上面的结果可视化.
### 2.2 Grafana
#### 2.2.1 安装
##### Install Stable

    wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.1.4_amd64.deb
    sudo apt-get install -y adduser libfontconfig
    sudo dpkg -i grafana_5.1.4_amd64.deb

##### 添加APT源
在 /etc/apt/sources.list 中添加一行:

    deb https://packagecloud.io/grafana/stable/debian/ stretch main

##### 更新并安装:

    sudo apt-get update
    sudo apt-get install grafana

##### 启动Grafana服务

    sudo service grafana-server start
grafana启动以后,在浏览器输入 http://localhost:3000 就可以进入grafana的登陆界面,在用户名那里输入:admin,密码也是admin,最后就可以看到:

![grafana](https://github.com/leondelee/photos/blob/master/grafana.png)

这就表明安装成功.

#### 2.2.2 创建DataSource
Datasource是grafana的数据源,顾名思义,就是被可视化的数据的来源.设置方法如下:<br>
进入Grafana界面以后,点击'Create your first datasource',然后会看到如下的界面:

![datasource](https://github.com/leondelee/photos/blob/master/datasource.png)

按照上图的方式填写设置信息,并保存.如果看到Data Source is working的字样,就说明数据源选择成功.

#### 2.2.3 创建Dashboard
Dashboard是grafana的仪表盘,也就是呈现各种数据的地方.配置方法如下:
<br>在创建Datasource的旁边,有一个'create dashboard',点击之后可以看到如下的界面,

![graph](https://github.com/leondelee/photos/blob/master/graph.png)

点击graph,创建一个新的图表:

![newdash](https://github.com/leondelee/photos/blob/master/newdash.png)

这时,我将要用这个图表来展示之前提到的cpu温度,点击'Panel Title',并点击edit来进行相应的设置:
  - 首先修改标题,点击General,把title修改为Cpu温度
  - 接着点击Metrics,按照下图进行设置:
    
    ![metric_set](https://github.com/leondelee/photos/blob/master/metric_set.png)

    注意,这里的foo_Cpu_temp正是我们使用prometheus-mysql-exporter暴露出来的metric
  
  - 然后设置预警值,点击alert,并点击create alert,按照下图设置:

    ![alertrule](https://github.com/leondelee/photos/blob/master/alertrule.png)

  - 保存

最后,可以看到数据绘制成功:

  ![cputemp](https://github.com/leondelee/photos/blob/master/cputemp.png)



