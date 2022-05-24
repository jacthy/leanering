### go语言文件读取与bufio包缓冲
- 引用 [原文链接](https://mp.weixin.qq.com/s/rjQ9_8TxfHXpZF4B1gj32w?st=9CEAEE1D4DDED2A3D0F15C5750E5C2335145112E3615BDA39DF7CFEB4F310AC74F7C98463CE4FF353D8F511EFE7B5D7952DB8F3DDC72A737A35A138C7931EAFF46C0C801BAC82DF21B57B85651E84C9CC5E39BB7AC1AAD06A815DBE46FE20D18F0FC4F878AFA7908F0D8F8FFC479BE038BA08F8B926616606F3821D5E29A85F12EB5854B9EABE669F3D5AFD602806CE4C6FAFFA5BBEEA2463506D9F7D9D8B5431B297F9E93C0B79142FD3DD1935B4DD71A738EB309868C0677B74F5814DB179CF43310C5F768E0F17DA69F12C472DCC1914A2AC95F427E0AEF143E4F7A7C6518&vid=1688854319363129&cst=2934E039D1A912A0FEAD12D40231342C6C3465C4DD04FF6FDA4F155E54A6CE6F36353FC057326C833FEEFF481E60FCB5&deviceid=4b2689bf-2267-4e35-9182-f101532c0144&version=4.0.6.99101&platform=mac)

- 普通读取文件方式:
    ```go
    //以读写模式打开文件
    fd, _ := os.OpenFile(filename, os.O_RDWR, 0666)
    b := make([]byte, 2)
    // 从文件中读取最多len(b)个字节的内容到b中
    n, err := fd.Read(b)
    ```
    - 这种方式相当于直接从磁盘文件中读取2个字节内容，如果频繁读取，要读磁盘io的话肯定很不理想
- 通过缓冲区读取文件的方式：
    ```go
    //以读写模式打开文件
    fd, _ := os.OpenFile(filename, os.O_RDWR, 0666)

    //将fd包装到buffer Reader中
    bufioReader := bufio.NewReader(fd)

    p := make([]byte, 2)
    n, _ := bufioReader.Read(p)
    ```
    - 这种方式读取是通过缓冲区读取，bufio包默认缓冲区大小为4k，缓冲区其实就是4k的数组，先读取文件到这个数组，然后再给变量读
        - 主要面临的场景有三种：
            - 读取长度>=缓存区，这时缓存没有意义，直接读文件就好了
            - 缓冲区缓存数为0时，这时有读取过来，缓冲区就会先读满，然后返回所需读取的字节数n，并把已读指针r指向n
            - 当读取过程中，读取数>缓存数(w-r)时，缓冲区写指针w会等于读指针r，此时将r和w都置0重新读，然后重新读一大块数据继续返回
    - 读取到指定字符，即类似按行读取，这种方式又会有不同的处理方式,主要场景：
        - 从缓冲区读取到指定字符，正常返回数据即可
        - 若缓冲区0 < r < w < len时，读取未能查到对应字符，则将arr[r:w]的数据移动到arr[0:w-r]及移动到最前，然后读取数据到缓冲区，再从w-r处开始查看找指定字符。
        - 若缓冲区满了也没找到指定字符，则将缓冲区数据移动到暂存区，再读取数据继续查找，重复这个动作直到找到字符就将暂存区的所有数据返回