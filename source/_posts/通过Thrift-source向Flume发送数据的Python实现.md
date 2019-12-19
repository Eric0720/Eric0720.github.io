---
title: 通过Thrift source向Flume发送数据的Python实现
date: 2019-08-16 11:12:45
tags: flume
---

前面说到，因为兼容之前的旧项目（使用的scribe收集日志，但facebook好像早就停止更新维护），现在的收集架构是scribe+flume，故Flume使用Thrift source. 

另外，测试环境下因为scribe日志机器和flume不在一台机器，所以这里使用python将数据发送到flume进行测试，以下是具体的实现过程:  
  
> 环境：Python 2.6.6/CDH-5.8.3 Flume 1.8/Thrift 0.5.0/  

首先，我们需要一个Thrift协议的Python Flume客户端的模块，这个模块可以根据Thrift的定义自动生成。先从flume官网下载src源码包 ：

`wget  http://www.apache.org/dist/flume/1.8.0/apache-flume-1.8.0-src.tar.gz`

<!-- more -->

下载到本地之后解压，在目录apache-flume-1.8.0-src\flume-ng-sdk\src\main\thrift下有Thrift对应的定义文件，并用它来生成对应的客户端模块：  

```
tar -xzvf apache-flume-1.8.0-src.tar.gz

cd apache-flume-1.8.0-src\flume-ng-sdk\src\main\thrift

thrift --gen py flume.thrit

```


你会在当前目录下得到一个叫做gen-py的目录，我们将其更名为genpy之后，放到Python的系统模块路径中去：  

`mv gen-py/  /usr/lib/python2.6/site-packag/genpy`

此时，你就可以通过以下过程来引用这个模块了：  
>[GCC 4.4.7 20120313 (Red Hat 4.4.7-3)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>from genpy import flume
> dir(flume)
>['__all__', '__builtins__', '__doc__', '__file__', '__name__', '__package__', '__path__']

* 代码进行测试

```
#!/usr/bin/env python
#encoding: utf-8
from genpy.flume import ThriftSourceProtocol  
from genpy.flume.ttypes import ThriftFlumeEvent  
from thrift.transport import TTransport, TSocket  
from thrift.protocol import TCompactProtocol  
  
class _Transport(object):  
    def __init__(self, thrift_host, thrift_port, timeout=None, unix_socket=None):  
        self.thrift_host = thrift_host  
        self.thrift_port = thrift_port  
        self.timeout = timeout  
        self.unix_socket = unix_socket 
        self._socket = TSocket.TSocket(self.thrift_host, self.thrift_port, self.unix_socket)  
        self._transport_factory = TTransport.TFramedTransportFactory()  
        self._transport = self._transport_factory.getTransport(self._socket)  
          
    def connect(self):  
        try:  
            if self.timeout:  
                self._socket.setTimeout(self.timeout)  
            if not self.is_open():  
                self._transport = self._transport_factory.getTransport(self._socket)  
                self._transport.open()  
        except Exception, e:
            print '#'*20  
            print(e)  
            self.close()  
                 
    def is_open(self):  
        return self._transport.isOpen()  
      
    def get_transport(self):  
        return self._transport  
      
    def close(self):  
        self._transport.close()  
          
class FlumeClient(object):  
    def __init__(self, thrift_host, thrift_port, timeout=None, unix_socket=None):  
        self._transObj = _Transport(thrift_host, thrift_port, timeout=timeout, unix_socket=unix_socket)  
        self._protocol = TCompactProtocol.TCompactProtocol(trans=self._transObj.get_transport())  
        self.client = ThriftSourceProtocol.Client(iprot=self._protocol, oprot=self._protocol)  
        self._transObj.connect()  
          
    def send(self, event):  
        try:  
            self.client.append(event)  
        except Exception, e:  
            print(e)  
        finally:  
            self._transObj.connect()  
      
    def send_batch(self, events):  
        try:  
            self.client.appendBatch(events)  
        except Exception, e:  
            print(e)  
        finally:  
            self._transObj.connect()  
      
    def close(self):  
        self._transObj.close()  
              
if __name__ == '__main__':  
    import random  
    flume_client = FlumeClient('xxxxxx',4444)  
    events = [ThriftFlumeEvent({'GAMENAME':'NYHX'},'NYevents under hello world %s' %(random.randint(0, 100))) for _ in range(100)]    
    events1 = [ThriftFlumeEvent({'GAMENAME':'FSZHS'},'FSevents under hello world %s' %(random.randint(0, 100))) for _ in range(100)]  
    flume_client.send_batch(events)  
    flume_client.send_batch(events1)  
    flume_client.close() 
```

*  贴上flume的配置，这里使用selector进行筛选，只输出header内容为FSZHS的日志

```
tier1.sources.source1.selector.type = multiplexing
tier1.sources.source1.selector.header = GAMENAME
tier1.sources.source1.selector.mapping.FSZHS = channel1


# 配置kafka接收数据
#tier1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
#tier1.sinks.sink1.brokerList = localhost:9092
#tier1.sinks.sink1.topic = test
#tier1.sinks.sink1.serializer.class = kafka.serializer.StringEncoder
#tier1.sinks.sink1.channel = channel1
```
然后日志输出了指定header的内容

>2019-08-16 10:57:41,700 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME= **FSZHS** } body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h } 
>2019-08-16 10:57:41,701 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h } 
>2019-08-16 10:57:41,701 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h }
> 2019-08-16 10:57:41,702 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h } 
> 2019-08-16 10:57:41,703 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h } 
> 2019-08-16 10:57:41,703 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h } 
> 2019-08-16 10:57:41,704 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{GAMENAME=**FSZHS**} body: 46 53 65 76 65 6E 74 73 20 75 6E 64 65 72 20 68 FSevents under h }

