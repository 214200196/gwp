# go-worker-pool
Support variable parameters task send worker

## 调用方式
```
workPool := v1.NewWorkerPool(5)
defer workPool.Dispatch()

go func() {
    for i :=0; i < 50; i++ {
        f := func(s interface{}) error {
            fmt.Println("task func", s)
            time.Sleep(time.Second)
            // return errors.New("task func error")
            return nil
        }
        task := workPool.NewTask(f, i)
        if err := workPool.Submit(task); err != nil {
            fmt.Println(err)
            break
        }
    }
    // 退出
    workPool.Quit()
}()
```
