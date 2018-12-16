# 从零开始写Socket Server（1）： Socket-Client框架

在golang中，网络协议已经被封装的非常完好了，想要写一个Socket的Server，我们并不用像其他语言那样需要为socket、bind、listen、receive等一系列操作头疼，只要使用Golang中自带的net包即可很方便的完成连接等操作~

在这里，给出一个最最基础的基于Socket的Server的写法：

```
package main
import (
	"fmt"
	"net"
	"log"
	"os"
)


func main() {

//建立socket，监听端口
	netListen, err := net.Listen("tcp", "localhost:1024")
	CheckError(err)
	defer netListen.Close()

	Log("Waiting for clients")
	for {
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}

		Log(conn.RemoteAddr().String(), " tcp connect success")
		handleConnection(conn)
	}
}
//处理连接
func handleConnection(conn net.Conn) {

	buffer := make([]byte, 2048)

	for {

		n, err := conn.Read(buffer)

		if err != nil {
			Log(conn.RemoteAddr().String(), " connection error: ", err)
			return
		}


		Log(conn.RemoteAddr().String(), "receive data string:\n", string(buffer[:n]))

	}

}
func Log(v ...interface{}) {
	log.Println(v...)
}

func CheckError(err error) {
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}
}
```

唔，抛除Go语言里面10行代码有5行error的蛋疼之处,你可以看到，Server想要建立并接受一个Socket，其核心流程就是：

```
netListen, err := net.Listen("tcp", "localhost:1024") 

conn, err := netListen.Accept() 

n, err := conn.Read(buffer) 
```

这三步，通过Listen、Accept 和Read，我们就成功的绑定了一个端口，并能够读取从该端口传来的内容~

Server写好之后，接下来就是Client方面啦，我手写一个HelloWorld给大家：

```
package main

import (
	"fmt"
	"net"
	"os"
)

func sender(conn net.Conn) {
		words := "hello world!"
		conn.Write([]byte(words))
	fmt.Println("send over")

}



func main() {
	server := "127.0.0.1:1024"
	tcpAddr, err := net.ResolveTCPAddr("tcp4", server)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}

	conn, err := net.DialTCP("tcp", nil, tcpAddr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}


	fmt.Println("connect success")
	sender(conn)

}

```

可以看到，Client这里的关键在于

```
tcpAddr, err := net.ResolveTCPAddr("tcp4", server)  
conn, err := net.DialTCP("tcp", nil, tcpAddr)  
```

这两步，主要是负责解析端口和连接~

运行结果，server

```
marsdeMacBook-Pro:webServer mars$ ./webserver
2018/06/13 15:14:58 Waiting for clients
2018/06/13 15:15:08 127.0.0.1:65325  tcp connect success
2018/06/13 15:15:08 127.0.0.1:65325 receive data string:
 hello world!
2018/06/13 15:15:08 127.0.0.1:65325  connection error:  EOF

```

client

```
marsdeMacBook-Pro:webServer mars$ ./webclient
connect success
send over
```

到这里，一个最基础的使用Socket的Server-Client框架就出来啦~

如果想要让Server能够响应来自不同Client的请求，我们只要在Server端的代码的main入口中，

在 handleConnection(conn net.Conn) 这句代码的前面加上一个 go，就可以让服务器并发处理不同的Client发来的请求啦



# 从零开始写Socket Server（2）： 自定义通讯协议

​	在上一章我们做出来一个最基础的demo后，已经可以初步实现Server和Client之间的信息交流了~ 这一章我会介绍一下怎么在Server和Client之间实现一个简单的通讯协议，从而增强整个信息交流过程的稳定性。

​        在Server和client的交互过程中，有时候很难避免出现网络波动，而在通讯质量较差的时候，Client有可能无法将信息流一次性完整发送，最终传到Server上的信息很可能变为很多段。

Server端要怎么判断收到的消息是否完整呢？~

