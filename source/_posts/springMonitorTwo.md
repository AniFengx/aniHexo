---
title: spring监控平台从零单排【二】
date: 2023-04-12 15:40:39
tags: [java,spring,monitor,Admin]
categories: [monitor]
---

第一章介绍了**Spring Boot Admin**的使用方法，但是其UI平台展示的内容离实际所需的监控内容还有较大差距，本章着重介绍其底层所使用的**Actuator**框架，并提供如何对接到**Prometheus**和**grafana**的方式

### Spring Boot Actuator

**Actuator**默认暴露了一些jvm信息，也可以自行借助打点方法记录需要的监控信息，其底层是使用的**Micrometer**

<!-- more -->

#### 集成

**Actuator**是Client模块用来暴露信息的框架，监控平台需要自行搭建，本次直接使用**grafana**制作好的看板进行展示

##### springboot集成

1. Client模块修改pom文件，添加如下内容

    ``` xml
    <!--  actuator  -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <!--  将actuator暴露的接口改造为prometheus支持的格式  -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    ```

2. 修改`application.yml`文件，添加如下内容

    ``` yaml
    management:
        endpoints:
            web:
                exposure:
                    include: 'prometheus'
        metrics:
            tags:
                #定义暴露指标的tag名称，如下设置是默认取用当前springboot应用名，也可写死
                application: ${spring.application.name}
    ```

以上步骤添加后，即完成了**Actuator**集成，可以运行当前应用，访问应用接口`http://localhost:8080/actuator/prometheus`可获得如下内容

