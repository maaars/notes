## Go读写文本及二进制文件

对于文件来说，其中的数据就是byte，无所谓文本还是二进制。当提及文本和二进制时，是指对这些bytes的处理方式，比如:`0x31`这是一个字节，如果当成字符来读是数字字符为`1`, 如果当成`short`来读则为数字`49`.

> 文本文件: 文件中的每个字节都当作字符来解释 二进制文件: 文件中的字节不当作字符解释，具体怎么处理，与其被使用到的上下文有关，比如–把每4个字节当作一个`int`处理，把每`40`个字节当作一个结构体处理等

文本及二进制的区别在对待内存数据的编解码方式上

1. 写文件: 内存中的数据根据自定义的规则变成bytes，依次写入到文件中
2. 读文件: 按照自定义规则，从文件上读取bytes，将bytes进行转换，得到想要的数据类型

`Go`中读取文件有几个package–`os`、`ioutil`和`bufferio`, 这些是对所有文件都适用(包括网络IO文件)

## 文本文件读写

写数据: 数据转成对就的文本表示，如`123` ==> `'123'`, 然后对每个字符采用`特定编码`转成bytes 读数据: 读取bytes, 采用`对应的解码方式`转成字符，根据对应的字符生成相应数据类型的数据

由于数据都转成文本string, 而string转成bytes，因此使用package – `strconv`, `[]byte`和`string`

1. `strconv`: base types <==> string
2. `[]byte{str}`: string ==> byte slice
3. `string(byteArray[:]`: byte slice ==> string

```
func txt_write() {
    
    // 1. create bytes with string
    data := []byte{"hello world\n"}

    // 2. create file to write
    file, _ := os.Create("data.txt")
    defer file.Close()
    bytes_written, _ := file.write(data)

    // 
    fmt.Printf("Wrote %d bytes to file \n", bytes)
}

func txt_read() {

    // 1. open file to read
    file, _ := os.Open("data.txt")
	defer file.Close()

    // 2. read all bytes
    stats, _ := file.Stat()
	buf := make([]byte, 1024)

    var size int64 = stats.Size()
    bytes := make([]byte, size)

    bufr := bufio.NewReader(file)
    _, _ := bufr.Read(bytes)
    
    // 3. convert bytes to string
    fmt.Println(string(bytes))
}


```

## 二进制文件读写

写数据: 将对象(struct)根据自定义的规则将转成bytes 读数据: 每次读取一段bytes, 对这一段bytes，按照对应的规则将其中的不同部分转成结构体中对应的数据

`Go`中有两个基础的package处理数据的二进制转换–`binary`和`gob`, 以下是具体的demo:

### DEMO1. 基本数据类型的二进制

```
func bin_write() {
	
	t := time.Now().Nanosecond()
	fp, _ := os.Create(path.Join("bin", "numbers.binary"))
	defer fp.Close()

	rand.Seed(int64(t))

	buf := new(bytes.Buffer)
	for i := 0; i < 10; i++ {
		binary.Write(buf, binary.LittleEndian, int32(i))
		fp.Write(buf.Bytes())
	}

    // bin file contains: 0~9
}

func bin_read() {

	fp, _ := os.Open(path.Join("bin", "numbers.binary"))
	defer fp.Close()

	data := make([]byte, 4)
	var k int32
	for {
		data = data[:cap(data)]

        // read bytes to slice
		n, err := fp.Read(data)
		if err != nil {
			if err == io.EOF {
				break
			}
			fmt.Println(err)
			break
		}

        // convert bytes to int32
		data = data[:n]
		binary.Read(bytes.NewBuffer(data), binary.LittleEndian, &k)
		fmt.Println(k)
	}
}


```

### DEMO2 struct类型的二进制

```
type MyData struct {
	_ [1]byte
	Y int32
	X int32
	Z int32
}

func struct_write() {
	fp, _ := os.Create(path.Join("bin", "struct.binary"))
	defer fp.Close()

    // 将结构体转成bytes, 按照字段的声明顺序，但是"_"被放在最后
	data := &MyData{X:1, Y:2, Z:3}
	buf := new(bytes.Buffer)
	binary.Write(buf, binary.LittleEndian, data)

    // 将bytes写入文件
	fp.Write(buf.Bytes())
	fp.Sync()
}

func struct_read() {

	fp, _ := os.Open(path.Join("bin", "struct.binary"))
	defer fp.Close()

    // 创建byte slice, 以读取bytes. 此处MyData的size为16，因为有字节对齐
	dataBytes := make([]byte, unsafe.Sizeof(MyData{}))
	data := MyData{}
	n, _ := fp.Read(dataBytes)
	dataBytes = dataBytes[:n]

    // 将bytes转成对应的struct
	binary.Read(bytes.NewBuffer(dataBytes), binary.LittleEndian, &data)
	fmt.Println(data)
}


```

## 备注

1. 参考链接: [Parsing binary files in Go](https://www.jonathan-petitcolas.com/2014/09/25/parsing-binary-files-in-go.html)
2. 以上代码的测试环境为`go1.8.3 adm64/darwin`
3. `gob`是`Golang`自带的以二进制形式序列化和反序列化的工具，主要是其编码/解码规则，以及对struct的格式要求，后续将专门介绍