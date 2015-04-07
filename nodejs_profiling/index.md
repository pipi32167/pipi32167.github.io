###零、优化原则

80%的性能被20%的代码吞噬掉了。

###一、环境

OS:     CentOS 6.x
node:   0.10.x
pomelo: 1.0.x

###二、准备

1、全局安装[webkit-devtools-agent-frontend](https://www.npmjs.com/package/webkit-devtools-agent)：

`npm i -g webkit-devtools-agent-frontend`

2、在package.json中添加[webkit-devtools-agent](https://www.npmjs.com/package/webkit-devtools-agent-frontend)依赖并下载安装依赖。

3、在app.js（程序入口文件，也可能是index.js）中加入如下代码

    process.on('SIGUSR2', function() {
      var agent = require('webkit-devtools-agent');
      if (agent.server) {
        agent.stop();
      } else {
        agent.start({
          port: 9999,
          bind_to: '0.0.0.0',
          ipc_port: 3333,
          verbose: true
        });
      }
    });


4、运行程序，在命令行中输入`kill -USR2 <pid>`，然后运行`webkit-devtools-agent-frontend`。打开chrome浏览器，访问`http://localhost:9090/inspector.html?host=<host>:9999&page=0`，在打开的页面中可以进行cpu和内存的profiling，其中host可以是本地也可以是远程的。

5、profiling结束后，在命令行中输入`kill -USR2 <pid>`，可以关闭webkit-devtools-agent。

7、安装dstat系统监控命令：`yum install -y dstat`。

8、安装valgrind命令：`yum install -y valgrind`。

9、按照需求完成压测机器人。

###三、cpu优化

cpu的优化分两块，第一个是统计每一类请求，找出特别消耗cpu的请求进行优化，第二个是通过压测观察哪块代码是cpu消耗大户，然后进行针对性的优化。

#####1、请求优化
在程序中为每种请求记录下开始时间和结束时间，每种请求都应该有个唯一ID，如pomelo的话，可能是gamelogic.chatHandler.globalChat或者gamelogic.userRemote.enter，如果是express的话，可能是/user/login。将所有的请求汇总统计，最后统计出平均消耗最高的请求和总消耗最高的请求，然后进行优化。

#####2、代码块优化
用压测机器人进行全面的测试，webkit-devtools-agent进行cpu profile，找出耗时最多的代码块。一般来说，遍历和查找操作是cpu消耗大户，尤其是习惯使用underscore等库进行的操作。有争论说使用函数式编程，这点效率不是问题，然而据我的经验来看，频繁操作的话，消耗并不小，尤其是使用链式调用，如_.filter().map()之类操作，相当于做了两次遍历，所以应尽可能地使用原生的遍历操作。退一步说，即使做不到全部替换为原生的遍历操作，也应根据profiling的结果进行针对性的代码优化。


###四、内存优化
内存优化一般有两块内容，第一个是内存泄露，这是最糟糕的，属于严重bug，在上线前必须尽可能地查出来，第二个是编码造成的内存浪费。
在nodejs的世界中，内存泄露分为两种，一种是js的内存泄露，另一种是C++  addon的内存泄露。

#####1、js的内存泄露
内存泄露一般要程序经过一段时间的运行才能发觉，所以需要使用压测机器人来模拟。值得注意的是，压测机器人应当尽可能地包含游戏的主要逻辑，注册、登录、退出、游戏操作等，然后制定压测策略，比如同时在线3000人，每秒钟注册登录2人，退出2人。运行机器人后需要预热一段时间，当在线用户数稳定下来以后，使用`dstat -tm 5`来观察系统的状况，这个命令每5秒钟打印一次内存的使用情况。
如下图所示：

![dstat memory usage](https://raw.githubusercontent.com/pipi32167/pipi32167.github.io/master/nodejs_profiling/dstat_memory_usage.png)

[其中memory-usage的used是实际使用内存](http://www.linuxatemyram.com/)。
程序运行一段时间后——这段时间视情况而定，如果短时间内没有显著的内存泄露，一般需要24小时以上——再观察内存的用量，如果出现显著的增长，那么停止压测，等待资源释放完毕，用webkit-devtools-agent将内存dump下来，查看哪些内存是未释放的。

#####2、C++ addon的内存泄露
如果第一步未能找出js的内存泄露，但是内存却存在显著的增长，那么有可能是C++ addon的内存泄露。使用valgrind来执行node程序，可以检测到C++的内存泄露问题。美中不足的是，用valgrind来执行程序会很消耗cpu，所以在运行valgrind的时候，进行常规的客户端测试比较好。
由于pomelo是使用spawn的方式来启动多个进程的，如果需要使用valgrind执行某个进程，可以修改node_modules/pomelo/lib/master/starter.js的spawnProcess函数，指定serverId用valgrind来运行。

#####3、js的内存使用优化
node程序很耗内存，尤其与C/C++程序相比较，而且还有一些陷阱存在，稍有不慎就栽个大跟头。
比如说我遇到的最大的陷阱是Object.defineProperty的使用不当：本应将属性绑定在prototype上，但是却绑定到了具体的对象上，结果造成了内存的剧增。
另一些是程序的具体逻辑优化，比如说某些完成了的任务，只需要记录一个ID即可，没有必要为其生成一个对象。

###五、存储优化
TODO

###六、系统监控️
TODO