``` java
# HELP jvm_buffer_count_buffers An estimate of the number of buffers in the pool
# TYPE jvm_buffer_count_buffers gauge
jvm_buffer_count_buffers{application="web",id="mapped",} 0.0
jvm_buffer_count_buffers{application="web",id="direct",} 3.0
# HELP jvm_buffer_total_capacity_bytes An estimate of the total capacity of the buffers in this pool
# TYPE jvm_buffer_total_capacity_bytes gauge
jvm_buffer_total_capacity_bytes{application="web",id="mapped",} 0.0
jvm_buffer_total_capacity_bytes{application="web",id="direct",} 8208.0
# HELP tomcat_sessions_created_sessions_total  
# TYPE tomcat_sessions_created_sessions_total counter
tomcat_sessions_created_sessions_total{application="web",} 0.0
# HELP tomcat_sessions_rejected_sessions_total  
# TYPE tomcat_sessions_rejected_sessions_total counter
tomcat_sessions_rejected_sessions_total{application="web",} 0.0
# HELP jvm_classes_loaded_classes The number of classes that are currently loaded in the Java virtual machine
# TYPE jvm_classes_loaded_classes gauge
jvm_classes_loaded_classes{application="web",} 12286.0
# HELP jvm_classes_unloaded_classes_total The total number of classes unloaded since the Java virtual machine has started execution
# TYPE jvm_classes_unloaded_classes_total counter
jvm_classes_unloaded_classes_total{application="web",} 56.0
# HELP jvm_gc_live_data_size_bytes Size of long-lived heap memory pool after reclamation
# TYPE jvm_gc_live_data_size_bytes gauge
jvm_gc_live_data_size_bytes{application="web",} 3.837896E7
# HELP jvm_threads_daemon_threads The current number of live daemon threads
# TYPE jvm_threads_daemon_threads gauge
jvm_threads_daemon_threads{application="web",} 27.0
# HELP process_cpu_usage The "recent cpu usage" for the Java Virtual Machine process
# TYPE process_cpu_usage gauge
process_cpu_usage{application="web",} 0.08109822305359074
# HELP jvm_memory_committed_bytes The amount of memory in bytes that is committed for the Java virtual machine to use
# TYPE jvm_memory_committed_bytes gauge
jvm_memory_committed_bytes{application="web",area="heap",id="PS Eden Space",} 3.13524224E8
jvm_memory_committed_bytes{application="web",area="nonheap",id="Code Cache",} 1.0289152E7
jvm_memory_committed_bytes{application="web",area="nonheap",id="Metaspace",} 6.8550656E7
jvm_memory_committed_bytes{application="web",area="heap",id="PS Old Gen",} 4.59800576E8
jvm_memory_committed_bytes{application="web",area="heap",id="PS Survivor Space",} 1.572864E7
jvm_memory_committed_bytes{application="web",area="nonheap",id="Compressed Class Space",} 9306112.0
# HELP process_start_time_seconds Start time of the process since unix epoch.
# TYPE process_start_time_seconds gauge
process_start_time_seconds{application="web",} 1.6812881222E9
# HELP jvm_gc_memory_allocated_bytes_total Incremented for an increase in the size of the (young) heap memory pool after one GC to before the next
# TYPE jvm_gc_memory_allocated_bytes_total counter
jvm_gc_memory_allocated_bytes_total{application="web",} 4.46238072E8
# HELP jvm_gc_memory_promoted_bytes_total Count of positive increases in the size of the old generation memory pool before GC to after GC
# TYPE jvm_gc_memory_promoted_bytes_total counter
jvm_gc_memory_promoted_bytes_total{application="web",} 2.1952392E7
# HELP process_uptime_seconds The uptime of the Java virtual machine
# TYPE process_uptime_seconds gauge
process_uptime_seconds{application="web",} 19.083
# HELP tomcat_sessions_alive_max_seconds  
# TYPE tomcat_sessions_alive_max_seconds gauge
tomcat_sessions_alive_max_seconds{application="web",} 0.0
# HELP jvm_threads_peak_threads The peak live thread count since the Java virtual machine started or peak was reset
# TYPE jvm_threads_peak_threads gauge
jvm_threads_peak_threads{application="web",} 64.0
# HELP tomcat_sessions_active_max_sessions  
# TYPE tomcat_sessions_active_max_sessions gauge
tomcat_sessions_active_max_sessions{application="web",} 0.0
# HELP system_cpu_count The number of processors available to the Java virtual machine
# TYPE system_cpu_count gauge
system_cpu_count{application="web",} 12.0
# HELP jvm_gc_pause_seconds Time spent in GC pause
# TYPE jvm_gc_pause_seconds summary
jvm_gc_pause_seconds_count{action="end of minor GC",application="web",cause="Allocation Failure",} 1.0
jvm_gc_pause_seconds_sum{action="end of minor GC",application="web",cause="Allocation Failure",} 0.01
jvm_gc_pause_seconds_count{action="end of minor GC",application="web",cause="Metadata GC Threshold",} 1.0
jvm_gc_pause_seconds_sum{action="end of minor GC",application="web",cause="Metadata GC Threshold",} 0.012
jvm_gc_pause_seconds_count{action="end of major GC",application="web",cause="Metadata GC Threshold",} 1.0
jvm_gc_pause_seconds_sum{action="end of major GC",application="web",cause="Metadata GC Threshold",} 0.112
# HELP jvm_gc_pause_seconds_max Time spent in GC pause
# TYPE jvm_gc_pause_seconds_max gauge
jvm_gc_pause_seconds_max{action="end of minor GC",application="web",cause="Allocation Failure",} 0.01
jvm_gc_pause_seconds_max{action="end of minor GC",application="web",cause="Metadata GC Threshold",} 0.012
jvm_gc_pause_seconds_max{action="end of major GC",application="web",cause="Metadata GC Threshold",} 0.112
# HELP tomcat_sessions_active_current_sessions  
# TYPE tomcat_sessions_active_current_sessions gauge
tomcat_sessions_active_current_sessions{application="web",} 0.0
# HELP jvm_gc_max_data_size_bytes Max size of long-lived heap memory pool
# TYPE jvm_gc_max_data_size_bytes gauge
jvm_gc_max_data_size_bytes{application="web",} 5.672271872E9
# HELP jvm_threads_states_threads The current number of threads having NEW state
# TYPE jvm_threads_states_threads gauge
jvm_threads_states_threads{application="web",state="timed-waiting",} 13.0
jvm_threads_states_threads{application="web",state="new",} 0.0
jvm_threads_states_threads{application="web",state="terminated",} 0.0
jvm_threads_states_threads{application="web",state="blocked",} 0.0
jvm_threads_states_threads{application="web",state="waiting",} 7.0
jvm_threads_states_threads{application="web",state="runnable",} 44.0
# HELP jvm_buffer_memory_used_bytes An estimate of the memory that the Java virtual machine is using for this buffer pool
# TYPE jvm_buffer_memory_used_bytes gauge
jvm_buffer_memory_used_bytes{application="web",id="mapped",} 0.0
jvm_buffer_memory_used_bytes{application="web",id="direct",} 8209.0
# HELP jvm_memory_used_bytes The amount of used memory
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{application="web",area="heap",id="PS Eden Space",} 1.65650568E8
jvm_memory_used_bytes{application="web",area="nonheap",id="Code Cache",} 1.0112832E7
jvm_memory_used_bytes{application="web",area="nonheap",id="Metaspace",} 6.4346336E7
jvm_memory_used_bytes{application="web",area="heap",id="PS Old Gen",} 3.837896E7
jvm_memory_used_bytes{application="web",area="heap",id="PS Survivor Space",} 0.0
jvm_memory_used_bytes{application="web",area="nonheap",id="Compressed Class Space",} 8478672.0
# HELP jvm_threads_live_threads The current number of live threads including both daemon and non-daemon threads
# TYPE jvm_threads_live_threads gauge
jvm_threads_live_threads{application="web",} 64.0
# HELP tomcat_sessions_expired_sessions_total  
# TYPE tomcat_sessions_expired_sessions_total counter
tomcat_sessions_expired_sessions_total{application="web",} 0.0
# HELP system_cpu_usage The "recent cpu usage" for the whole system
# TYPE system_cpu_usage gauge
system_cpu_usage{application="web",} 0.028650178842763663
# HELP jvm_memory_max_bytes The maximum amount of memory in bytes that can be used for memory management
# TYPE jvm_memory_max_bytes gauge
jvm_memory_max_bytes{application="web",area="heap",id="PS Eden Space",} 2.796552192E9
jvm_memory_max_bytes{application="web",area="nonheap",id="Code Cache",} 2.5165824E8
jvm_memory_max_bytes{application="web",area="nonheap",id="Metaspace",} -1.0
jvm_memory_max_bytes{application="web",area="heap",id="PS Old Gen",} 5.672271872E9
jvm_memory_max_bytes{application="web",area="heap",id="PS Survivor Space",} 1.572864E7
jvm_memory_max_bytes{application="web",area="nonheap",id="Compressed Class Space",} 1.073741824E9
# HELP logback_events_total Number of error level events that made it to the logs
# TYPE logback_events_total counter
logback_events_total{application="web",level="info",} 14.0
logback_events_total{application="web",level="debug",} 0.0
logback_events_total{application="web",level="error",} 0.0
logback_events_total{application="web",level="warn",} 0.0
logback_events_total{application="web",level="trace",} 0.0
```

