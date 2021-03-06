# 以太坊 rlp 包的用法

RLP (递归长度前缀)提供了一种适用于任意二进制数据数组的编码，RLP 已经成为以太坊中对对象进行序列化的主要编码方式。RLP 的唯一目标就是解决结构体的编码问题；对原子数据类型（比如，字符串，整数型，浮点型）的编码则交给更高层的协议；以太坊中要求数字必须是一个大端字节序的、没有零占位的存储的格式（也就是说，一个整数 0 和一个空数组是等同的）。

```go
/*
rlp包用法
rlp目的是可以将常用的数据结构,uint,string,[]byte,struct,slice,array,big.int等序列化以及反序列化.
要注意的是rlp特别不支持有符号数的序列化
具体用法见下
*/
//编码
type TestRlpStruct struct {
    A      uint
    B      string
    C      []byte
    BigInt *big.Int
}

//rlp用法
func TestRlp(t *testing.T) {
    //1.将一个整数数组序列化
    arrdata, err := rlp.EncodeToBytes([]uint{32, 28})
    fmt.Printf("unuse err:%v\n", err)
    //fmt.Sprintf("data=%s,err=%v", hex.EncodeToString(arrdata), err)
    //2.将数组反序列化
    var intarray []uint
    err = rlp.DecodeBytes(arrdata, &intarray)
    //intarray 应为{32,28}
    fmt.Printf("intarray=%v\n", intarray)

    //3.将一个布尔变量序列化到一个writer中
    writer := new(bytes.Buffer)
    err = rlp.Encode(writer, true)
    //fmt.Sprintf("data=%s,err=%v",hex.EncodeToString(writer.Bytes()),err)
    //4.将一个布尔变量反序列化
    var b bool
    err = rlp.DecodeBytes(writer.Bytes(), &b)
    //b:true
    fmt.Printf("b=%v\n", b)

    //5.将任意一个struct序列化
    //将一个struct序列化到reader中
    _, r, err := rlp.EncodeToReader(TestRlpStruct{3, "44", []byte{0x12, 0x32}, big.NewInt(32)})
    var teststruct TestRlpStruct
    err = rlp.Decode(r, &teststruct)
    //{A:0x3, B:"44", C:[]uint8{0x12, 0x32}, BigInt:32}
    fmt.Printf("teststruct=%#v\n", teststruct)

}
```

# RLP 编码的五个规则

RLP 编码后的第一个字节很重要，根据输入的不同，第一个字节代表的含义也不同。

下表中列出了不同的规则对应的第一个字节的范围（tip: 一个字节 8 位，换成 16 进制是 2 位）

| 规则 | 规则一    | 规则二    | 规则三    | 规则四    | 规则五    |
| ---- | --------- | --------- | --------- | --------- | --------- |
| 范围 | 0x00~0x7f | 0x80~0xb7 | 0xb8~0xbf | 0xc0~0xf7 | 0xf8~0xff |


1. 规则一  
考虑到ASCII码的特殊性，如果输入的是ASCII码（值范围[0x00, 0x7f])，他的RLP编码就是他本身。

2. 规则二  
如果输入的长度是0-55字节，他的RLP编码包含一个单字节的前缀，后面跟着字符串本身，这个前缀的值是0x80加上字符串的长度。由于被编码的字符串最大长度是55=0x37,因此单字节前缀的最大值是0x80+0x37=0xb7，即编码的第一个字节的取值范围是[0x80, 0xb7]。（长度范围为什么是0-55，看完规则三就知道了）.

3. 规则三  
如果字符串的长度大于55个字节，它的RLP编码包含一个单字节的前缀，后面跟着字符串的长度，在后面跟着字符串本身。这个前缀的值是0xb7加上字符串长度的二进制形式的字节长度，说的有点绕，举个例子就明白了，例如一个字符串的长度是1024，它的二进制形式是10000000000，这个二进制形式的长度是2个字节，所以前缀应该是0xb7+2=0xb9，字符串长度1024=0x400，因此整个RLP编码应该是\xb9\x04\x00再跟上字符串本身。编码的第一个字节即前缀的取值范围是[0xb8, 0xbf]，因为字符串长度二进制形式最少是1个字节，因此最小值是0xb7+1=0xb8，字符串长度二进制最大是8个字节，因此最大值是0xb7+8=0xbf。（支持的最大长度是8个字节，2^32大小也够用了。另外规则二中为什么最大长度是55，因为55+8=64 -1，范围最大值刚好是0xbf)

4. 规则四  
如果一个列表的总长度（列表的总长度指的是它包含的项的个数加上它包含的各项的长度之和）是0-55字节，它的RLP编码包含一个单字节的前缀，后面跟着列表中各元素的RLP编码，这个前缀的值是0xc0加上列表的总长度。编码的第一个字节的取值范围是[0xc0, 0xf7].

5. 规则五  
如果一个列表的总长度大于55字节，它的RLP编码包含一个单字节的前缀，后面跟着列表的长度，后面再跟着列表中各元素项的RLP编码，这个前缀的值是0xf7加上列表总长度的二进制形式的字节长度。编码的第一个字节的取值范围是[0xf8, 0xff]。

RLP编码例子
 
1. 字符串 "dog" = [0x83, 'd', 'o', 'g' ] （规则二）

2. 列表 ["cat","dog"] = [0xc8, 0x83, 'c', 'a', 't', 0x83, 'd', 'o', 'g' ] （规则四）

空字符串 "" = 0x80 （规则二）

空列表 [] = [0xc0] （规则四）

整数 15('\x0f') = 0x0f （规则一）

整数 1024('\x04\00') = [0x82, 0x04, 0x00] （规则二）

列表 [ [], [[]], [ [], [[]] ] ] = [0xc7, 0xc0, 0xc1, 0xc0, 0xc3, 0xc0, 0xc1, 0xc0] （规则四）

字符串 "Lorem ipsum dolor sit amet, consectetur adipisicing elit" = [0xb8, 0x38, 'L', 'o', 'r', 'e', 'm', ' ', ... , 'e', 'l', 'i', 't']（规则三）