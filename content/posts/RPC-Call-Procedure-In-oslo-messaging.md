---
date: "2018-06-21T9:18:57+08:00"
draft: false
title: "OpenStack RPC Call/CAST 调用流程"
subtitle: "oslo.messaging 库代码分析(一)"
categories: "OpenStack"
tags: ["OpenStack","oslo.messaging","RabbitMQ"]
---

最近遇到了一个关于nova 组件调用oslo.messaging 库代码出现异常的问题。抛出的异常，大概描述如下

```
2018-06-13 11:06:33.927 8965 ERROR oslo.messaging._drivers.impl_rabbit [req-6d5699d8-63be-4bba-b2e2-fe7f53dc2699 4f1b9c862dfa4191b61370b72e04795d 3b9839f8a90f47faa6566ef473b77593 - default default] Failed to publish message to topic 'nova': 'NoneType' object has no attribute 'getitem'

2018-06-13 11:06:33.927 8965 ERROR oslo.messaging._drivers.impl_rabbit [req-6d5699d8-63be-4bba-b2e2-fe7f53dc2699 4f1b9c862dfa4191b61370b72e04795d 3b9839f8a90f47faa6566ef473b77593 - default default] Unable to connect to AMQP server on 10.127.3.64:5672 after None tries: 'NoneType' object has no attribute 'getitem'

2018-06-13 11:06:33.928 8965 ERROR nova.api.openstack.extensions [req-6d5699d8-63be-4bba-b2e2-fe7f53dc2699 4f1b9c862dfa4191b61370b72e04795d 3b9839f8a90f47faa6566ef473b77593 - default default] Unexpected exception in API method: MessageDeliveryFailure: Unable to connect to AMQP server on 10.127.3.64:5672 after None tries: 'NoneType' object has no attribute 'getitem'

```

对着错误抛出的地方，找到相关代码片段，还是不知道是哪里的问题，遂准备对oslo.messaging 库中RabbitMQ 连接的初始化过程的相关代码进行完成分析，分析中用到的oslo.messsaging 版本为Queens 。在分析之前，自己参考了oslo.messaging中RPC  的例子，进行改动，用以实际调试验证。在分析最后，将附上用于调试的程序。

