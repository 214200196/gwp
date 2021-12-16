package v1

import (
	"errors"
	"sync"
	"sync/atomic"
)

// task任务
type TaskHandler struct {
	f   func(interface{}) error
	arg interface{}
}

// 配置
type WorkerPool struct {
	maxWorkers  int32
	taskQueue   chan TaskHandler
	workerQueue chan TaskHandler
	wg          sync.WaitGroup
	closeOnce   *sync.Once
	err         chan error
	stat        int32 // 0 正常 1 关闭
}

// 初始化
func NewWorkerPool(max int32) *WorkerPool {
	if max < 1 {
		max = 1
	}

	return &WorkerPool{
		maxWorkers:  max,
		taskQueue:   make(chan TaskHandler, max*2),
		workerQueue: make(chan TaskHandler, max),
		err:         make(chan error, 1),
		closeOnce:   new(sync.Once),
		stat:        0,
	}
}

// 初始化task
func (w *WorkerPool) NewTask(f func(interface{}) error, arg interface{}) TaskHandler {
	return TaskHandler{f, arg}
}

// 新增任务
func (w *WorkerPool) Submit(task TaskHandler) error {
	// 检查是关闭
	if atomic.LoadInt32(&w.stat) == 1 {
		return errors.New("work pool closed")
	}

	// 出现报错停止投递任务
	if len(w.err) > 0 {
		w.Quit()
		err, _ := <-w.err
		return err
	}

	if task.f != nil {
		w.taskQueue <- task
	}

	return nil
}

// 退出
func (w *WorkerPool) Quit() {
	atomic.StoreInt32(&w.stat, 1)
	w.closeOnce.Do(func() {
		close(w.taskQueue)
	})
}

// 执行任务
func (w *WorkerPool) exec() {
	defer w.wg.Done()
	for task := range w.workerQueue {
		if err := task.f(task.arg); err != nil {
			if len(w.err) == 0 {
				w.err <- err
				break
			}
		}
	}
}

// 调度
func (w *WorkerPool) Dispatch() {
	defer close(w.err)

	// task任务投递work
	go func() {
		defer close(w.workerQueue)
		for task := range w.taskQueue { // range捕获channel关闭则结束阻塞
			w.workerQueue <- task
		}
	}()

	// 创建并发协程
	w.wg.Add(int(w.maxWorkers))
	for i := int32(0); i < w.maxWorkers; i++ {
		go w.exec()
	}
	w.wg.Wait()
}