下面代码为监控日志文件并实时发送给flume：
大体流程，读取日志，发送到flume，然后记录读取位置。

```
#!/usr/bin/env python
# encoding: utf-8
from multiprocessing.dummy import Pool as ThreadPool
import time,datetime,os,sys

from flume import FlumeClient
from genpy.flume.ttypes import ThriftFlumeEvent

hour=datetime.datetime.now().strftime('%H')

if hour == '23':
    day=(datetime.datetime.now() + datetime.timedelta(days=1)).strftime('%Y-%m-%d')
else:
    day=datetime.datetime.now().strftime('%Y-%m-%d')

def read_server():
    time.sleep(5)
    while True:
        try:
            if not os.path.exists('/data/scribe/log_xxx/PRODUCT_xxx_ANDR_%s/PRODUCT_xxx_ANDR_%s_00000'%(day,day)):
                print('服务端日志文件不存在,sleep........50')
                time.sleep(50)
                continue
            fread_tell = '0'
            if os.path.exists('/home/eric/servertell/server_tell_%s.txt' % day):
                fread = open('/home/eric/servertell/server_tell_%s.txt' % day)
                fread_tells = fread.readlines()
                fread_tell = fread_tells[-1]
                fread.close()
            else:
                fread = open('/home/eric/servertell/server_tell_%s.txt' % day,'a+')
                fread.write('0\n')
                fread.close()
            ffile = open('/data/scribe/log_xxx/PRODUCT_xxxx_ANDR_%s/PRODUCT_xxx_ANDR_%s_00000'%(day,day))
            if not len(fread_tell):
                continue
            ffile.seek(int(fread_tell))
            lines = ffile.readlines()
            log = None
            print'未读行数======',(len(lines))
            if len(lines)<1:
                time.sleep(10)
                print '行数少于1休眠10s..................'
            else:
                for line in lines:
                    if line == line.strip() or len(line)<10:
                        time.sleep(5)
                        print "空行或者小于10行.........."
                        continue
                    else:
                        print "reading................."
                        print line
                        flume_client = FlumeClient('xxxxxxx',4444)
                        events1 = [ThriftFlumeEvent({'GAMENAME':'FSZHS'},line.strip('\n'))]
                        flume_client.send_batch(events1)
                        #bilog.sendBI(line.strip('\n'),'PRODUCT_CKQY_ANDR_%s'%(day))
            where1 = ffile.tell()
            while True:
                where1 = where1 -1
                seek = ffile.seek(where1,os.SEEK_SET)
                if ffile.read(1)=='\n':
                    break
            print '读取位置',(where1)
            f = open('/home/eric/servertell/server_tell_%s.txt' % day,'a+')
            f.write('\n')
            f.write(str(where1))
            f.close()
            ffile.close()
            if log:
                log.close()
        except Exception as e:
            traceback.print_exc()
            time.sleep(5)
            continue
if __name__ == '__main__':
    pool = ThreadPool(processes = 1)
    pool.apply_async(read_server)
    pool.close()
    pool.join()
```
至此日志转发完成。