在oslo.messaging 库中，实现了对多个消息队列的调用封装，封装的消息队列有RibbitMQ, ZeroMQ, Kafka等。使用RabbitMQ, 实现一个RPC client 端(生产者）调用大概要经过三个步骤，具体的伪代码如下：

```
#步骤一
transport= messaging.get_rpc_transport(conf,url='rabbit://admin:pass@xxx:5672/')

#步骤二
target= messaging.Target(topic='xxx',server='xxx')
client= messaging.RPCClient(transport, target)

#步骤三
ret=client.call(xxx)
```

针对上述伪代码，本文将对oslo.messaging 库中代码的调用做三点分析：

1. oslo.messaging中transport 的初始化流程

2. oslo.messaging中target，RPC client 的初始化

3. oslo.messaging中call() 方法的执行 

通过以上三步，将澄清在oslo.messaging中，是何时何处建立RabbitMQ TCP，channel 连接的疑问。

   

# transport 的初始化流程

**在transport  的初始化流程中，最重要的应该是完了一个具体的消息队列驱动的初始化。**

从oslo_messaging/rpc/transport.py 中的 get_rpc_transport() 方法开始，看一个transport 的初始化流程。

```
def get_rpc_transport(conf, url=None,
    return msg_transport._get_transport( #入oslo_messaging.transport.py的_get_transport 方法
        conf, url, allowed_remote_exmods, 
        transport_cls=msg_transport.RPCTransport)

```

先不管参数conf 的具体值，假设url 的值为 rabbit://openstack:12345@10.127.6.125:5672/。transport_cls 的值实际为oslo_messaging.transport.py 中的RPCTransport 类,该类继承了同模块下的Transport 类。

oslo_messaging/transport.py

```
def _get_transport(conf, url=None, allowed_remote_exmods=None, aliases=None,
                   transport_cls=RPCTransport):
    allowed_remote_exmods = allowed_remote_exmods or []
    conf.register_opts(_transport_opts)

    if not isinstance(url, TransportURL):
        url = TransportURL.parse(conf, url, aliases) # 实例化一个 TransportURL 类

    kwargs = dict(default_exchange=conf.control_exchange,
                  allowed_remote_exmods=allowed_remote_exmods)

    try:
        mgr = driver.DriverManager('oslo.messaging.drivers', 
                                   url.transport.split('+')[0],
                                   invoke_on_load=True,
                                   invoke_args=[conf, url],
                                   invoke_kwds=kwargs)
    except RuntimeError as ex:
        raise DriverLoadFailure(url.transport, ex)

    # transport_cls 为RPCTransport 类,使用RabbitMQ时，mgr.driver为RabbitDriver实例
    return transport_cls(mgr.driver) 

```

返回值mgr 的driver 属性为某一消息队列的驱动（或具体消息队列调用的封装）。该driver的具体值和传入的URL 有关系。在上文给出的URL中，已指明使用的消息队列为RibbitMQ。所以，此处driver 的值为RabbitDriver 类的一个实例。RabbitDriver 类的实现在oslo_messaging/_drivers/impl_rabbit.py 中。具体如何在driver.DriverManager（）方法中调用到RabbitDriver 类暂不研究。

RabbitDriver 类继承了amqpdriver.AMQPDriverBase 及base.BaseDriver 方法，如：send， _send, send_notification,listen,cleanup 等方法。

先看RabbitDriver 的初始化：

oslo_messaging/_drivers/impl_rabbit.py

```
class RabbitDriver(amqpdriver.AMQPDriverBase):
    """RabbitMQ Driver

    The ``rabbit`` driver is the default driver used in OpenStack's
    integration tests.

    The driver is aliased as ``kombu`` to support upgrading existing
    installations with older settings.

    """

    def __init__(self, conf, url, # 传入的url 已经变为一个TransportURL类
                 default_exchange=None,
                 allowed_remote_exmods=None):
        opt_group = cfg.OptGroup(name='oslo_messaging_rabbit',
                                 title='RabbitMQ driver options')
        conf.register_group(opt_group)
        conf.register_opts(rabbit_opts, group=opt_group)
        conf.register_opts(rpc_amqp.amqp_opts, group=opt_group)
        conf.register_opts(base.base_opts, group=opt_group)
        conf = rpc_common.ConfigOptsProxy(conf, url, opt_group.name)

        self.missing_destination_retry_timeout = (
            conf.oslo_messaging_rabbit.kombu_missing_consumer_retry_timeout)

        self.prefetch_size = (
            conf.oslo_messaging_rabbit.rabbit_qos_prefetch_count)

        # the pool configuration properties  
        max_size = conf.oslo_messaging_rabbit.rpc_conn_pool_size  # 从配置文件获取参数
        min_size = conf.oslo_messaging_rabbit.conn_pool_min_size
        ttl = conf.oslo_messaging_rabbit.conn_pool_ttl

        connection_pool = pool.ConnectionPool( # 重点看这里
            conf, max_size, min_size, ttl, # max_size ,min_size 为连接池中连接的最大，最小数量
            url, Connection) // Connection 为该模块中Connection 类

        super(RabbitDriver, self).__init__( 
            conf, url,
            connection_pool, # pool.ConnectionPool 实例。
            default_exchange,
            allowed_remote_exmods
        )
```



重点看pool.ConnectionPool() 的初始化过程。在pool.ConnectionPool 类中，实现了建立连接，获取连接，归还连接，清空连接池等方法。该类初始化过程中，传入了连接池的TCP 连接数量的上 ，下限值，及具体的连接类。

oslo_messaging/_drivers/pool.py

```
class ConnectionPool(Pool):
    """Class that implements a Pool of Connections."""

    def __init__(self, conf, max_size, min_size, ttl, url, connection_cls):
        # 具体的一个连接类，采用RabbitMQDriver 时，值为impl_rabbit.py的Connection类
        self.connection_cls = connection_cls 
        self.conf = conf
        self.url = url
        
        # max_size,min_size 传入父类，在父类Pool中，看到max_size,min_size ,ttl 的默认值分别为：
        # 4,2,1200(秒)，ttl 表明空闲连接的存活时间.
        super(ConnectionPool, self).__init__(max_size, min_size, ttl, 
                                             self._on_expire)
```



总的来说，ConnectionPool维护了一个连接池，保管连接实例，但目前连接池为空，没有建立好的连接实例。何时调用create() 建立连接？带着这个疑问继续往下走。

返回到_get_transport ，完成了driver.DriverManager() 方法的调用，接着执行transport_cls(mgr.driver) 实例化一个transport，该transport中还未建立TCP 连接。



# target，client 初始化

target ,client的初始化在代码实现上都很明了，但是得明白每个参数的意义才能清楚初始化的目的。在相关类的源码中便可了解到传入参数的意义。

target 实例由target.py 中的Target 类实例化而来。Target类中封装了一个消息的目的地址，传入target 中的参数即构成了描述目的地址的元素。例如：在上文中的伪代码
```target= messaging.Target(topic='test',server='server1')```中，指定了消息发往的服务器是监听 ’test' topic 的server1 服务器。

oslo_messaging/target.py

```
class Target(object):
    def __init__(self, exchange=None, topic=None, namespace=None,
                 version=None, server=None, fanout=None,
                 legacy_namespaces=None):
        self.exchange = exchange
        self.topic = topic
        self.namespace = namespace
        self.version = version
        self.server = server
        self.fanout = fanout
        self.accepted_namespaces = [namespace] + (legacy_namespaces or [])
```



RPCClient 类中实现了远程调用的方法。实例化RPClient 的时候需要transport , target ,retry 等参数。

oslo_messaging/rpc/client.py

```
class RPCClient(_BaseCallContext):
    _marker = _BaseCallContext._marker
    def __init__(self, transport, target,
                 timeout=None, version_cap=None, serializer=None, retry=None,
                 call_monitor_timeout=None):
        #retry参数指明了retry 的次数，默认值None 值表示永久尝试，具体retry 是指什么动作，将在后边看到
        #timeout call() 调用的执行时间（包括等待时间）
        #call_monitor_timeout call() 调用期间服务端发送心跳的最大间隔时间
        if serializer is None:
            serializer = msg_serializer.NoOpSerializer() # 消息序列化方法

        if not isinstance(transport, msg_transport.RPCTransport):
            LOG.warning("Using notification transport for RPC. Please use "
                        "get_rpc_transport to obtain an RPC transport "
                        "instance.")
        #client 的初始化主要是调用父类初始化方法
        super(RPCClient, self).__init__(
            transport, target, serializer, timeout, version_cap, retry,
            call_monitor_timeout
        )
        
        #添加call 调用的等待回复时间参数：rpc_response_timeout，默认是60s
        self.conf.register_opts(_client_opts) 
```

RPCClient 有cast和call两种远程调用方式。cast方式为异步调用，call方式为同步调用。 使用这两个方法时，需要指定的参数有：调用的上下文参数(字典格式)，远程方法的名字，传递给远程方法的参数(字典格式)。

通过RPCClient 的prepre() 方法，还可为某一个远程方法调用修改client 的属性值，获得一个新的调用客户端。如修改retry 的值。



# call() 方法的执行 

深入分下RPClient的call() 或cast() 方法，会发现最终会调用_BaseCallContext类中的call() 或cast()方法，以call() 为例，看一下最后的call() 方法的实现。

oslo_messaging/rpc/client.py 

```
class _BaseCallContext(object):

    def call(self, ctxt, method, **kwargs):
        """Invoke a method and wait for a reply. See RPCClient.call()."""
        if self.target.fanout: #call调用的topic 不能是fanout 方式
            raise exceptions.InvalidTarget('A call cannot be used with fanout',
                                           self.target) 

        msg = self._make_message(ctxt, method, kwargs)
        msg_ctxt = self.serializer.serialize_context(ctxt)

        timeout = self.timeout
        # timeout 为空时，置为rpc_response_timeout，默认60s，不为空时，timeout 为有效等待时间参数
        if self.timeout is None:# timeout 为空时，置为rpc_response_timeout，默认60s
            timeout = self.conf.rpc_response_timeout  

        cm_timeout = self.call_monitor_timeout 

        self._check_version_cap(msg.get('version'))

        try:
            result = self.transport._send(self.target, msg_ctxt, msg, #调用transport 实例
                                          wait_for_reply=True, timeout=timeout,
                                          call_monitor_timeout=cm_timeout,#cm_timeout 可能为空
                                          retry=self.retry) #self.retry 传入_send
        except driver_base.TransportDriverError as ex:
            raise ClientSendError(self.target, ex)

        return self.serializer.deserialize_entity(ctxt, result)
```



前文说到：transport  为RPCTransport 类的实例，进入该类的_send() 方法

oslo_messaging/transport.py

     def _send(self, target, ctxt, message, wait_for_reply=None, timeout=None,
              call_monitor_timeout=None, retry=None):
         if not target.topic: # call或cast 都需指定topic 类型
                raise exceptions.InvalidTarget('A topic is required to send',
                                           target)
            # 调用具体driver 的send 方法                               
          return self._driver.send(target, ctxt, message,
                                 wait_for_reply=wait_for_reply, # 留意wait_for_reply为True
                                 timeout=timeout,
                                 call_monitor_timeout=call_monitor_timeout,
                                 retry=retry) # 留意retry 的值，从RPCClient 的初始化值或prepare()获取


回到具体的RabbitDriver 类，看具体的send 方法。在RabbitDriver 类中，send方法继承于基类AMQPDriverBase中的send()方法，最后调用了该基类的_send() 方法.

oslo_messaging/_drivers/amqpdriver.py

```
  def _send(self, target, ctxt, message,
              wait_for_reply=None, timeout=None, call_monitor_timeout=None,
              envelope=True, notify=False, retry=None):  
        msg = message
        if wait_for_reply:
            msg_id = uuid.uuid4().hex
            msg.update({'_msg_id': msg_id})
            
            # 初始化self._waiter,对于call() 方法，会在_get_reply_q()方法中，调用_get_connection
            # 建立TCP连接及channel 
            msg.update({'_reply_q': self._get_reply_q()}) 
            msg.update({'_timeout': call_monitor_timeout})

        rpc_amqp._add_unique_id(msg)
        unique_id = msg[rpc_amqp.UNIQUE_ID]

        rpc_amqp.pack_context(msg, ctxt)

        if envelope:
            msg = rpc_common.serialize_msg(msg)

        if wait_for_reply:
            self._waiter.listen(msg_id) #将msg id 加入待回复列表
            log_msg = "CALL msg_id: %s " % msg_id
        else:
            log_msg = "CAST unique_id: %s " % unique_id

        try:
            #返回一个ConnectionContext实例
            #根据消息发送类型，调用不同发送函数，并且RPCClient 中参数的retry 的值传到这里。
            #并且，此处的retry 是在连接建立好后使用，所以retry 的动作并不是尝试建立TCP连接

            with self._get_connection(rpc_common.PURPOSE_SEND) as conn: 
                if notify:
                   ...
                elif target.fanout:
                  ...
                else:
                  ...
    
            if wait_for_reply: 
                result = self._waiter.wait(msg_id, timeout, #调用ReplyWaiter类的wait方法
                                           call_monitor_timeout)
                if isinstance(result, Exception):
                    raise result
                return result
        finally:
            if wait_for_reply: 
                self._waiter.unlisten(msg_id) # 最后从回复列表移除
```



无论是call()或cast() 方法，都会调用```_get_connection() ```从连接池拿到一个连接，如果连接池为空，会建立连接。下面重点看一下```_get_connection()```方法的执行流程，这是建立通信的关键。

oslo_messaging/_drivers/amqpdriver.py

```
 def _get_connection(self, purpose=rpc_common.PURPOSE_SEND):
     # self._connection_pool 为RabbitDriver 初始化是生成的ConnectionPool 实例
     # rpc_common.PURPOSE_SEND 为字符串send
     # #返回一个ConnectionContext 实例
     return rpc_common.ConnectionContext(self._connection_pool,purpose=purpose)
```



oslo_messaging/_drivers/common.py

```
class ConnectionContext(Connection):
    def __init__(self, connection_pool, purpose):
        self.connection = None
        self.connection_pool = connection_pool
        pooled = purpose == PURPOSE_SEND # 改为pooled = (purpose == PURPOSE_SEND) 或许更明了

        if pooled:
            self.connection = connection_pool.get()# 创建新的连接或获取存在的连接
        else:
            # a non-pooled connection is requested, so create a new connection
            self.connection = connection_pool.create(purpose) #创建新的连接
        self.pooled = pooled
        self.connection.pooled = pooled
```

看了ConnectionContext类的初始化方法，我们还应当留意该类实现的```__enter__()```、```__exit__()```、```__del__()```方法，它们都默默做了一些工作。

再看```self.connection = connection_pool.get()```一句的具体调用：

oslo_messaging/_drivers/pool.py

```
class Pool(object):    
    def get(self):
        with self._cond:
            while True:
                try:
                    ttl_watch, item = self._items.pop()
                    self.expire()
                    return item # 返回一个有效连接
                except IndexError:
                    pass

                if self._current_size < self._max_size:
                    self._current_size += 1
                    break
    
                wait_condition(self._cond)
    
        # We've grabbed a slot and dropped the lock, now do the creation
        try:
            return self.create() #创建连接，至此，看到了create()方法的调用
        except Exception:
            with self._cond:
                self._current_size -= 1
            raise
```



在调用call() 方法后，我们看到了create()方法的调用，可见在oslo.messaing 中建立连接采取了一种滞后的方法，即真正第一次有远程方法调用时，开始建立连接。

oslo_messaging/_drivers/pool.py

    def create(self, purpose=common.PURPOSE_SEND):
        LOG.debug('Pool creating new connection')
        #self.connection_cls 是在驱动实例化时赋值，返回RabbitDriver类初始化函数，查看connection_cls的值
        return self.connection_cls(self.conf, self.url, purpose)

关于建立连接的具体过程，参考： [Rabbit Driver Connection 创建流程 ](https://rootdeep.github.io/posts/rpc-rabbit-con-in-oslo-messaging/)

## 根据消息分发类型，调用具体的方法

通过```_get_connection```得到一个pool.ConnectionContext 实例，返回到_send() 方法,继续执行分支。以 topic = target.topic 分支为例，继续往下看。进入该分支：

oslo_messaging/_drivers/amqpdriver.py

```
topic = target.topic
exchange = self._get_exchange(target) #获得交换器名字
if target.server:
    topic = '%s.%s' % (target.topic, target.server)
log_msg += "exchange '%(exchange)s'" \
           " topic '%(topic)s'" % {
               'exchange': exchange,
               'topic': topic}
LOG.debug(log_msg)
conn.topic_send(exchange_name=exchange, topic=topic, 
                msg=msg, timeout=timeout, retry=retry)
```

初次看conn.topic_send 一句，发现ConnectionContext类并没有topic_send() 方法，但多次阅读ConnectionContext类的实现后，发现该类中重写了```__getattr()__```方法，实际上调用的还是impl_rabbit.py /Connection  类的方法。(不得不感叹Python 代码太灵活,暗藏玄机)

oslo_messaging/_drivers/impl_rabbit.py

```
class Connection(object):  
   def topic_send(self, exchange_name, topic, msg, timeout=None, retry=None):
        """Send a 'topic' message."""
        exchange = kombu.entity.Exchange(
            name=exchange_name,
            type='topic',
            durable=self.amqp_durable_queues,
            auto_delete=self.amqp_auto_delete)

        self._ensure_publishing(self._publish, exchange, msg,
                                routing_key=topic, timeout=timeout,
                                retry=retry)

          
    def _ensure_publishing(self, method, exchange, msg, routing_key=None,
                           timeout=None, retry=None):
        """Send to a publisher based on the publisher class."""
    
        def _error_callback(exc):
            log_info = {'topic': exchange.name, 'err_str': exc}
            LOG.error(_LE("Failed to publish message to topic "
                          "'%(topic)s': %(err_str)s"), log_info)
            LOG.debug('Exception', exc_info=exc)
    
        method = functools.partial(method, exchange, msg, routing_key, timeout)
    
        with self._connection_lock:
            self.ensure(method, retry=retry, error_callback=_error_callback) # 带入了retry 值
```

关于进入ensure() 方法后的执行流程，在 Rabbit Driver Connection 创建流程 一节已经进行了分析，不同于从ensure_connection(）调用ensure() 的是，这次传给ensure () method 的值变了，并且ensure() 方法内部调用autoretry() 方法时， self.channel 也有值了。



还有不明白的问题是：在执行完autoretry_method()方法后（此时，autoretry_method() 方法中，传递的execute_method() 方法也执行完毕），为何还要调用self._set_current_channel() 方法。



回到oslo_messaging/_drivers/amqpdriver.py的  _send() 方法中， 如果是一个call 同步调用，还会单独建立一个TCP连接，等待回复消息。


oslo_messaging/_drivers/amqpdriver.py
```
class ReplyWaiter(object):    
    def wait(self, msg_id, timeout, call_monitor_timeout):
        timer = rpc_common.DecayingTimer(duration=timeout)
        timer.start()
        if call_monitor_timeout:
            call_monitor_timer = rpc_common.DecayingTimer(
                duration=call_monitor_timeout)
            call_monitor_timer.start()
        else:
            call_monitor_timer = None
        final_reply = None
        ending = False
        while not ending:
            timeout = timer.check_return(self._raise_timeout_exception, msg_id)
            if call_monitor_timer and timeout > 0:
                cm_timeout = call_monitor_timer.check_return(
                    self._raise_timeout_exception, msg_id)
                if cm_timeout < timeout:
                    timeout = cm_timeout
            try:
                message = self.waiters.get(msg_id, timeout=timeout) #阻塞等待，超时抛异常
            except moves.queue.Empty:
                self._raise_timeout_exception(msg_id)

            reply, ending = self._process_reply(message)
            if reply is not None:
                # NOTE(viktors): This can be either first _send_reply() with an
                # empty `result` field or a second _send_reply() with
                # ending=True and no `result` field.
                final_reply = reply
            elif ending is False:
                LOG.debug('Call monitor heartbeat received; '
                          'renewing timeout timer')
                call_monitor_timer.restart() # 此处有bug ,如果 call_monitor_timer 为None
        return final_reply
```



最后，再引用一张call调用的经典图：

![RabbitMQ-CALL](/RabbitMQ-CALL.png)



遗留的问题：

1. 在执行完autoretry_method()方法后（此时，autoretry_method() 方法中，传递的execute_method() 方法也执行完毕），为何还要调用self._set_current_channel() 方法。
2. 在初始化一个 ReplyWaiter类实例的时候，又创建了线程，并且把self.poll() 方法传递给线程，这个线程大概得目的是啥？







# 一个简单的RPC 调用示例 

**生产者端代码**

配置文件 :producer.conf

```
[DEFAULT]
log_dir=/root/rabbimt_test/
log_file=producer.log
debug=True
default_log_levels=oslo_messaging=DEBUG,oslo.messaging=DEBUG


[oslo_messaging_rabbit]
rabbit_max_retries=0
```

代码: rabbit-producer.py
```
#-*- coding:utf-8 -*-

from oslo_config import cfg
from oslo_log import log as logging
import oslo_messaging as messaging
import sys


# log initialization
CONF = cfg.CONF
logging.register_options(CONF)
CONF(sys.argv[1:])
logging.setup(CONF, "producer")



#module log init
LOG = logging.getLogger(__name__)


messaging.set_transport_defaults('myexchange')

LOG.info("Create a transport...")
transport= messaging.get_transport(cfg.CONF,url='rabbit://admin00:admin00@192.168.51.250:5671/')

#construct target, topic parameter is essential
LOG.info("Creating target")
target= messaging.Target(topic='test',server='server1')

LOG.info("create a client")
client= messaging.RPCClient(transport, target)

LOG.info("prepare a context")

ret= client.call(ctxt = {},method = 'test',arg = 'myarg')
LOG.info("the value from remote is %s"%ret)


cctxt= client.prepare(namespace='control', version='2.0', retry=1)
cctxt.cast({},'stop')

  
transport.cleanup()
```

运行命令： python   rabbit-producer.py  --config-file=./producer.conf  （producer.conf 和rabbit-producer.py 在相同文件夹下）



**消费者端代码**

rabbit-cosumer.py

```
#-*- coding:utf-8 -*-

from oslo_config import cfg
import  oslo_messaging as messaging
import time

class ServerControlEndpoint(object):

    target = messaging.Target(namespace='control', version='2.0')
    def __init__(self, server):
        self.server = server

    def stop(self, ctx):
        print("Call ServerControlEndpoint.stop()")        
        if self.server is not None:
           self.server.stop()



class TestEndpoint(object):

    def test(self, ctx, arg):
        print("Call TestEndpoint.test()")
        return arg

 
messaging.set_transport_defaults('myexchange')
transport = messaging.get_transport(cfg.CONF,url='rabbit://admin00:admin00@192.168.51.250:5671/')

target = messaging.Target(topic='test', server='server1')

endpoints = [
    ServerControlEndpoint(None),
    TestEndpoint(),
]

server = messaging.get_rpc_server(transport, target, endpoints,executor='blocking')

try:
    server.start()
    while True:
        time.sleep(1)
except KeyboardInterrupt:
   print("Stopping server")

server.stop()
server.wait()
transport.cleanup()
```



