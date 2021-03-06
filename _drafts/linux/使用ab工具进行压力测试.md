##使用ab工具对服务器进行压力测试

`ab`是Apache服务器附带的用于基准测试的工具。

在使用之前，需要先确认是否服务器安装了Apache Http Server，如果没有安装，则需要先安装：


    $ sudo yum install httpd
    $ ab -V
    This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/

下面是对`ab`工具比较常用的一些参数的解释


    aicode:~ mylxsw$ ab -h
    Usage: ab [options] [http[s]://]hostname[:port]/path
    Options are:
        -n requests     要执行的请求数量
        -c concurrency  并发请求数量
        -b windowsize   TCP发送/接收缓冲区大小，单位所以byte
        -p postfile     指定POST发送的数据文件，不要忘记设置-T参数
        -u putfile      指定PUT发送的数据文件，不要忘记设置-T参数
        -T content-type 使用POST/PUT发送数据时，指定Content-type请求头，例如.
                        'application/x-www-form-urlencoded'
                        默认是 'text/plain'
        -w              以HTML表格的形式输出结果
        -i              请求方式使用HEAD代替GET
        -C attribute    添加Cookie，例如'Apache=1234'. (可以重复设置)
        -H attribute    添加任意的请求Header，例如. 'Accept-Encoding: gzip'(可重复设置)
        -A attribute    添加基本的WWW认证信息，这个属性是用英文逗号分隔的用户名和密码
        -P attribute    添加代理服务器认证信息，使用逗号分隔用户名和密码
        -X proxy:port   指定代理服务器的地址和端口号
        -k              使用HTTP的KeepAlive特性
        -r              当Socket收到错误信息时不要退出.
        -Z ciphersuite  指定SSL/TLS加密套件
        -f protocol     指定SSL/TLS协议(SSL2, SSL3, TLS1 or ALL)
        ...


假如我们需要对`http://letv.com`进行压力测试，指定请求总数为100，并发用户数为10，我们可以以下面的方式进行测试

    $ ab -n 100 -c 10 http://letv.com/
    This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/

    Benchmarking letv.com (be patient).....done


    Server Software:        nginx/1.2.1
    Server Hostname:        letv.com
    Server Port:            80

    Document Path:          /
    Document Length:        184 bytes

    Concurrency Level:      10
    Time taken for tests:   0.396 seconds
    Complete requests:      100
    Failed requests:        0
    Write errors:           0
    Non-2xx responses:      100
    Total transferred:      37300 bytes
    HTML transferred:       18400 bytes
    Requests per second:    252.29 [#/sec] (mean)
    Time per request:       39.637 [ms] (mean)
    Time per request:       3.964 [ms] (mean, across all concurrent requests)
    Transfer rate:          91.90 [Kbytes/sec] received

    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        4    5   0.9      5       8
    Processing:     4   33  87.4      6     312
    Waiting:        4   33  87.3      5     311
    Total:          9   39  87.6     12     317

    Percentage of the requests served within a certain time (ms)
      50%     12
      66%     12
      75%     13
      80%     14
      90%     15
      95%    316
      98%    317
      99%    317
     100%    317 (longest request)


需要注意的几个字段是

- `Requests per second` 吞吐率（reqs/s），该字段值为252.29，该值表明了服务器每秒能够处理的请求数量。
- `Time per request` 平均请求处理时间，可以看到，该字段分为两行，有两个不同的值，代表了处理每隔请求所需要的时间，但是第一行的值是第二行的10倍。这是因为我们指定的并发数量为10，第一行为每次并发请求的平均耗时，第二行为每隔请求的耗时，因此，第一行值为第二行的值乘上并发请求数量。可以尝试将并发数改为20，这样就会看到第一行是第二行的20倍。
- `Transfer rate` 每秒从服务器获取的数据的长度。
