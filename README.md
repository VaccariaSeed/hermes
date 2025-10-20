# hermes
基于TCP和UDP的信息通讯交互框架；Information Communication Interaction Framework Based on TCP and UDP

## TCP通讯框架;TCP communication framework

#### 服务端;server
```go
//创建消息处理器
var _ vtcp.STCPServerHandle = (*ServerMsgHandle)(nil)

type ServerMsgHandle struct{}

func (s ServerMsgHandle) ClientLoginTimeOut(clientIpPort string) {
    fmt.Println(clientIpPort + "客户端连接超时")
}

func (s ServerMsgHandle) ClientConnected(clientAddress, clientId string) {
    fmt.Println(clientAddress + "," + clientId + "客户端建立连接")
}

func (s ServerMsgHandle) ClientApplyDisConnected(clientId string) {
    fmt.Println(clientId + "客户端主动断开连接")
}

func (s ServerMsgHandle) ClientAbnormalDisConnected(clientId string, err error) {
    fmt.Println(clientId + "客户端连接异常关闭")
}

//客户端携带必要信息进行登录，该方法返回true则表示允许连接，否则拒绝连接
func (s ServerMsgHandle) ClientLogin(clientId string, username string, password string, loginTs int64) bool {
    fmt.Println("---------客户端登录-----------")
    fmt.Println(clientId)
    fmt.Println(username)
    fmt.Println(password)
    fmt.Println(loginTs)
    fmt.Println("--------------------")
    return true
}

//如果收到的消息找不到指定的topic，将会进入这里，返回的消息就是回复的信息，responseTopic可以为默认值，为默认值则表示按照收到消息的topic发送
func (s ServerMsgHandle) DefaultMsgHandler(clientId string, topic string, msg []byte) (responseTopic string, responseMsg []byte, err error) {
    fmt.Println("----------收到客户端消息----------")
    fmt.Println(clientId)
    fmt.Println(topic)
    fmt.Println(string(msg))
    return "", nil, errors.New("test error")
}

//用于添加topic和及其对应的处理器，如果clientId为“”，“*”,则表示适配所有客户端
func (s ServerMsgHandle) AppendMessageHandleFunc(msgHandle *vtcp.ServerMsgHandle) {
    msgHandle.Append("客户端1", "test", testFunc)
    msgHandle.Append("*", "test", testFunc)
    return
}

func testFunc(clientId string, topic string, msg []byte) (responseTopic string, responseMsg []byte, err error) {
    fmt.Println("----------test topic 收到客户端消息----------")
    fmt.Println(clientId)
    fmt.Println(topic)
    fmt.Println(string(msg))
    return "", nil, errors.New("return a test error")
}

//创建服务端
config := &vtcp.ConnectOpt{
    Port:           端口号,
    ConnectTimeout: 连接超时时间,
    WriteTimeout:   写超时,
    ReadTimeout:    读超时,
}
sm := &ServerMsgHandle{}
server, err = vtcp.NewSwiftTCPServer(config, sm)
if err != nil {
    fmt.Println(err)
    return
}
_ = server.Open()

//向指定客户端发送消息
err := server.Send("测试客户端", "topic", "message")
//向指定客户端发送消息并等待客户端的回复
clientResponse, err := server.SendOnIdempotent("客户端", "topic", "message", timeout)
```

#### 客户端;client
```go
var _ vtcp.TCPClientHandle = (*ClientHandler)(nil)

type ClientHandler struct{}

func (c ClientHandler) LoseConnect(err error) {
    fmt.Println("客户端连接失败：" + err.Error())
}

func (c ClientHandler) Connected(localAddr string, remoteAddr string) {
    fmt.Println("客户端连接成功")
}

func (c ClientHandler) LoseLogin(err error) {
    fmt.Println("客户端登录失败：" + err.Error())
}

func (c ClientHandler) AppendHandle(opt *vtcp.ClientHandleOpt) {
    opt.Append("topic", testClientFunc)
    return
}

func (c ClientHandler) DefaultDataRequestHandler(topic string, data []byte) (responseTopic string, responseData []byte, err error) {
    fmt.Println("--------收到主站发过来的消息------------")
    fmt.Println(topic)
    fmt.Println(string(data))
    return "", nil, errors.New("return a error")
}
// 创建客户端
ch := &ClientHandler{}
client = &vtcp.TidingsTCPClient{
    Ip:             "127.0.0.1",
    Port:           9970,
    ConnectTimeout: 3,
    WriteTimeout:   3,
    ReadTimeout:    3,
    AutoConnect:    true,
    ClientId:       "测试客户端",
    Username:       "admin",
    Password:       "admin@123",
    ClientHandle:   ch,
    HeartBeatTimer: 5,
}
err = client.Connect()
if err != nil {
    fmt.Println("connect err:", err)
}
//向主站发送消息
err := client.SendMsg("topic", "message")
//向主站发送消息并等待回复
response, err := client.SendOnIdempotent("topic", "message"， timeout)
```

## uds 通讯；uds  communication framework
通过*.sock进行通讯。分为Master和slave，只能一对一通讯。
Sock属性是*.sock的名字，master和slave需要保持一致。

Communicate through *. sock. Divided into master and slave, can only communicate one-on-one.
The Sock attribute is the name of *. sock, and master and slave need to be consistent.

#### 创建Master; create master
```go
func clientClosed(err error) {
    fmt.Println("client closed:", err)
}

func slaveMessageReceiveHandle(msg []byte) []byte {
    fmt.Println("收到客户端的消息：" + string(msg) + "\n\n")
    return nil
}

Master = vuds.UDSClient{
		Sock:                       "uds",
		ClientType:                 vuds.MASTER,
		RetrySize:                  10,
		RetryDelay:                 3,
		ReadTimeout:                10,
		UnexpectedCloseHandler:     clientClosed,
		SlaveMessageReceiveHandler: slaveMessageReceiveHandle,
	}
err := Master.Connect()

//发送消息
err := Master.Send([]byte("wo shi master"))
//发送消息并等待回复
resp, err := Master.SendAndWaitForReply([]byte("message"), timeout)
//关闭
Master.Close()
```
#### 创建slave
与**创建Master**相同，只是ClientType变更为vuds.SLAVE

Same as creating Master * *, except that the Client Type has been changed to vuds.SLAVE