唔，答案就是这篇文章的主题啦：在Server和Client交互的时候，加入一个通讯协议（protocol），让二者的交互通过这个协议进行封装，从而使Server能够判断收到的信息是否为完整的一段。（也就是解决分包的问题）

​        因为主要目的是为了让Server能判断客户端发来的信息是否完整，因此整个协议的核心思路并不是很复杂：

协议的核心就是设计一个头部（headers），在Client每次发送信息的时候将header封装进去，再让Server在每次收到信息的时候按照预定格式将消息进行解析，这样根据Client传来的数据中是否包含headers，就可以很轻松的判断收到的信息是否完整了~

​        如果信息完整，那么就将该信息发送给下一个逻辑进行处理，如果信息不完整（缺少headers），那么Server就会把这条信息与前一条信息合并继续处理。

​        下面是协议部分的代码,主要分为数据的封装（Enpack）和解析（Depack）两个部分，其中Enpack用于Client端将传给服务器的数据封装，而Depack是Server用来解析数据，其中Const部分用于定义Headers，HeaderLength则是Headers的长度，用于后面Server端的解析。这里要说一下ConstMLength,这里代表Client传入信息的长度，因为在golang中，int转为byte后会占4长度的空间，因此设定为4。每次Client向Server发送信息的时候，除了将Headers封装进去意以外，还会将传入信息的长度也封装进去，这样可以方便Server进行解析和校验。

```
//通讯协议处理
package protocol

import (
	"bytes"
	"encoding/binary"
)
const (
	ConstHeader         = "Headers"
	ConstHeaderLength   = 7
	ConstMLength = 4
)

//封包
func Enpack(message []byte) []byte {
	return append(append([]byte(ConstHeader), IntToBytes(len(message))...), message...)
}

//解包
func Depack(buffer []byte, readerChannel chan []byte) []byte {
	length := len(buffer)

	var i int
	for i = 0; i < length; i = i + 1 {
		if length < i+ConstHeaderLength+ConstMLength {
			break
		}
		if string(buffer[i:i+ConstHeaderLength]) == ConstHeader {
			messageLength := BytesToInt(buffer[i+ConstHeaderLength : i+ConstHeaderLength+ConstMLength])
			if length < i+ConstHeaderLength+ConstLength+messageLength {
				break
			}
			data := buffer[i+ConstHeaderLength+ConstMLength : i+ConstHeaderLength+ConstMLength+messageLength]
			readerChannel <- data

		}
	}

	if i == length {
		return make([]byte, 0)
	}
	return buffer[i:]
}

//整形转换成字节
func IntToBytes(n int) []byte {
	x := int32(n)

	bytesBuffer := bytes.NewBuffer([]byte{})
	binary.Write(bytesBuffer, binary.BigEndian, x)
	return bytesBuffer.Bytes()
}

//字节转换成整形
func BytesToInt(b []byte) int {
	bytesBuffer := bytes.NewBuffer(b)

	var x int32
	binary.Read(bytesBuffer, binary.BigEndian, &x)

	return int(x)
}
```

协议写好之后，接下来就是在Server和Client的代码中应用协议啦，下面是Server端的代码，主要负责解析Client通过协议发来的信息流：

```
package main  
  
import (  
    "protocol"  
    "fmt"  
    "net"  
    "os"  
)  
  
func main() {  
    netListen, err := net.Listen("tcp", "localhost:6060")  
    CheckError(err)  
  
    defer netListen.Close()  
  
    Log("Waiting for clients")  
    for {  
        conn, err := netListen.Accept()  
        if err != nil {  
            continue  
        }  
  
        //timeouSec :=10  
        //conn.  
        Log(conn.RemoteAddr().String(), " tcp connect success")  
        go handleConnection(conn)  
  
    }  
}  
  
func handleConnection(conn net.Conn) {  
  
  
    // 缓冲区，存储被截断的数据  
    tmpBuffer := make([]byte, 0)  
  
    //接收解包  
    readerChannel := make(chan []byte, 16)  
    go reader(readerChannel)  
  
    buffer := make([]byte, 1024)  
    for {  
    n, err := conn.Read(buffer)  
    if err != nil {  
    Log(conn.RemoteAddr().String(), " connection error: ", err)  
    return  
    }  
  
    tmpBuffer = protocol.Depack(append(tmpBuffer, buffer[:n]...), readerChannel)  
    }  
    defer conn.Close()  
}  
  
func reader(readerChannel chan []byte) {  
    for {  
        select {  
        case data := <-readerChannel:  
            Log(string(data))  
        }  
    }  
}  
  
func Log(v ...interface{}) {  
    fmt.Println(v...)  
}  
  
func CheckError(err error) {  
    if err != nil {  
        fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())  
        os.Exit(1)  
    }  
}  
```

