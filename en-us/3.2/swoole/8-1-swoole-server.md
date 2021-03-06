# Swoole server

Swoole server relies on [swoole](https://github.com/JanHuang/swoole) and provides a flexible and elegant implementation.

!> swoole must rely on the ext-swoole extension, and version> = 1.9.6

swoole server configuration file stored in `config/server.php`, where `host` option is required, `options` configuration items see [server configuration options](http://wiki.swoole.com/wiki/page/274.html)

`` `php
$ php bin/server {start|status|stop|reload} [-t: web root directory, the default using the current command execution path]
`` `

The default mode of operation is `status`, which can be used to simply view the process status.

!> The way to access is consistent with the visit under FPM. It is recommended to use FPM in the development environment, and the production environment can try Swoole instead of FPM. If necessary

### daemon

To start daemon, just add a `--daemon | -d` option.

`` `php
$ php bin/server {start|status|stop|reload} -d > /dev/null
`` `

### Multi-port snooping

Server supports multi-port monitoring, only need a simple configuration.

```php
<?php 

return [
    'host' => 'http://0.0.0.0:9527',
    'class' => \FastD\Servitization\Server\HTTPServer::class,
    'options' => [
        'pid_file' => __DIR__ . '/../runtime/pid/' . app()->getName() . '.pid',
        'worker_num' => 10,
        'task_worker_num' => 20,
    ],
    'listeners' => [
        [
            'class' => \FastD\Servitization\Server\TCPServer::class,
            'host' => 'tcp://127.0.0.1:9528',
            'options' => [

            ]
        ],
        [
            'class' => \FastD\Servitization\Server\ManagerServer::class,
            'host' => 'tcp://127.0.0.1:9529',
        ],
    ],
];
```

The framework provides a default TCP service to provide TCP calls, which can be implemented by creating a discovery server and monitoring an RPC system.

Each port to be monitored needs to inherit the `FastD\Swoole\Server` object and implement the internal abstract method as shown in [examples](https://github.com/JanHuang/swoole/blob/master/examples/multi_port_server.php )

!> `swoole_http_server` and` swoole_websocket_server` Because it is implemented using inheritance subclasses, you can not create a Http/WebSocket server using listen. If the server's main function is RPC, but want to provide a simple Web management interface.
In this scenario, you can create an Http/WebSocket server and then listen for the listener's RPC server port. Official Description: [Swoole Monitor Port](http://wiki.swoole.com/wiki/page/525.html)

Because multiple ports are handled by shared processes, the `doTask` callback method needs to be handled on the master if task task handling is required.

### server process

Process server built-in process, when the server is started, it will automatically pull up the process, through [Swoole::addProcess](http://wiki.swoole.com/wiki/page/390.html).

The configuration is still [server.php](https://github.com/JanHuang/dobee/blob/master/config/server.php).

```php
<?php

return [
    'host' => 'http://0.0.0.0:9527',
    'class' => \FastD\Servitization\Server\HTTPServer::class,
    'options' => [
        'pid_file' => __DIR__ . '/../runtime/pid/' . app()->getName() . '.pid',
        'worker_num' => 10,
        'task_worker_num' => 20,
    ],
    'processes' => [
        \Processor\DemoProcessor::class
    ],
];
```

Rewrite `` `` handle` method of `FastD \ Swoole \ Process`,` handle` is a transaction that is executed by the process. Example: [DemoProcessor] (../../ tests / app / src / Processor / DemoProcessor.php)

> Complete configuration

```php
<?php
return [
    'listen' => 'http://0.0.0.0:9527',
    'class' => \FastD\Servitization\Server\HTTPServer::class,
    'options' => [
        'pid_file' => '',
        'worker_num' => 10,
        'task_worker_num' => 20,
    ],
    'processes' => [
        \Processor\DemoProcessor::class
    ],
    'listeners' => [
       [
           'class' => \FastD\Servitization\Server\TCPServer::class,
           'host' => 'tcp://127.0.0.1:9528',
       ]
    ],
];
```

### Task Server

In the swoole component, the Swoole Task feature has been implemented, the task itself is not provided by the framework itself, but its own Task server can be implemented by means of an integrated extension.

```php
<?php

namespace Server;


use FastD\Servitization\Server\HTTPServer;
use Psr\Http\Message\ServerRequestInterface;
use swoole_server;

class TaskServer extends HTTPServer
{
    public function doTask(swoole_server $server, $data, $taskId, $workerId)
    {
        echo $data . PHP_EOL;
    }

    public function doRequest(ServerRequestInterface $serverRequest)
    {
        $response = parent::doRequest($serverRequest); // TODO: Change the autogenerated stub

        server()->task("some data");

        return $response;
    }
}
```

When we implement our own swoole server, and need to start, we need to modify the `server.php` configuration file and modify the` class` configuration as follows:

Adjust the configuration:

```php
<?php

return [
    // code...
    'class' => \Server\TaskServer::class,
    // code...
];
```

Start the server.

### Swoole client

In addition to providing Swoole Server, it provides a remotely-operated Client-connected client that manages the remote server through the client after the server is started.

Use: `php bin/client schema://host:port`

When prompted, enter the command for a specific operation.