以上信息都是遵守**prometheus**规范生成的返回信息

##### 数据导入到prometheus

{% note %} 安装方式参考[官网安装说明](https://prometheus.io/docs/introduction/first_steps/#starting-prometheus "官网安装说明")，安装最新版本即可 {% endnote %}

修改配置文件`prometheus.yml`，添加如下内容

``` yaml
scrape_configs:
    # 一个scrape_configs下可以有多个job_name，任意写，建议英文，不要包含特殊字符
    - job_name: 'spring'
        # 多久采集一次数据
        scrape_interval: 3s
        # 采集时的超时时间，注意：超时时间需要小于采集时间
        scrape_timeout: 2s
        # 采集的路径是啥
        metrics_path: 'actuator/prometheus'
        # 采集服务的地址，设置成上面Spring Boot应用所在服务器的具体地址。
        static_configs:
            - targets: ['127.0.0.1:8080']
```

如上配置后prometheus即可每3s请求一次Client

启动prometheus，在prometheus根目录下执行如下命令

``` shell
./prometheus --config.file=prometheus.yml
```

访问`http://localhost:9090`，可查看prometheus采集的指标如下

![avatar](1.png)

##### 通过grafana进行展示

{% note %} 安装方式参考[官网安装说明](https://grafana.com/docs/grafana/latest/setup-grafana/installation/ "官网安装说明")，安装最新版本即可 {% endnote %}

{% note warning %} **grafana**如果通过installer方式安装，默认是电脑重启后自动启动的，这个可以根据情况手动关闭自动启动服务 {% endnote %}

启动grafana，进入安装后文件夹下`bin`中，执行如下命令

``` shell
./grafana-server
```

登录`http://localhost:3000/login`，初始账号/密码为`admin/admin`，进入页面后需要配置一些信息

1. 配置数据源，界面如下

    ![avatar](2.png)

    选择`prometheus`后，填入其服务地址`http://localhost:9090`，点击`Save & test`，成功添加数据源

2. 创建看板，界面如下
    
    ![avatar](3.png)

    网上针对jvm相关指标已有成熟看板可供使用，填入`id`为`4701`即可，并且数据源选择刚刚添加的数据源，然后点击`import`导入

    ![avatar](4.png)

到此为止即可借助**Actuator**提供的默认指标监控jvm的运行情况，效果如下

![avatar](5.png)

{% note %} **grafana**有专门的看板仓库，其中积累了大量线程的各类指标看板，可自行选择，点击前往[Grafana Lab - Dashboards](https://grafana.com/grafana/dashboards/?pg=hp&plcmt=lt-box-dashboards "Dashboards") {% endnote %}