然后是Client端的代码，这个简单多了，只要给信息封装一下就可以了~：

```
package main  
import (  
"protocol"  
"fmt"  
"net"  
"os"  
"time"  
"strconv"  
  
)  
  
func send(conn net.Conn) {  
    for i := 0; i < 100; i++ {  
        session:=GetSession()  
        words := "{\"ID\":"+ strconv.Itoa(i) +"\",\"Session\":"+session +"2015073109532345\",\"Meta\":\"golang\",\"Content\":\"message\"}"  
        conn.Write(protocol.Enpacket([]byte(words)))  
    }  
    fmt.Println("send over")  
    defer conn.Close()  
}  
  
func GetSession() string{  
    gs1:=time.Now().Unix()  
    gs2:=strconv.FormatInt(gs1,10)  
    return gs2  
}  
  
func main() {  
    server := "localhost:6060"  
    tcpAddr, err := net.ResolveTCPAddr("tcp4", server)  
    if err != nil {  
        fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())  
        os.Exit(1)  
    }  
  
    conn, err := net.DialTCP("tcp", nil, tcpAddr)  
    if err != nil {  
        fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())  
        os.Exit(1)  
    }  
  
  
    fmt.Println("connect success")  
    send(conn)  
  
  
  
}  
```

​       这样我们就成功实现在Server和Client之间建立一套自定义的基础通讯协议啦，让我们运行一下看下效果：





# 从零开始写Socket Server（3）：对长、短连接的处理策略（模拟心跳）

​    通过前两章，我们成功是写出了一套凑合能用的Server和Client，并在二者之间实现了通过协议交流。这么一来，一个简易的socket通讯框架已经初具雏形了，那么我们接下来做的，就是想办法让这个框架更加稳定，茁壮~

​    作为一个可能会和很多Client进行通讯交互的Server，首先要保证的就是整个Server运行状态的稳定性，因此在和Client建立连接通讯的时候，确保连接的及时断开非常重要，否则一旦和多个客户端建立不关闭的长连接，对于服务器资源的占用是很可怕的。因此，我们需要针对可能出现的短连接和长连接，设定不同的限制策略。

** 针对短连接**，我们可以使用golang中的net包自带的timeout函数，一共有三个，分别是：

```
func (*IPConn) SetDeadline
func (c *IPConn) SetDeadline(t time.Time) error

func (*IPConn) SetReadDeadline
func (c *IPConn) SetReadDeadline(t time.Time) error

func (*IPConn) SetWriteDeadline
func (c *IPConn) SetWriteDeadline(t time.Time) error
```

​    如果想要给服务器设置短连接的timeout,我们就可以这么写：

```

netListen, err := net.Listen("tcp", Port)
	Log("Waiting for clients")
	for {
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}

		conn.SetReadDeadline(time.Now().Add(time.Duration(10) * time.Second))
```

​    这里的三个函数都是用于设置每次socket连接能够维持的最长时间，一旦超过设置的timeout后，便会在Server端自动断开连接。其中SetReadline, SetWriteline设置的是读取和写入的最长持续时间，而SetDeadline则同时包含了 SetReadline, SetWriteline两个函数。

​    通过这样设定，每个和Server通讯的Client连接时长最长也不会超过10s了~~

