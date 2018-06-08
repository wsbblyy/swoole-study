#swoole 快速学习记录
---------------

简单看了一遍文档, 感觉是swoole就是拿php实现了golang
其中除了多线程外, swoole中的 **coroutine** 就是go里的 **goroutine**
并且都是用**go**和**chan**来做routine的启动和协作

用swoole就可以不用nginx了, 直接在服务器输入命令 **-php server.php**
就可以执行服务端的代码了, 具体你希望服务端做什么需要先有业务逻辑, 然后
通过端口带参数访问, 都是非常基础的东西

#示例展示

以下代码放入你的server.php(名字随便)里直接用**-php server.php**
就行了不用nginx配置, 在文件里配置端口就行了

##创建HTTP服务端
```php
$http = new swoole_http_server("0.0.0.0", 9501);

$http->on('request', function ($request, $response) {
    var_dump($request->get, $request->post);
    $response->header("Content-Type", "text/html; charset=utf-8");
    $response->end("<h1>Hello Swoole. #".rand(1000, 9999)."</h1>");
});

$http->start();
```

##创建WebSocket服务端
```php
//创建websocket服务器对象，监听0.0.0.0:9502端口
$ws = new swoole_websocket_server("0.0.0.0", 9502);

//监听WebSocket连接打开事件
$ws->on('open', function ($ws, $request) {
    var_dump($request->fd, $request->get, $request->server);
    $ws->push($request->fd, "hello, welcome\n");
});

//监听WebSocket消息事件
$ws->on('message', function ($ws, $frame) {
    echo "Message: {$frame->data}\n";
    $ws->push($frame->fd, "server: {$frame->data}");
});

//监听WebSocket连接关闭事件
$ws->on('close', function ($ws, $fd) {
    echo "client-{$fd} is closed\n";
});

$ws->start();
```

##创建TCP服务端
```php
//创建Server对象，监听 127.0.0.1:9501端口
$serv = new swoole_server("127.0.0.1", 9501); 

//监听连接进入事件
$serv->on('connect', function ($serv, $fd) {  
    echo "Client: Connect.\n";
});

//监听数据接收事件
$serv->on('receive', function ($serv, $fd, $from_id, $data) {
    $serv->send($fd, "Server: ".$data);
});

//监听连接关闭事件
$serv->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});

//启动服务器
$serv->start(); 
```

##创建Serverd对象
```php
$serv = new swoole_server('0.0.0.0', 9501, SWOOLE_BASE, SWOOLE_SOCK_TCP);
```

##设置运行时参数
```php
$serv->set(array(
    'worker_num' => 4,
    'daemonize' => true,
    'backlog' => 128,
));
```

##注册事件回调函数
```php
$serv->on('Connect', 'my_onConnect');
$serv->on('Receive', 'my_onReceive');
$serv->on('Close', 'my_onClose');
```
##监听新端口
```php
$port1 = $server->listen("127.0.0.1", 9501, SWOOLE_SOCK_TCP);
$port2 = $server->listen("127.0.0.1", 9502, SWOOLE_SOCK_UDP);
$port3 = $server->listen("127.0.0.1", 9503, SWOOLE_SOCK_TCP | SWOOLE_SSL);
```

##设置网络协议
```php
$port1->set([
    'open_length_check' => true,
    'package_length_type' => 'N',
    'package_length_offset' => 0,
    'package_max_length' => 800000,]
);
```

##设置回调函数
```php
$port1->on('connect', function ($serv, $fd){
    echo "Client:Connect.\n";
});

$port1->on('receive', function ($serv, $fd, $from_id, $data) {
    $serv->send($fd, 'Swoole: '.$data);
    $serv->close($fd);
});

$port1->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});
```

以上都是创建服务端的方法

下面都方法

##设置定时器
```php
//每隔2000ms触发一次
swoole_timer_tick(2000, function ($timer_id) {
    echo "tick-2000ms\n";
});

//3000ms后执行此函数
swoole_timer_after(3000, function () {
    echo "after 3000ms.\n";
});
```

##执行异步任务
```php
$serv = new swoole_server("127.0.0.1", 9501);

//设置异步任务的工作进程数量
$serv->set(array('task_worker_num' => 4));

$serv->on('receive', function($serv, $fd, $from_id, $data) {
    //投递异步任务
    $task_id = $serv->task($data);
    echo "Dispath AsyncTask: id=$task_id\n";
});

//处理异步任务
$serv->on('task', function ($serv, $task_id, $from_id, $data) {
    echo "New AsyncTask[id=$task_id]".PHP_EOL;
    //返回任务执行的结果
    $serv->finish("$data -> OK");
});

//处理异步任务的结果
$serv->on('finish', function ($serv, $task_id, $data) {
    echo "AsyncTask[$task_id] Finish: $data".PHP_EOL;
});

$serv->start();
```

##swoole携程(特色)

####使用示例
```php
$http = new swoole_http_server("127.0.0.1", 9501);

$http->on("request", function ($request, $response) {
    $client = new Swoole\Coroutine\Client(SWOOLE_SOCK_TCP);
    $client->connect("127.0.0.1", 8888, 0.5);
    //调用connect将触发协程切换
    $client->send("hello world from swoole");
    //调用recv将触发协程切换
    $ret = $client->recv();
   $response->header("Content-Type", "text/plain");
    $response->end($ret);
    $client->close();
});

$http->start();
```

####创建协程
```php
go(function () {
    co::sleep(0.5);
    echo "hello";
});
go("test");
go([$object, "method"]);
```

####通道操作
```php
$c = new chan(1);
$c->push($data);
$c->pop();
```

####协程客户端
```php
$redis = new Co\Redis;
$mysql = new Co\MySQL;
$http = new Co\Http\Client;
$tcp = new Co\Client;
$http2 = new Co\Http2\Client;
```

####其他
```php
co::sleep(100);
co::fread($fp);
co::gethostbyname('www.baidu.com');
```

####Coroutine::create
```php
for($i = 0; $i < 100; $i++) {
    Swoole\Coroutine::create(function() use ($i) {
        $redis = new Swoole\Coroutine\Redis();
        $res = $redis->connect('127.0.0.1', 6379);
        $ret = $redis->incr('coroutine');
        $redis->close();
        if ($i == 50) {
            Swoole\Coroutine::create(function() use ($i) {
                $redis = new Swoole\Coroutine\Redis();
                $res = $redis->connect('127.0.0.1', 6379);
                $ret = $redis->set('coroutine_i', 50);
                $redis->close();
            });
        }
    });
}
```


