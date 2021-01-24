---
title: wait/notify简单实现生产消费模式
layout: post
categories: [Concurrent, Java]
image: /assets/img/notes/PCpattern.png
description: "Welcome"
---

## 异步模式——生产者/消费者模式

### 要点 

- 与前面的保护性暂停中的 GuardObject 不同，不需要产生结果和消费结果的线程一一对应 
- 消费队列可以用来平衡生产和消费的线程资源 
- 生产者仅负责产生结果数据，不关心数据该如何处理，而消费者专心处理结果数据 
- 消息队列是有容量限制的，满时不会再加入数据，空时不会再消耗数据 JDK 中各种阻塞队列，采用的就是这种模式

![image-20210122221401527](/assets/img/notes/image-20210122221401527.png)

### 实现

```java
class Message {
 	private int id;
 	private Object message;
 	public Message(int id, Object message) {
 		this.id = id;
 		this.message = message;
 	}	
 	public int getId() {
		 return id;
	}
 	public Object getMessage() {
 		return message;
     }
}
class MessageQueue {
 	private LinkedList<Message> queue;
 	private int capacity;
 	public MessageQueue(int capacity) {
 		this.capacity = capacity;
 		queue = new LinkedList<>();
 	}
 	public Message take() {
 		synchronized (queue) {
 			while (queue.isEmpty()) {
 				log.debug("没货了, wait");
 				try {
					queue.wait();
 				} catch (InterruptedException e) {
 					e.printStackTrace();
 				}
 			}
 			Message message = queue.removeFirst();
 			queue.notifyAll();
 			return message;
 		}
 	}
 	public void put(Message message) {
 		synchronized (queue) {
 			while (queue.size() == capacity) {
 				log.debug("库存已达上限, wait");
 				try {
 					queue.wait();
 				} catch (InterruptedException e) {
 					e.printStackTrace();
 				}
 			}
 			queue.addLast(message);
			queue.notifyAll();
 		}
 	}
}

```

### 应用

```java
MessageQueue messageQueue = new MessageQueue(2);
// 4 个生产者线程, 下载任务
for (int i = 0; i < 4; i++) {
 	int id = i;
 	new Thread(() -> {
 		try {
     		log.debug("download...");
	 		List<String> response = Downloader.download();
 			log.debug("try put message({})", id);
			messageQueue.put(new Message(id, response));
 		} catch (IOException e) {
		 	e.printStackTrace();
 		}
 	}, "生产者" + i).start();
}
// 1 个消费者线程, 处理结果
new Thread(() -> {
 	while (true) {
 		Message message = messageQueue.take();
 		List<String> response = (List<String>) message.getMessage();
 		log.debug("take message({}): [{}] lines", message.getId(), response.size());
 	}
}, "消费者").start();
```

### output

```java
10:48:38.070 [生产者3] c.TestProducerConsumer - download...
10:48:38.070 [生产者0] c.TestProducerConsumer - download...
10:48:38.070 [消费者] c.MessageQueue - 没货了, wait
10:48:38.070 [生产者1] c.TestProducerConsumer - download...
10:48:38.070 [生产者2] c.TestProducerConsumer - download...
10:48:41.236 [生产者1] c.TestProducerConsumer - try put message(1)
10:48:41.237 [生产者2] c.TestProducerConsumer - try put message(2)
10:48:41.236 [生产者0] c.TestProducerConsumer - try put message(0)
10:48:41.237 [生产者3] c.TestProducerConsumer - try put message(3)
10:48:41.239 [生产者2] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [生产者1] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [消费者] c.TestProducerConsumer - take message(0): [3] lines
10:48:41.240 [生产者2] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [消费者] c.TestProducerConsumer - take message(3): [3] lines
10:48:41.240 [消费者] c.TestProducerConsumer - take message(1): [3] lines
10:48:41.240 [消费者] c.TestProducerConsumer - take message(2): [3] lines
10:48:41.240 [消费者] c.MessageQueue - 没货了, wait
```