**    搞定短连接后，接下来就是针对长连接的处理策略了~~**

​    作为长连接，由于我们往往很难确定什么时候会中断连接，因此并不能像处理短连接那样简单粗暴的设定一个timeout就可以搞定，而在Golang的net包中，并没有针对长连接的函数，因此需要我们自己设计并实现针对长连接的处理策略啦~

​    针对socke长连接，常见的做法是在Server和Socket之间设计通讯机制，当两者之间没有信息交互时，双方便会定时发送数据包(心跳)，以维持连接状态。

​    这种方法是目前使用相对比较多的做法，但是开销相对也较大，特别是当Server和多个client保持长连接的时候，并发会比较高，考虑到公司的业务需求，我最后选择了逻辑相对简单，开销相对较小的策略：

​    当Server每次收到Client发到的信息之后，便会开始心跳计时，如果在心跳计时结束之前没有再次收到Client发来的信息，那么便会断开跟Client的连接。而一旦在设定时间内再次收到Client发来的信息，那么Server便会重置计时器，再次重新进行心跳计时，直到超时断开连接为止。

下面就是实现该计时的代码：

```

//长连接入口
func handleConnection(conn net.Conn,timeout int) {

	buffer := make([]byte, 2048)
	for {
		n, err := conn.Read(buffer)

		if err != nil {
			LogErr(conn.RemoteAddr().String(), " connection error: ", err)
			return
		}
		Data :=(buffer[:n])
		messnager := make(chan byte)
		//心跳计时
		go HeartBeating(conn,messnager,timeout)
		//检测每次Client是否有数据传来
		go GravelChannel(Data,messnager)
		Log( "receive data length:",n)
		Log(conn.RemoteAddr().String(), "receive data string:", string(Data))

	}
}

//心跳计时，根据GravelChannel判断Client是否在设定时间内发来信息
func HeartBeating(conn net.Conn, readerChannel chan byte,timeout int) {
		select {
		case fk := <-readerChannel:
			Log(conn.RemoteAddr().String(), "receive data string:", string(fk))
			conn.SetDeadline(time.Now().Add(time.Duration(timeout) * time.Second))
			//conn.SetReadDeadline(time.Now().Add(time.Duration(5) * time.Second))
			break
		case <-time.After(time.Second*5):
			Log("It's really weird to get Nothing!!!")
			conn.Close()
		}

}

func GravelChannel(n []byte,mess chan byte){
	for _ , v := range n{
		mess <- v
	}
	close(mess)
}


func Log(v ...interface{}) {
	log.Println(v...)
}
```

​    可以发现，Sender函数中time.Sleep阻塞的时间设定的比Server中的timeout短的时候，Client端的信息可以自由的发送到循环结束，而当我们设定Sender函数的阻塞时间较长时，就只能发出第一次循环的信息。

# 从零开始写Socket Server（4）：将运行参数放入配置文件（XML/YAML）

