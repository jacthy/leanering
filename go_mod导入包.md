- 切换到项目地址执行go get
    ```
    cd your_project_path
    go get github.com/ahmetb/go-linq/v3@tag //不要@就拉最新
    ```
    理论上这时go.mod中应该有这个记录
    ```
    module go-paas

    go 1.14

    require (
        github.com/ahmetb/go-linq/v3 v3.2.0
        )
    ```
    没有的话需要自己加进去
- 到自己代码需要用到的包里导入对应的包，这时还是有可能不行
    ```
    import . "github.com/ahmetb/go-linq/v3"
    import . "github.com/ahmetb/go-linq/v3/tool"
    ```
    - 如果出现import不到，鼠标移上去，点击sycn.....
    - 或者执行go mod tidy同步包