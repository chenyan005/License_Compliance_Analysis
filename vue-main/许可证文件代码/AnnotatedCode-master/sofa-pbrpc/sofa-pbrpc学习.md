# 客户端请求server
以echo为例
1. Stub.Echo
2. channel.callMethod(echo_method_id, request, response, done)
3. 根据done == null来判断是否为sync
4. RpcClientImpl.callMethod(request, response, cntl)
   * 封装rpcmeta
5. RpcClientStream.call_method
6. RpcMessageStream.async_send_message()


客户端异步发送后的接收回复的处理模块：RpcClientStream.on_received
关于接收后端回复这块比较复杂，首先有个继承关系：
RpcClientStream -> RpcMessageStream -> RpcByteStream

首先 在RpcByteStream的async_read_some函数中，定义callback为on_read_some。

这个on_read_some在RpcMessageStream中实现了。在这个函数里面对原始的字节消息进行了分割、处理后，才交给RpcClientStream.on_received处理。





然后调用了RpcController::Done() 对结果进行处理，值得注意的是，

这里是_done_callbacks是个双端队列deque，但实际上sofa-pbrpc把它当做stack用了
即，最先放进去的，最后取出来。这是因为第一个放进去的是用户定义的done，所以需要等后面的把response赋值了之后，
才能执行，因此必须最后一个执行

那么，都会放进去什么呢？在什么时间放呢？

针对于simple_rpc_channel来说，
目前来说，只会有两个地方放，
simple_rpc_channel_impl里面会把自己的DoneCallback放进去

第二个 rpc_client_impl.c里面，也会把自己的DoneCallback放进去



server接收消息，交给BinaryRpcRequest::ProcessRequest进行处理


然后调用RpcRequest::CallMethod
封装done
Service()->CallMethod就是执行真正的逻辑函数，处理完之后，在里面必须调用done->Run()才可以
在done中会进行下一步的response动作