​    为了将我们写好的Server发布到服务器上，就要将我们的代码进行build打包，这样如果以后想要修改一些代码的话，需要重新给代码进行编译打包并上传到服务器上。 
   显然，这么做过于繁琐。。。因此常见的做法都是将Server运行中可能会频繁变更的变量、数值写入配置文件中，这样直接让程序从配置文件读取参数，避免对代码频繁的操作。 
   关于配置文件的格式，在这里推荐YAML 和XML~ XML是传统的配置文件写法，不过本人比较推荐yaml，他比XML要更加人性化，也更好写，关于yaml的详细信息可以参考： [yaml官网](http://yaml.org/))

   比如我们可以将Server监听的端口作为变量，写入配置文件 config.yaml 和 config.xml，放入代码的根目录下，这样当我们想要更换服务器端口的时候，只要在配置文件中修改port对应的值就可以拉。 config.xml内容如下：

```

<?xml version="1.0" encoding="UTF-8"?>
<Config1>GetConfig</Config1>
<Config2>THE</Config2>
<Config3>Information</Config3>
<Feature1>HereIsTEST1</Feature1>
<Feature2>1024</Feature2>
<Feature3>Feature23333</Feature3>
```

config.yaml内容如下：

```

Address: 172.168.0.1
Config1: Easy
Config2:
  Feature1: 2
  Feature2: [3, 4]
Port: :6060
Config4: IS
Config5: ATest
```









# 从零开始写Socket Server（5）：Server的解耦—通过Router+Controller实现逻辑分发

​       在实际的系统项目工程中中，我们在写代码的时候要尽量避免不必要的耦合，否则你以后在更新和维护代码的时候会发现如同深陷泥潭，随便改点东西整个系统都要变动的酸爽会让你深切后悔自己当初为什么非要把东西都写到一块去（我不会说我刚实习的时候就是这么干的。。。）

​       所以这一篇主要说说如何设计Sever的内部逻辑，将Server处理Client发送信息的这部分逻辑与Sevrer处理Socket连接的逻辑进行解耦～

​       这一块的实现灵感主要是在读一个HTTP开源框架： [Beego](https://github.com/astaxie/beego)  的源代码的时候产生的，Beego的整个架构就是高度解耦的





​       在这里，我们可以仿照Beego的架构，在Server内部加入一层Router,通过Router对通过Socket发来的信息通过我们设定的规则进行判断后，调用相关的Controller进行任务的分发处理。在这个过程中不仅Controller彼此独立，匹配规则和Controller之间也是相互独立的。

​       下面给出Router的实现代码，其中Msg的结构对应的是Json字符串，当然考虑到实习公司现在也在用这个，修改了一部分，不过核心思路是一样的哦：

```

import (
	"utils"
	"fmt"
	"encoding/json"
)

type Msg struct {
	Conditions   map[string]interface{} `json:"meta"`
	Content interface{}            `json:"content"`
}

type Controller interface {
	Excute(message Msg) []byte
}

var routers [][2]interface{}

func Route(judge interface{} ,controller Controller) {
	switch judge.(type) {
	case func(entry Msg)bool:{
		var arr [2]interface{}
		arr[0] = judge
		arr[1] = controller
		routers = append(routers,arr)
	}
	case map[string]interface{}:{
		defaultJudge:= func(entry Msg)bool{
			for keyjudge , valjudge := range judge.(map[string]interface{}){
				val, ok := entry.Meta[keyjudge]
				if !ok {
					return false
				}
				if val != valjudge {
					return false
				}
			}
			return true
		}
		var arr [2]interface{}
		arr[0] = defaultjudge
		arr[1] = controller
		routers = append(routers,arr)
		fmt.Println(routers)
		}
	default:
		fmt.Println("Something is wrong in Router")
	}
}


```

​      通过自定义接口Router，我们将匹配规则judge和对应的controller封装了进去，然后在Server端负责接收socket发送信息的函数handleConnection那里再实现Router内部的遍历即可：

```

for _ ,v := range routers{
		pred := v[0]
		act := v[1]
		var message Msg
		err := json.Unmarshal(postdata,&message)
		if err != nil {
			Log(err)
		}
		if pred.(func(entry Msg)bool)(message) {
			result := act.(Controller).Excute(message)
			conn.Write(result)
			return
		}
	}
```

​       这样Client每次发来信息，我们就可以让Router自动跟现有的规则进行匹配，最后调用对应的Controller进行逻辑的实现啦，下面给出一个controller的编写实例，这个Controll的作用是发来的json类型是mirror的时候，将Client发来的信息原样返回：

```
type MirrorController struct  {

}

func (this *MirrorController) Excute(message Msg)[]byte {
	mirrormsg,err :=json.Marshal(message)
	CheckError(err)
	return mirrormsg
}


func init() {
	var mirror 
	routers = make([][2]interface{} ,0 , 20)
	Route(func(entry Msg)bool{
		if entry.Meta["msgtype"]=="mirror"{
		return true}
		return  false
	},&mirror)
}
```



# 从零开始写Socket Server（6）【完结】：日志模块的设计与定时任务模块模块

作为一个Server，日志（Log）功能是必不可少的，一个设计良好的日志模块，不论是开发Server时的调试，还是运行时候的维护，都是非常有帮助的。

因为这里写的是一个比较简化的Server框架，因此我选择对Golang本身的log库进行扩充，从而实现一个简单的Log模块。

在这里，我将日志的等级大致分为Debug，Operating，Error 3个等级，Debug主要用于存放调试阶段的日志信息，Operateing用于保存Server日常运行时产生的信息，Error则是保存报错信息。

模块代码如下：

```

func LogErr(v ...interface{}) {

	logfile := os.Stdout
	log.Println(v...)
	logger := log.New(logfile,"\r\n",log.Llongfile|log.Ldate|log.Ltime);
	logger.SetPrefix("[Error]")
	logger.Println(v...)
	defer logfile.Close();
}

func Log(v ...interface{}) {

	logfile := os.Stdout
	log.Println(v...)
	logger := log.New(logfile,"\r\n",log.Ldate|log.Ltime);
	logger.SetPrefix("[Info]")
	logger.Println(v...)
	defer logfile.Close();
}

func LogDebug(v ...interface{}) {
	logfile := os.Stdout
	log.Println(v...)
	logger := log.New(logfile,"\r\n",log.Ldate|log.Ltime);
	logger.SetPrefix("[Debug]")
	logger.Println(v...)
	defer logfile.Close();
}

func CheckError(err error) {
	if err != nil {
		LogErr(os.Stderr, "Fatal error: %s", err.Error())
	}
}
```

注意这里log的输出我使用的是stdout，因为这样在Server运行的时候可以直接将log重定向到指定的位置，方便整个Server的部署。不过在日常开发的时候，为了方便调试代码，我推荐将log输出到指定文件位置下，这样在调试的时候会方便很多（主要是因为golang的调试实在太麻烦，很多时候都要依靠打log的时候进行步进。便于调试的Log模块代码示意：

```

func Log(v ...interface{}) {

	logfile := os.OpenFile("server.log",os.O_RDWR|os.O_APPEND|os.O_CREATE,0);
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		return    }
	log.Println(v...)
	logger := log.New(logfile,"\r\n",log.Ldate|log.Ltime);
	logger.SetPrefix("[Info]")
	logger.Println(v...)
	defer logfile.Close();
}
```



然后就是计时循环模块啦，日常运行中，Server经常要执行一些定时任务，比如隔一定时间刷新后台，隔一段时间自动刷新爬虫等等，在这里我设计了一个Task接口，通过类似于TaskList的的方式将所有定时任务注册后统一执行，代码如下：

```

type DoTask interface {
	Excute()
}

var tasklist []interface{}

func AddTask(controller DoTask) {
	var arr interface{}
	arr = controller
	tasklist = append(tasklist,arr)
	fmt.Println(tasklist)
	}
```

在这里以一个定时报时任务作为例子，注意，在这里我定义了一个channel用于阻塞main函数，否则main函数这里直接就结束了没法看到结果，实际使用中，通过类似的方法保持server一直运行就可以啦：

```

var channel = make(chan int, 10)
type Task1 struct {}
func (this * Task1)Excute() {
	for {
		time.Sleep(2 * time.Second)
		fmt.Println("this is task1") 
	}
}

type Task2 struct {}

func (this * Task2)Excute() {
	 for {
		time.Sleep(4 * time.Second)
		fmt.Println("this is task2")
	 }
}

func main(){
	var task1 Task1
	var task2 Task2
	tasklist = make([]interface{} ,0 , 20)
	AddTask(&task1)
	AddTask(&task2)
	m :=0
	for _, _ = range tasklist {
		m = m+1
	}

	for _, v := range tasklist{
		go v.(DoTask).Excute()
	}
	<-channel
}
```

运行结果如下，2秒间隔的task1执行两次后，正好完成4秒间隔的task2，证明定时接口成功被循环执行：