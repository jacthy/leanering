### go语言中error的分类与用法
- 原文引用：极客时间中的课程《Go error处理最佳实践》
- 前言：本文要讨论的就是go中error的基本原理/类型，以及最重要的几个问题：1.go代码开发中因为要处理太多的error，会导致充斥大量的if err!=nil的处理；2.大量错误日志信息的打印，容易泛滥，且信息可能不完整（如哨兵错误，只有"open fail"，没有详细信息）不能很好的追溯；3.在很多需要判断错误类型中过度关注内容，导致上下层业务耦合严重；4.日志内容是割裂的，如：a1行打印的日志和b2行的日志实际日志隔了很多行，但实际上函数行为上它们是连续的。
#### go错误处理的设计理念
1. 与其他语言相关设计的比较
    - go与c++的区别：go函数可以有多个返回值，通常是func add(a,b int)(int,error)这样的形式，便于判断错误的处理err!=nil就是报错，而c++函数：int add(int a, int b)只能返回单值，就需要根据返回值判断状态，不好区分函数执行结果
    - go error与java exception，java的exception主要是基于goto这种实现，exception会带有堆栈信息，需要通过try-catch捕获，使用上和效果来说，exception都比较友好，但是使用格式固定，且容易泛滥打印大量错误信息
2. go error与panci的设计
    - go error设计的理念是及时处理，一个函数返回error就应该被及时处理，这个error是轻量的
    - go panic是程序遇到无法执行的错误时使用，非常不建议当exception使用，尽量少的主动调用panic
3. error处理上的一些建议
    - 应该能够支撑完整的错误追踪能力，也不应抛过多无用的信息，对调试无用的信息其实就是噪音信息，应该被质疑
    - 应该只处理一次(理想情况下)，要么能cover住不影响流程而不处理，要么返回err并退出，不应该打印再退出这样两个步骤
#### go error的分类与特点
- error设计：error are value,就是一个含错误内容值的接口
    ```go
    type error interface {
        Error() string
    }
    ```
- error的模式
    - sentinel error哨兵错误,通常用于等值判断，如io的EOF/ErrShortBuffer等,读文件/缓冲区时返回的错误类型；带来的问题是：1.加大接口的暴露面积，容易导致循环依赖;2.依赖error的值进行判断，所以二次封装接口时，不能轻易修改它(会破坏原值)，可能只能打印记录
        ```go
        var EOF = errors.New("EOF")
        var ErrUnexpectedEOF = errors.New("unexpected EOF")
        var ErrShortBuffer = errors.New("short buffer")
        ``` 
    - errors Type 类型错误, 如os.PathError,比哨兵错误的好处是只附加了信息(保留了原上下文)，没有破坏原错误，保留了原错误。判断可依赖于if err.(type)==os.PathError也可err.Err的方式 强依赖于错误，对业务侵入大，调用方容易产生强耦合
        ```go
        // os.PathError
        type PathError struct {
            Op   string
            Path string
            Err  error
        }

        func (e *PathError) Error() string { return e.Op + " " + e.Path + ": " + e.Err.Error() }
        ```
    - opaque errors不透明错误，调用者不关注它具体内容，只需要if err!=nil判断就可以了，有需要的如net.Error，其中timeOut()和temporary()是开放出去的行为，这样降低被强耦合的可能。但net.Error抛出error可能导致业务耦合（即没有遵循开闭原则）
        ```go
        // An Error represents a network error.
        type Error interface {
            error
            Timeout() bool   // Is the error a timeout?
            Temporary() bool // Is the error temporary?
        }

        type AddrError struct {
            Err  string
            Addr string
        }

        func (e *AddrError) Error() string {
            if e == nil {
                return "<nil>"
            }
            s := e.Err
            if e.Addr != "" {
                s = "address " + e.Addr + ": " + s
            }
            return s
        }

        func (e *AddrError) Timeout() bool   { return false }
        func (e *AddrError) Temporary() bool { return false }
        ```
    - wrap error包装错误，思想：对错误进行包装，会保留原err，只添加包装信息，不会改变原err。第三方包github.com/pkg/errors就很好的实现了这些功能，可以通过wrap()方法，对错误进行包装且带了堆栈信息，通过cause()方法获取原来的错误，直接打印可以将所有信息一次性打印出来，避免了割裂
        ```go
        func main() {
            _,err := ReadConfig()
            if err != nil {
                //org err: *os.PathError open /Users/js/.setting.xml: no such file or directory
                fmt.Printf("org err: %T %v\n",errors.Cause(err),errors.Cause(err))
                // wrap err: could not read config: open fail: open /Users/js/.setting.xml: no such file or directory
                fmt.Printf("wrap err: %v\n",err)
                // 详细堆栈信息
                fmt.Printf("stack trace: \n %+v\n",err)
            }
        }

        func ReadFile(path string)([]byte,error)  {
            f,err := os.Open(path)
            if err != nil {
                // 只对err进行包装，不破坏原错误，携带了附加的错误信息&堆栈信息
                return nil, errors.Wrap(err,"open fail")
            }
            defer f.Close()
            return nil, err
        }

        func ReadConfig() ([]byte, error) {
            home := os.Getenv("HOME")
            config,err:=ReadFile(filepath.Join(home,".setting.xml"))
            // 这里也只直接进行包装
            return config,errors.WithMessage(err,"could not read config")
        }
        ```
