下面系统性地介绍 Zephyr
  中核心系统组件的用法。每个组件我会给出关键 API
  和代码示例。

  1. 线程 (Thread)

  Zephyr 的线程分三种：协作线程（cooperative，-1
  优先级，不抢占）、抢占式线程（preemptible，0 到
  N）、工作队列线程（由内核管理）。

  创建线程

  // 定义线程栈和线程数据
  #define STACK_SIZE 1024
  #define MY_PRIORITY 5

  K_THREAD_STACK_DEFINE(my_stack, STACK_SIZE);
  struct k_thread my_thread_data;
  k_tid_t my_tid;

  // 线程入口函数
  void thread_entry(void *p1, void *p2, void *p3) {
      while (1) {
          // 业务逻辑
          k_sleep(K_MSEC(100));
      }
  }

  // 创建线程
  my_tid = k_thread_create(&my_thread_data, my_stack,
  STACK_SIZE,
                           thread_entry, NULL, NULL,
  NULL,
                           MY_PRIORITY, 0, K_NO_WAIT);

  两种启动方式

  // 方式1：立即启动（K_NO_WAIT）
  k_thread_create(&my_thread_data, my_stack,
  STACK_SIZE,
                  thread_entry, NULL, NULL, NULL,
                  MY_PRIORITY, 0, K_NO_WAIT);

  // 方式2：延迟启动（传入 ms）
  k_thread_create(&my_thread_data, my_stack,
  STACK_SIZE,
                  thread_entry, NULL, NULL, NULL,
                  MY_PRIORITY, 0, K_MSEC(500));

  // 动态线程（从堆分配栈，不推荐嵌入式场景）

  线程控制

  k_thread_start(my_tid);           // 手动启动
  k_thread_suspend(my_tid);         // 挂起
  k_thread_resume(my_tid);          // 恢复
  k_thread_abort(my_tid);           // 终止
  k_thread_join(my_tid, K_FOREVER); // 等待线程结束

  注意： Zephyr 没有 k_thread_stop()，终止线程用
  abort，但更推荐让线程函数 return。

  ---
  2. 互斥量 (Mutex)

  互斥量用于保护共享资源，只能被获取它的线程释放，支持
  优先级继承：

  // 定义互斥量
  K_MUTEX_DEFINE(my_mutex);

  void thread_a(void *p1, void *p2, void *p3) {
      // 加锁（K_FOREVER = 永久等待）
      if (k_mutex_lock(&my_mutex, K_FOREVER) == 0) {
          // 临界区
          shared_counter++;
          // 解锁
          k_mutex_unlock(&my_mutex);
      }
  }

  void thread_b(void *p1, void *p2, void *p3) {
      // 非阻塞尝试，拿不到就干别的
      if (k_mutex_lock(&my_mutex, K_NO_WAIT) == 0) {
          // 拿到了锁
          shared_data = 42;
          k_mutex_unlock(&my_mutex);
      } else {
          // 没拿到，做其他事
          printk("Mutex busy, skipping\n");
      }
  }

  注意：
  - 中断上下文中不能使用 mutex（会 context switch）
  - mutex 的优先级继承可以防止优先级反转

  ---
  3. 信号量 (Semaphore)

  用于线程间的发送信号和资源计数：

  // 定义信号量（初始值0，最大值1 = 二进制信号量）
  K_SEM_DEFINE(my_sem, 0, 1);

  // 生产者线程
  void producer(void *p1, void *p2, void *p3) {
      while (1) {
          // 生产数据...
          produce_data();
          // 给消费者发信号
          k_sem_give(&my_sem);
          k_sleep(K_MSEC(100));
      }
  }

  // 消费者线程
  void consumer(void *p1, void *p2, void *p3) {
      while (1) {
          // 等待信号量
          if (k_sem_take(&my_sem, K_SECONDS(1)) == 0)
  {
              // 拿到信号，消费数据
              consume_data();
          } else {
              // 超时了，信号没来
              printk("Timeout waiting for data\n");
          }
      }
  }

  计数信号量

  K_SEM_DEFINE(counting_sem, 5, 10); //
  初始5个资源，最大10个

  // 获取资源
  k_sem_take(&counting_sem, K_FOREVER);

  // 归还资源
  k_sem_give(&counting_sem);

  // 重置信号量
  k_sem_reset(&counting_sem);

  // 获取当前计数值
  unsigned int count = k_sem_count_get(&counting_sem);

  关键区别：
  - 二进制信号量（max=1）：线程间同步（生产者-消费者模
  式、ISR 通知线程）
  - 计数信号量（max>1）：管理 N 个相同资源（如 N 个
  buffer）

  ---
  4. 消息队列 (Message Queue)

  Zephyr 的消息队列传的是数据拷贝（不是指针），按固定
  大小消息传递：

  // 定义消息类型
  struct sensor_data {
      int32_t temperature;
      int32_t humidity;
      uint32_t timestamp;
  };

  // 定义消息队列（最大10条消息）
  K_MSGQ_DEFINE(sensor_msgq, sizeof(struct
  sensor_data), 10, 4);

  // 生产者（传感器线程）
  void sensor_thread(void *p1, void *p2, void *p3) {
      struct sensor_data data;
      while (1) {
          // 读取传感器
          data.temperature = read_temp();
          data.humidity = read_humidity();
          data.timestamp = k_uptime_get();

          // 发送消息（K_NO_WAIT = 队列满时不等待）
          int ret = k_msgq_put(&sensor_msgq, &data,
  K_NO_WAIT);
          if (ret != 0) {
              printk("Queue full, dropping sample\n");
          }
          k_sleep(K_MSEC(200));
      }
  }

  // 消费者（处理线程）
  void processor_thread(void *p1, void *p2, void *p3)
  {
      struct sensor_data data;
      while (1) {
          // 获取消息（等待直到有数据）
          if (k_msgq_get(&sensor_msgq, &data,
  K_FOREVER) == 0) {
              process_data(&data);
          }
      }
  }

  ISR 中使用消息队列

  // 在中断中发送（非等待版本）
  void my_isr_handler(void *arg) {
      struct sensor_data data;
      data.temperature = read_temp_from_isr();
      data.timestamp = k_uptime_get();

      // ISR 只能用 K_NO_WAIT
      k_msgq_put(&sensor_msgq, &data, K_NO_WAIT);
  }

  清空和查看队列

  // 清空整个队列
  k_msgq_purge(&sensor_msgq);

  // 查看已用/空闲数量
  uint32_t used = k_msgq_num_used_get(&sensor_msgq);
  uint32_t free = k_msgq_num_free_get(&sensor_msgq);

  // 非破坏性 peek
  struct sensor_data peek_data;
  k_msgq_peek(&sensor_msgq, &peek_data); //
  读取但不移除

  ---
  5. FIFO 和 LIFO（数据队列，传指针）

  适合传大块数据（只传指针，零拷贝），不像 msgq
  那样拷贝数据：

  // 数据类型（需要内嵌 FIFO/LIFO 节点）
  struct data_item {
      void *fifo_reserved;   // FIFO 节点（内核使用）
      int value;
  };

  K_FIFO_DEFINE(my_fifo);

  void producer(void *p1, void *p2, void *p3) {
      // 静态分配 item
      struct data_item items[5] = {
          {.value = 10},
          {.value = 20},
          {.value = 30},
      };
      for (int i = 0; i < 3; i++) {
          k_fifo_put(&my_fifo, &items[i]);
      }
  }

  void consumer(void *p1, void *p2, void *p3) {
      while (1) {
          struct data_item *item =
  k_fifo_get(&my_fifo, K_FOREVER);
          printk("Got value: %d\n", item->value);
          // item 不再需要时，归还到内存池
      }
  }

  FIFO 的关键特性

  // FIFO（先进先出）
  k_fifo_put(&fifo, &item);    // 尾部插入
  item = k_fifo_get(&fifo, K_FOREVER); // 头部取出

  // LIFO（后进先出，类似栈）
  k_lifo_put(&lifo, &item);    // 压入
  item = k_lifo_get(&lifo, K_FOREVER); // 弹出

  // ISR 中只能用 put（不阻塞）
  k_fifo_put(&fifo, &item);    // OK in ISR
  // k_fifo_get 不能在 ISR 中使用（会阻塞）

  MsgQ vs FIFO 选择：
  - MsgQ：消息小（几字节到几十字节），拷贝开销可接受，
  简单安全
  - FIFO：消息大（几百字节+），避免拷贝，但需要自己管
  理内存

  ---
  6. 栈 (Stack — 内核栈，非线程栈)

  用于后进先出的数据传递（注意这是数据栈，不是线程栈）
  ：

  K_STACK_DEFINE(my_stack, 10); // 最多存10个 uint32_t

  void producer(void *p1, void *p2, void *p3) {
      for (int i = 0; i < 5; i++) {
          uint32_t data = i * 10;
          k_stack_push(&my_stack, data);
      }
  }

  void consumer(void *p1, void *p2, void *p3) {
      uint32_t data;
      while (1) {
          if (k_stack_pop(&my_stack, &data, K_FOREVER)
   == 0) {
              printk("Popped: %u\n", data);
          }
      }
  }

  ---
  7. 事件 (Events — 更高效的信号)

  Events 是 Zephyr 的线程间通知机制，比信号量更灵活，
  一个线程可以同时等多个事件：

  // 定义事件对象
  struct k_event my_event;

  // 初始化
  k_event_init(&my_event);

  // 也可以静态定义
  K_EVENT_DEFINE(my_event);

  // 生产者线程 — 发送事件
  void producer(void *p1, void *p2, void *p3) {
      while (1) {
          if (sensor_ready()) {
              k_event_post(&my_event, BIT(0));  //
  发送事件位0
          }
          if (button_pressed()) {
              k_event_post(&my_event, BIT(1));  //
  发送事件位1
          }
          k_sleep(K_MSEC(50));
      }
  }

  // 消费者线程 — 等待多个事件中的任意一个
  void consumer(void *p1, void *p2, void *p3) {
      while (1) {
          // 等待位0或位1任意一个（逻辑或）
          uint32_t events = k_event_wait(&my_event,
                                         BIT(0) |
  BIT(1),  // 感兴趣的位
                                         false,
        // false=逻辑或, true=逻辑与
                                         K_FOREVER);
        // 超时

          if (events & BIT(0)) {
              read_sensor();
          }
          if (events & BIT(1)) {
              handle_button();
          }
          // 等价于上面的掩码
          k_event_set(&my_event, 0); // 清空事件
      }
  }

  Events 高级用法

  // 等待所有事件都发生（逻辑与）
  uint32_t events = k_event_wait(&my_event,
                                 BIT(0) | BIT(1) |
  BIT(2),
                                 true,          //
  true = 必须全满足
                                 K_SECONDS(5));

  // 从 ISR 发送事件
  void my_isr(void *arg) {
      k_event_post(&my_event, BIT(3)); // ISR 安全
  }

  // 设置特定事件位而不消费它们
  k_event_set(&my_event, BIT(4));

  // 等待并自动清除匹配的事件
  uint32_t events = k_event_wait(&my_event,
                                 BIT(0) | BIT(1),
                                 false,
                                 K_FOREVER);
  // events 包含实际触发的事件位组合

  ---
  8. 工作队列 (Work Queue)

  用于延迟执行或委托到系统工作线程，避免创建过多线程：

  系统工作队列

  // 定义工作项
  struct k_work my_work;

  // 工作处理函数
  void work_handler(struct k_work *work) {
      printk("Work executed! Uptime: %lld\n",
  k_uptime_get());
      // 这里可以做一次性任务
  }

  // 提交到系统工作队列
  k_work_init(&my_work, work_handler);
  k_work_submit(&my_work); // 系统工作队列线程会执行它

  延迟工作（一次性定时器替代）

  struct k_work_delayable delay_work;

  void delayed_handler(struct k_work *work) {
      struct k_work_delayable *dwork =
  k_work_delayable_from_work(work);
      printk("Delayed work fired!\n");

      // 可以重新调度（实现周期性执行）
      k_work_schedule(dwork, K_MSEC(500)); // 500ms
  后再来
  }

  // 初始化并调度
  k_work_init_delayable(&delay_work, delayed_handler);
  k_work_schedule(&delay_work, K_MSEC(200)); // 200ms
  后首次执行

  // 取消未执行的延迟工作
  k_work_cancel_delayable(&delay_work);

  自定义工作队列（独立线程）

  #define MY_WORK_STACK_SIZE 1024
  #define MY_WORK_PRIORITY   5

  K_THREAD_STACK_DEFINE(workq_stack,
  MY_WORK_STACK_SIZE);
  struct k_work_q my_workq;

  // 初始化自定义工作队列
  k_work_queue_start(&my_workq, workq_stack,
  MY_WORK_STACK_SIZE,
                     MY_WORK_PRIORITY, NULL);

  // 提交到自定义队列
  k_work_submit_to_queue(&my_workq, &my_work);

  系统工作队列 vs 自定义队列：
  - 系统工作队列：简单场景，系统自带，不要提交会长时间
  阻塞或睡眠的工作
  - 自定义队列：高优先级或需要长时间运行的任务，不影响
  系统工作队列

  ---
  9. 定时器 (Timer)

  // 一次性定时器
  struct k_timer my_timer;

  void timer_handler(struct k_timer *timer) {
      printk("One-shot timer fired!\n");
  }

  k_timer_init(&my_timer, timer_handler, NULL);
  k_timer_start(&my_timer, K_SECONDS(1), K_NO_WAIT);
  // 1秒后触发一次

  // 周期性定时器
  k_timer_start(&my_timer, K_MSEC(500), K_MSEC(200));
  // 500ms后首次，之后每200ms

  // 停止
  k_timer_stop(&my_timer);

  // 检查状态
  uint32_t remaining =
  k_timer_remaining_get(&my_timer);

  ---
  10. 轮询 (Polling — 同时等多个事件源)

  // 轮询多个内核对象
  struct k_poll_event events[2];

  struct k_poll_event_init(&events[0],
                           K_POLL_TYPE_SEM_AVAILABLE,
                           K_POLL_MODE_NOTIFY_ONLY,
                           &my_sem);

  struct k_poll_event_init(&events[1],

  K_POLL_TYPE_MSGQ_DATA_AVAILABLE,
                           K_POLL_MODE_NOTIFY_ONLY,
                           &sensor_msgq);

  // 等待任意一个事件
  int ret = k_poll(events, 2, K_SECONDS(5));
  if (ret == 0) {
      for (int i = 0; i < 2; i++) {
          if (events[i].state ==
  K_POLL_STATE_SEM_AVAILABLE) {
              k_sem_take(&my_sem, K_NO_WAIT);
              // 处理信号量...
          }
          if (events[i].state ==
  K_POLL_STATE_MSGQ_DATA_AVAILABLE) {
              // 处理消息队列...
          }
      }
  }

  ---
  总结对照表

  ┌──────┬──────────┬──────┬─────┬──────────────┐
  │      │          │ ISR  │ 传  │              │
  │ 组件 │   用途   │ 安全 │ 数  │   关键特性   │
  │      │          │      │ 据  │              │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Thre │ 独立执行 │ —    │ —   │ 协作/抢占优  │
  │ ad   │ 流       │      │     │ 先级         │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Mute │ 互斥访问 │ 不可 │ 无  │ 优先级继承， │
  │ x    │ 共享资源 │ 用   │     │ 防反转       │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Sema │ 信号通知 │ give │     │ 计数型 >     │
  │ phor │  +       │ /pos │ 无  │ 资源管理     │
  │ e    │ 资源计数 │ t    │     │              │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ MsgQ │ 拷贝消息 │ put  │ 拷  │ 固定大小，简 │
  │      │ 传递     │ 可用 │ 贝  │ 单安全       │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ FIFO │ 指针传递 │ put  │ 指  │ 零拷贝，需管 │
  │ /LIF │ 消息     │ 可用 │ 针  │ 理内存       │
  │ O    │          │      │     │              │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Stac │ LIFO     │ push │ uin │              │
  │ k    │ 数据传递 │      │ t32 │ 简单栈操作   │
  │      │          │ 可用 │     │              │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Even │ 多条件位 │ post │ 无  │ 等任意条件， │
  │ ts   │ 通知     │ /set │     │ 比信号量强   │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Work │ 延迟/委  │ subm │     │ 避免创建过多 │
  │  Que │ 托执行   │ it   │ 无  │ 线程         │
  │ ue   │          │      │     │              │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │ Time │          │      │     │ 一次/周期，h │
  │ r    │ 定时触发 │ —    │ 无  │ andler ISR   │
  │      │          │      │     │ 上下文       │
  ├──────┼──────────┼──────┼─────┼──────────────┤
  │      │ 同时等多 │ 不可 │     │ epoll/select │
  │ Poll │ 对象     │ 用   │ 无  │  的 Zephyr   │
  │      │          │      │     │ 版           │
  └──────┴──────────┴──────┴─────┴──────────────┘