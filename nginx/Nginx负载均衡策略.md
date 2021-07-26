# Nginx负载均衡策略
负载均衡用于从“upstream”模块定义的后端服务器列表中选取一台服务器接受用户的请求。一个最基本的upstream模块是这样的，模块内的server是服务器列表：
```
upstream dynamic {
        server localhost:8080;
        server localhost:8081;
        server localhost:8082;
        server localhost:8083;
}
```
在upstream模块配置完成后，要让指定的访问反向代理到服务器列表：
```
 location ~ .*$ {
            index index.html;
            proxy_pass http://dynamic;
        }
```
这就是最基本的负载均衡实例，但这不足以满足实际需求；目前Nginx服务器的upstream模块支持6种方式的分配：
| 策略              |      描述       |
| ---------------- | -------------- |
| 轮询              | 默认方式        |
| weight           | 权重方式        |
| ip_hash          | 依据ip分配方式   |
| least_conn       | 最少连接方式     |
| fair（第三方）     | 响应时间方式     |
| url_hash（第三方） | 依据URL分配方式  |

## 轮询
最基本的配置方法，上面的例子就是轮询的方式，它是upstream模块默认的负载均衡默认策略。每个请求会按时间顺序逐一分配到不同的后端服务器。
```
upstream dynamic {
        server localhost:8080 max_fails=10 fail_timeout=60s;
        server localhost:8081 max_fails=10 fail_timeout=60s;
        server localhost:8082 max_fails=10 fail_timeout=60s;
        server localhost:8083 max_fails=10 fail_timeout=60s;
}
```  
有如下参数：  
fail_timeout与max_fails结合使用。  
max_fails设置在fail_timeout参数设置的时间内最大失败次数，如果在这个时间内，所有针对该服务器的请求都失败了，那么认为该服务器会被认为是停机了，fail_time服务器会被认为停机的时间长度,默认为10s。    
backup标记该服务器为备用服务器。当主服务器停止时，请求会被发送到它这里。  
down标记服务器永久停机了。  
注意：  
1、nginx缺省配置就是轮询策略  
2、在轮询中，如果服务器down掉了，会自动剔除该服务器。  
3、此策略适合服务器配置相当，无状态且短平快的服务使用。  

## weight
权重方式，在轮询策略的基础上指定轮询的几率。例子如下：
```
    #动态服务器组
    upstream dynamic {
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082   backup;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```
在该例子中，weight参数用于指定轮询几率，weight的默认值为1,；weight的数值与访问比率成正比，比如Tomcat 7.0被访问的几率为其他服务器的两倍。  
 
注意：  
1、权重越高分配到需要处理的请求越多。   
2、此策略可以与least_conn和ip_hash结合使用。  
3、此策略比较适合服务器的硬件配置差别比较大的情况。  

## ip_hash
指定负载均衡器按照基于客户端IP的分配方式，这个方法确保了相同的客户端的请求一直发送到相同的服务器，以保证session会话。这样每个访客都固定访问一个后端服务器，可以解决session不能跨服务器的问题。
```
#动态服务器组
    upstream dynamic {
        ip_hash;    #保证每个访客固定访问一个后端服务器
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```
注意：  
1、在nginx版本1.3.1之前，不能在ip_hash中使用权重（weight）。  
2、ip_hash不能与backup同时使用。  
3、此策略适合有状态服务，比如session。  
4、当有服务器需要剔除，必须手动down掉。  

## least_conn
把请求转发给连接数较少的后端服务器。轮询算法是把请求平均的转发给各个后端，使它们的负载大致相同；但是，有些请求占用的时间很长，会导致其所在的后端负载较高。这种情况下，least_conn这种方式就可以达到更好的负载均衡效果。
```
    #动态服务器组
    upstream dynamic {
        least_conn;    #把请求转发给连接数较少的后端服务器
        server localhost:8080   weight=2;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082 backup;  #tomcat 8.5
        server localhost:8083   max_fails=3 fail_timeout=20s;  #tomcat 9.0
    }
```
注意：  
1、此负载均衡策略适合请求处理时间长短不一造成服务器过载的情况。  

## 第三方策略
第三方的负载均衡策略的实现需要安装第三方插件。  
1、fair  
按照服务器端的响应时间来分配请求，响应时间短的优先分配。
```
    #动态服务器组
    upstream dynamic {
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
        fair;    #实现响应时间短的优先分配
    }
```
 2、url_hash  
 按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，要配合缓存命中来使用。同一个资源多次请求，可能会到达不同的服务器上，导致不必要的多次下载，缓存命中率不高，以及一些资源时间的浪费。而使用url_hash，可以使得同一个url（也就是同一个资源请求）会到达同一台服务器，一旦缓存住了资源，再此收到请求，就可以从缓存中读取。　
```
    #动态服务器组
    upstream dynamic {
        hash $request_uri;    #实现每个url定向到同一个后端服务器
        server localhost:8080;  #tomcat 7.0
        server localhost:8081;  #tomcat 8.0
        server localhost:8082;  #tomcat 8.5
        server localhost:8083;  #tomcat 9.0
    }
```