#### error处理优化的一个案例
- 统计io.Reader读取内容的行数的一个方法，大概实现会是这样的
    ```go
    func CountLines(r io.Reader)(int,error)  {
        var (
            br = bufio.NewReader(r)
            lines = 0
            err error
        )
        for  {
            _,err = br.ReadString('\n')//读取一行
            lines++
            if err != nil {//遇到错误就退出
                break
            }
        }
        if err != io.EOF {//如果不是eof错误表面读取出错
            return 0,nil
        }
        return lines,nil
    }
    ```
    其中出现了两次错误处理，是因为Reader.ReadString返回的是一个具体的错误，导致调用者需要去处理错误。而改进后的版本：
    ```go
    func CountLines2(r io.Reader)(int,error)  {
        sc := bufio.NewScanner(r)
        lines := 0
        for sc.Scan() {
            lines++
        }
        return lines, sc.Err()
    }
    ```
    其中Scanner.Scan的方法直接屏蔽了具体错误，只返回行为结构：即scan失败时(包括eof和err)返回false，而Scanner.Err也屏蔽了EOF错误，所以使得此处调用点能显得简洁。对比scanner和reader就是我们设计使应该考虑的，如果上层强依赖结果，可能需要我们返回具体值，反之我们就应该设计得更简洁，降低耦合
- 一个处理HTTP请求的例子
    ```go
    func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
        _, err := fmt.Fprintf(w, "http/1.1 %d %s\r\n", st.Code, st.Reason)
        if err != nil {
            return err
        }
        for _, h := range headers {
            _, err := fmt.Fprintf(w, "%s:%s\r\n", h.Key, h.Value)
            if err != nil {
                return err
            }
        }
        if _, err := fmt.Fprintf(w, "\r\n"); err != nil {
            return err
        }
        _, err = io.Copy(w, body)
        return err
    }
    ```
    其中因为要对w进行多次写，调用的是Fprintf这种写方法会返回err，必须要去处理，所以导致一堆err!=nil的处理，这种似乎没有什么好的方法去解决，你可能会说调用一个不会返回错误的方法不就可以了，但如果没有呢，rob pike(go创始人)提供了一种思路：
    ```go
    func WriteResponse2(w io.Writer, st Status, headers []Header, body io.Reader) error {
        ew := &errWrite{Writer:w}
        fmt.Fprintf(ew, "http/1.1 %d %s\r\n", st.Code, st.Reason)
        for _, h := range headers {
            fmt.Fprintf(ew, "%s:%s\r\n", h.Key, h.Value)
        }
        fmt.Fprintf(ew, "\r\n")
        io.Copy(ew, body)
        return ew.err
    }

    type errWrite struct {
        io.Writer
        err error
    }

    func (e *errWrite) Write(buf []byte)(int,error)  {
        if e.err != nil {
            return 0,e.err
        }
        n := 0
        n,e.err = e.Writer.Write(buf)
        return n,nil
    }
    ```
    其中for循环部分原来可以直接retrun的逻辑，会变成最后才return，但如果headers数组小（1万次循环才1毫秒不到），可以说都是小问题。在代码层面上，如果我们errWrite的调用点比较多，那代码的简洁度上的优化是巨大的。