## MLB基本概念

Marathon-LB通过脚本`marathon_lb.py`连接到Marathon的API检索所有正在运行的应用程序，然后生成HAProxy配置并重新加载HAProxy。默认情况下，Marathon-LB绑定到每个应用程序的服务端口，并将传入请求发送给对应的应用程序实例。

应用程序通过在其Marathon程序定义中声明的服务端口（`servicePort`）来提供服务。此外，应用程序仅在具有与Marathon应用程序的标签（使用`HAPROXY_GROUP`）中定义的标记（或组）相同的LB上暴露服务。可以为特定标签标记的应用针对性的调整HAProxy参数。

要创建一个或多个虚拟主机，需要在应用程序的Marathon程序定义中设置HAPROXY\_{n}\_VHOST标签。配置了虚拟主机的应用除了在服务端口（`servicePort`）提供服务外，也通过80和443端口公开服务。可以在HAPROXY\_{n}\_VHOST中使用逗号作为主机域名之间的分隔符来指定多个虚拟主机。

所有应用程序也使用HTTP头`X-Marathon-App-Id`在端口9091上公开。参考[HAProxy配置参考](/dcos-component-marathon-lb-template.md)中的HAPROXY\_HTTP\_FRONTEND\_APPID\_HEAD配置。

可以通过`http://<public-node-ip>:9090/haproxy?stats`获取HAProxy的统计信息。可以通过`http://<public-node-ip>:9090/_haproxy_getconfig`获取当前HAProxy的配置。

### 部署与运行

