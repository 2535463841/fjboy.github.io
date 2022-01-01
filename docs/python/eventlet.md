eventlet的核心是协程（也叫做green thread）。

它通过[greenlet](http://www.oschina.net/p/greenlet)提供的协程功能，让开发者可以不用将以往的多线程等并发程序的开发方式转变成异步状态机模型，就能直接使用select/epoll/kqueue等操作系统提供的支持高并发IO接口，并且能尽可能地发挥它们在并发上的优势。

 ```python
 import sys
 import socket
 import eventlet
 import eventlet.wsgi
 import webob.dec
 import os
 import time
 
 '''
  主进程 创建 若干个子进程，同时监听socket文件
 '''
 socket_path = '/home/sockettest'
 mysocket = eventlet.listen(socket_path, family=socket.AF_UNIX, backlog=10)
 application = MetadataProxyHandler()
 children = {}
 workers = 5
 def start_child():
     pid = os.fork()
     if pid == 0:
         while True:
             listen_socket()
         sys.exit(1)
     else:
         children[pid] = application
         print 'fork process %d' % pid
 def listen_socket():
     client, address = mysocket.accept()
     print("server[%s] %s dealing request...  %s" %
           (os.getpid(), str(client), address))
     buf = client.recv(1024)
     # time.sleep(3)
     #print("         %s" %(buf))
     client.close()
 while len(children) < workers:
     start_child()
 
 ```

```python
import eventlet
import eventlet.greenpool
from eventlet import wsgi
import time
import socket
import os
# import threading
def handler(env, start_response):
    print '%s handling ... %s' % (os.getpid(), env['PATH_INFO'])
    if env['PATH_INFO'] == '/sleep':
        start = time.time()
        while (time.time() - start) < 60:
            pass
        # eventlet.sleep(10)
    # if env['PATH_INFO'] != '/':
    #     start_response('404 Not Found', [('Content-Type', 'text/plain')])
    #     return ['Not Found\r\n']
    start_response('200 OK', [('Content-Type', 'text/plain')])
    data = {"users": [{"id": "1"}]}
    return data
_socket = eventlet.listen(('', 8093))
_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
pool = eventlet.greenpool.GreenPool(10)
pool.spawn(wsgi.server, _socket, handler)
pool.waitall()
print 'waiting ...'
```

