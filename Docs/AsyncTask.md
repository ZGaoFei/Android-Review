#### AsyncTask

```
SERIAL_EXECUTOR：SerialExecutor，保证线程是按照顺序执行的，串行执行
SerialExecutor：内部将任务加入到任务队列中，然后将任务交给THREAD_POOL_EXECUTOR来执行
THREAD_POOL_EXECUTOR：一个线程池，核心线程数为1，最大线程数为20，保活时间3s，同步阻塞队列的线程池

```

