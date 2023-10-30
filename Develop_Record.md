<h1>IOT平台开发日志</h1>

***Form Yuhao***

## DGIOT平台
### 1.平台实地部署 
DGIOT平台选用腾讯云服务器，操作系统选取CentOS7.6 64位系统，实例配置为标准型S5 - 2核 2G。公网IP：43.137.18.23

### 2.开发环境配置（TODO）
选取IDEA+Erlang配置开发环境：
git clone 项目代码至本地。试图为项目配置Erlang的SDK失败。
当Erlang/OTP 26.1.2加入系统环境变量后，在cmd调用erl出现资源包crash的情况。
查询Chatgpt后认为：在控制台模式设置的erl的某些配置或本地环境配置发生了冲突。解决方法未知。
>
C: \Users\yuhao>erl --version<br />
一ERROR REPORT—=== 30-Oct-2023: :10:42:58.271000===**State machine user_drv terminating<br />
When server state= {undefined, undefined}Reason for termination = error :<br /> {badmatch,<br />
{error,<br />
{'SetConsoleMode’,’虏腻胸媒麓铆腻贸隆拢\r\n’}}}<br />
*Callback modules = [user_drv]<br />
** Callback mode = state_functions*Stacktrace =<br />
水*[{prim_tty, init,1,[{file, "prim_tty.erl"}，{line,221}]}，<br />
{user_drv,init,1,[{file,"user_drv.er1"}，{line,169} ]}，<br />
{gen_statem,init_it,6,[{file,"gen_statem.erl"}，{line, 966} ]}，{proc_lib,<br />init_p_do_apply,3,[{file,"proc_lib.erl"}，{line,241}]}]<br />
=CRASH REPORT==== 30-Oct-2023::10:42:58.272000===<br />
crasher:<br />
initial call: user_drv : init/1pid: <0.65.0><br />
registered_name:[]<br />
exception error : no match of right hand side value<br />
{error,{'SetConsoleMode',’虏脓腾媒麓铆腻贸隆拢\r\n’}}<br />
in function prim_tty:init/1 (prim_tty.erl，line 221)<br />
in call from user_drv :init/1 (user_drv.erl，line 169)<br />
in call from gen_statem: init_it/6 (gen_statem.erl，line 966)ancestors: [<0.64.0>, kernel_sup,<0.47.0>]<br />

### 3.源码研读
##### 1.通信部分：（src/emqx_ws_connectrion）
**（1）.DGIOT平台选取Websocket进行网页和服务器的双向通信**
>主要特点和优势：
实时性：WebSocket提供了真正的实时通信能力，可以通过单个长期连接在客户端和服务器之间进行快速、低延迟的数据传输。 <br />
双向通信：WebSocket支持双向通信，客户端和服务器可以同时发送和接收数据，而不仅仅是传统的请求-响应方式。 <br />
减少延迟：相对于轮询或长轮询等技术，WebSocket减少了不必要的数据传输和延迟，提供更高效的通信。
节省带宽：WebSocket使用更轻量级的消息头，降低了通信开销，节省了带宽和服务器资源。 <br />
兼容性：WebSocket协议兼容现代的Web浏览器和服务器，并且有广泛的支持。 <br />
WebSocket协议基于标准的HTTP握手过程，在初始连接之后，会升级到WebSocket连接。连接建立后，双方可以直接通过发送消息进行通信，而无需频繁地发起新的HTTP请求。
在前端开发中，可以使用JavaScript的WebSocket API来创建和管理WebSocket连接，并通过事件处理程序接收和发送消息。后端服务器需要支持WebSocket协议，一些常见的Web服务器框架或库提供了对WebSocket的支持，如Node.js的Socket.IO、Java的Java-WebSocket等。 <br />
WebSocket被广泛应用于实时聊天应用、在线游戏、股票市场行情、推送通知等需要实时数据传输和即时通信的场景。它为开发者提供了强大而简便的工具，使得构建实时应用变得更加容易和高效。

Websocket也可以在java中被运用：
>java—WebSocket是基于Java的开源库，用于实现websocket协议的客户端和服务器端。其提供了一组易于使用的API，使开发者能够轻松地构建websocket应用程序。

**（2）websocket 心跳重连（保护机制）**

websocket是前后端交互的长连接，前后端也都可能因为一些情况导致连接失效并且相互之间没有反馈提醒。因此为了保证连接的可持续性和稳定性，使用websocket心跳重连。

在使用原生websocket的时候，如果设备网络断开，不会立刻触发websocket的任何事件，前端也就无法得知当前连接是否已经断开。这个时候如果调用websocket.send方法，浏览器才会发现链接断开了，便会立刻或者一定短时间后（不同浏览器或者浏览器版本可能表现不同）触发onclose函数。

后端websocket服务也可能出现异常，造成连接断开，这时前端也并没有收到断开通知，因此需要前端定时发送心跳消息ping，后端收到ping类型的消息，立马返回pong消息，告知前端连接正常。如果一定时间没收到pong消息，就说明连接不正常，前端便会执行重连。

为了解决以上两个问题，以前端作为主动方，定时发送ping消息，用于检测网络和前后端连接问题。一旦发现异常，前端持续执行重连逻辑，直到重连成功。
~~~erl
%% Pings should be replied with pongs, cowboy does it automatically
%% Pongs can be safely ignored. Clause here simply prevents crash.
websocket_handle(Frame, State) when Frame =:= ping; Frame =:= pong ->
    return(State);

websocket_handle({Frame, _}, State) when Frame =:= ping; Frame =:= pong ->
    return(State);

%%%函数websocket_handle根据输入的帧类型（ping或pong）决定如何处理。如果是ping或pong帧，则直接返回当前状态State。证明连接依旧存在

websocket_handle({Frame, _}, State) ->
    %% TODO: should not close the ws connection
    ?LOG(error, "Unexpected frame - ~p", [Frame]),
    shutdown(unexpected_ws_frame, State).

websocket_info({call, From, Req}, State) ->
    handle_call(From, Req, State);

websocket_info({cast, rate_limit}, State) ->
    Stats = #{cnt => emqx_pd:reset_counter(incoming_pubs),
              oct => emqx_pd:reset_counter(incoming_bytes)
             },
    NState = postpone({check_gc, Stats}, State),
    return(ensure_rate_limit(Stats, NState));

websocket_info({cast, Msg}, State) ->
    handle_info(Msg, State);

websocket_info({incoming, Packet = ?CONNECT_PACKET(ConnPkt)}, State) ->
    Serialize = emqx_frame:serialize_opts(ConnPkt),
    NState = State#state{serialize = Serialize},
    handle_incoming(Packet, cancel_idle_timer(NState));

websocket_info({incoming, Packet}, State) ->
    handle_incoming(Packet, State);

websocket_info({outgoing, Packets}, State) ->
    return(enqueue(Packets, State));

websocket_info({check_gc, Stats}, State) ->
    return(check_oom(run_gc(Stats, State)));

websocket_info(Deliver = {deliver, _Topic, _Msg},
               State = #state{active_n = ActiveN}) ->
    Delivers = [Deliver|emqx_misc:drain_deliver(ActiveN)],
    with_channel(handle_deliver, [Delivers], State);

websocket_info({timeout, TRef, limit_timeout},
               State = #state{limit_timer = TRef}) ->
    NState = State#state{sockstate   = running,
                         limit_timer = undefined
                        },
    return(enqueue({active, true}, NState));

%%%函数websocket_info用于处理WebSocket相关的信息和事件。其中涉及到请求调用（call）、广播（cast）、传入消息（incoming）、传出消息（outgoing）、检查垃圾回收（check_gc）以及交付消息（deliver）等情况。具体的处理方式可能需要根据实际需求进行实现。

websocket_info({timeout, TRef, Msg}, State) when is_reference(TRef) ->
    handle_timeout(TRef, Msg, State);

websocket_info({shutdown, Reason}, State) ->
    shutdown(Reason, State);

websocket_info({stop, Reason}, State) ->
    shutdown(Reason, State);

websocket_info(Info, State) ->
    handle_info(Info, State).

websocket_close({_, ReasonCode, _Payload}, State) when is_integer(ReasonCode) ->
    websocket_close(ReasonCode, State);
websocket_close(Reason, State) ->
    ?LOG(debug, "Websocket closed due to ~p~n", [Reason]),
    handle_info({sock_closed, Reason}, State).

%%%函数websocket_close用于处理WebSocket连接关闭的情况，根据传入的关闭原因（ReasonCode或Reason）执行相应的操作。在示例中，会打印一条调试日志并将关闭事件传递给handle_info函数进行处理。

terminate(Reason, _Req, #state{channel = Channel}) ->
    ?LOG(debug, "Terminated due to ~p", [Reason]),
    emqx_channel:terminate(Reason, Channel);

terminate(_Reason, _Req, _UnExpectedState) ->
    ok.
%%%函数terminate用于在WebSocket处理器终止时进行清理工作。根据传入的终止原因（Reason），执行相应的操作，如记录日志或终止相关的通道。
~~~

**（3）DGIOT的初始化端口**
~~~erl
-define(CM, emqx_cm).
-define(ChanInfo,#{conninfo =>
                   #{socktype => tcp,
                     peername => {{127,0,0,1}, 5000},
                     sockname => {{127,0,0,1}, 1883},
                     peercert => nossl,
                     conn_mod => emqx_connection,
                     receive_maximum => 100}}).

%%%peername:对等端的地址和端口号; sockname：套接字的地址和端口号
~~~


### 2。APIS：（src/emqx_access_control.erl）
TODO


### 4.实例IOT部署（TODO）
缺乏实际的硬件部署，考虑虚拟串口部署：https://gitee.com/dgiiot/dgiot/wikis/%E5%BF%AB%E9%80%9F%E6%8E%A5%E5%85%A5/%E8%99%9A%E6%8B%9F%E7%94%B5%E8%A1%A8%E6%8E%A5%E5%85%A5/%E8%99%9A%E6%8B%9F%E4%B8%B2%E5%8F%A3