在选择Marathon-LB的部署方式前，首先了解一下Marathon-LB如何运行。Marathon-LB的入口程序是**marathon-lb.py**。可以选择**直接调用**或通过打包好的**Docker镜像（\*\***[mesosphere\/marathon-lb](https://hub.docker.com/r/mesosphere/marathon-lb)**\*\*）**来启动。通过Marathon-LB容器的[Dockerfile](https://github.com/mesosphere/marathon-lb/blob/master/Dockerfile)定义可以看到，容器启动是通过在[run脚本](https://github.com/mesosphere/marathon-lb/blob/master/run)中调用marathon-lb.py并传递一组参数：

脚本run：

```
......
exec /marathon-lb/marathon_lb.py \ 
    --syslog-socket $SYSLOG_SOCKET \ 
    --haproxy-config /marathon-lb/haproxy.cfg \ 
    --ssl-certs "${SSL_CERTS}" \ 
    --command "sv reload ${HAPROXY_SERVICE}" \ 
    $ARGS 
......
```

Dockerfile:

```
WORKDIR /marathon-lb 

ENTRYPOINT [ "/marathon-lb/run" ] 

CMD [ "sse", "--health-check", "--group", "external" ]

EXPOSE 80 443 9090 9091
```

#### Marathon-LB的运行模式

Marathon-LB支持3种运行模式：sse，event和poll。

**SSE模式**
在SSE模式下，marathon-lb.py脚本连接到Marathon事件API以获得有关应用状态更改的通知。这只适用于Marathon 0.11.0或更高版本。

```
docker run mesosphere/marathon-lb sse [other args]
```

**EVENT模式**

**注意：**EVENT模式已弃用，并且将在以后的版本中从marathon-lb中删除。

在事件模式下，marathon-lb.py脚本在Marathon中注册HTTP回调，以便在应用状态更改时获得通知。

```
docker run mesosphere/marathon-lb event callback-addr:port [other args]
```

**POLL模式**

在POLL模式下，marathon-lb.py脚本可以轮询API以定期获取调度程序状态。

```
docker run mesosphere/marathon-lb poll [other args]
```

可以设置POLL\_INTERVAL环境变量，更改轮询间隔（默认为60秒）。

#### 部署

在DC\/OS中，Marathon-LB以Docker镜像的方式部署，并且在Universe中存在预定义的包。可以通过DC\/OS CLI或WEB UI部署Marathon-LB。

```
dcos package install marathon-lb
```

如果要调整Marathon-LB的某些默认配置，可以通过下述命令检查当前Marathon-LB应用定义所支持的自定义参数：

```
dcos package describe --config marathon-lb
```

创建自定义的配置文件，使用自定义文件安装：

```
dcos package install --options=config.json  marathon-lb
```

**提供SSL证书**

在部署Marathon-LB时，可以为Docker容器指定SSL证书的方式有多种：

1. 可以通过设置`HAPROXY_SSL_CERT`环境变量来传递自己的SSL前端证书。如果需要多个证书，可以通过设置`HAPROXY_SSL_CERT0` - `HAPROXY_SSL_CERT100`来指定其他证书。变量内容将被写入`/etc/ssl/cert.pem`。

2. 可以使用`--ssl-certs`命令行参数提供SSL证书路径。MLB将使用这些证书路径指定的证书。

3. 如果不提供任何配置，MLB将在`/etc/ssl/cert.pem`上创建自签名证书，并且配置使用它。


**使用HAProxy Maps进行后端查找**

可以使用HAProxy映射加快Web应用程序（vhosts）到后端的查找速度。这对于大型部署非常有用，因为传统的vhost到后端的规则匹配比较需要相当长的时间，它需要顺序地比较每个规则。HAProxy映射会创建一个基于散列的查找表，因此它与其他方法相比速度较快，在Marathon-LB中可以用`--haproxy-map`配置参数来启用此功能。

### 管理接口

当Marathon-LB处于轮询模式时，这些接口将不起作用，因为在该模式中Marathon-LB进程在每次轮询之后退出，所以无法接收信号通知。

| Endpoint | Description |
| --- | --- |
| `:9090/haproxy?stats` | HAProxy stats endpoint. This produces an HTML page which can be viewed in your browser, providing various statistics about the current HAProxy instance. |
| `:9090/haproxy?stats;csv` | This is a CSV version of the stats above, which can be consumed by other tools. For example, it's used in the `zdd.py` script. |
| `:9090/_haproxy_health_check` | HAProxy health check endpoint. Returns `200 OK` if HAProxy is healthy. |
| `:9090/_haproxy_getconfig` | Returns the HAProxy config file as it was when HAProxy was started. Implemented in `getconfig.lua`. |
| `:9090/_haproxy_getvhostmap` | Returns the HAProxy vhost to backend map. This endpoint returns HAProxy map file only when the `--haproxy-map` flag is enabled, it returns an empty string otherwise. Implemented in `getmaps.lua`. |
| `:9090/_haproxy_getappmap` | Returns the HAProxy app ID to backend map. Like `_haproxy_getvhostmap`, this requires the `--haproxy-map` flag to be enabled and returns an empty string otherwise. Also implemented in `getmaps.lua`. |
| `:9090/_haproxy_getpids` | Returns the PIDs for all HAProxy instances within the current process namespace. This literally returns `$(pidof haproxy)`. Implemented in `getpids.lua`. This is also used by the `zdd.py` script to determine if connections have finished draining during a deploy. |
| `:9090/_mlb_signal/hup`\* | Sends a `SIGHUP` signal to the marathon-lb process, causing it to fetch the running apps from Marathon and reload the HAProxy config as though an event was received from Marathon. |
| `:9090/_mlb_signal/usr1`\* | Sends a `SIGUSR1` signal to the marathon-lb process, causing it to restart HAProxy with the existing config, without checking Marathon for changes. |

### HAProxy配置

可以通过两种途径在MLB中对HAProxy进行配置，一种是通过应用的标签（labels），另一种是通过模板（template）。

#### 应用标签

应用标签在Marathon的应用程序JSON定义中指定。这些标签可以用来覆盖HAProxy的默认行为。例如，可以将应用设置为`external`分组并为其配置一个名为`service.mesosphere.com`的虚拟主机：

```
{
  "id": "http-service",
  "labels": {
    "HAPROXY_GROUP":"external",
    "HAPROXY_0_VHOST":"service.mesosphere.com"
  }
}
```

可以为每个服务端口指定一些特定的标签，这些端口在标签中用{n}参数表示，其中{n}对应于从0开始的服务端口索引。可通过[HAProxy配置参考](/dcos-component-marathon-lb-template.md)查看可用的配置标签。

#### 模板

MLB在templates\/目录中搜索HAProxy的配置文件，该目录相对于启动MLB的运行脚本的路径位置。一些模板参数可以根据特定的服务端口进行配置。可以将自定义的模板参数添加到Docker镜像，或在镜像启动时（Marathon的应用JSON定义中）提供。

**覆盖模板配置**

某些模板参数可以使用应用程序标签覆盖。字符串在HAProxy的配置模板中被解释为文字配置参数，并遵守替换规则。HAProxy的配置将在更改后重新加载之前验证其正确性。注意：由于HAProxy在重新加载之前会检查配置，如果应用程序的标签的语法不正确，那么HAProxy将不会重新加载，因此可能导致过时的配置。

下面的示例是一个名为`http-service`的服务，它要求禁用`http-keep-alive`：

```
{
  "id": "http-service",
  "labels":{
    "HAPROXY_GROUP":"external",
    "HAPROXY_0_BACKEND_HTTP_OPTIONS":"  option forwardfor\n  no option http-keep-alive\n  http-request set-header X-Forwarded-Port %[dst_port]\n  http-request add-header X-Forwarded-Proto https if { ssl_fc }\n"
  }
}
```

**全局默认选项**

一种在不覆盖全局模板而添加HAProxy全局默认选项的快捷方式是可以通过环境变量`HAPROXY_GLOBAL_DEFAULT_OPTIONS`来指定用逗号分隔的选项列表。环境变量`HAPROXY_GLOBAL_DEFAULT_OPTIONS`的默认值为`redispatch,http-server-close,dontlognull`；作为示例，要添加`httplog`选项（并保留现有的默认值），应该指定：

```
HAPROXY_GLOBAL_DEFAULT_OPTIONS=redispatch,http-server-close,dontlognull,httplog
```

注意，当`HAPROXY_HEAD`模板参数已覆盖指定时，上述设置不起作用。

