
线程同步机制包装类
===============
多线程同步，确保任一时刻只能有一个线程能进入关键代码段.
> * 信号量
> * 互斥锁
> * 条件变量

linux下c++线程同步机制

参考：https://zhuanlan.zhihu.com/p/537995532

模拟多线程多窗口售票情况

```c
#include <pthread.h>
#include <unistd.h>
#include <iostream>
int ticket_num = 20;
void* sell_ticket(void* arg)
{
	for(int i=0; i<20; i++)
	{
		if(ticket_num > 0)
		{
			sleep(1);
			std::cout << "sell the " << 20-ticket_num+1 << "th" <<std::endl;
			ticket_num--;
		}
	}
	return 0;
}
int main()
{
	int ret;
	pthread_t pid[4];
	
	for(int i=0; i<4; i++)
	{
		ret = pthread_create(&pid[i], NULL, &sell_ticket, NULL);
		if(ret < 0)
		{
			std::cout << "thread create err, ret = " << ret << std::endl;
			return ret;
		}
	}
	
	sleep(30);
	
	void *ans;
	for(int i=0; i<4; i++)
	{
		ret = pthread_join(pid[i], &ans);
		if(ret < 0)
		{
			std::cout << "thread join err, ret = " << ret << std::endl;
			return ret;
		}
		std::cout << "ans:" << ans <<std::endl;
	}
	
	return 0;
}
```

### 1.互斥锁

本质就是一个特殊的全局变量，拥有lock和unlock两种状态，unlock的互斥锁可以由某个线程获得，一旦获得，这个互斥锁会锁上变成lock状态，此后只有该线程由权力打开该锁，其他线程想要获得互斥锁，必须得到互斥锁再次被打开之后

思路：通过为售票的核心代码段加互斥锁使得其变成了一个原子性操作！ 不会被其他线程影响

```c
#include <pthread.h>
#include <unistd.h>
#include <iostream>
int ticket_num = 20;
pthread_mutex_t mutex_x=PTHREAD_MUTEX_INITIALIZER;//static init mutex
void* sell_ticket(void* arg)
{
	for(int i=0; i<20; i++)
	{
		pthread_mutex_lock(&mutex_x);//atomic opreation through mutex lock
		if(ticket_num > 0)
		{
			sleep(1);
			std::cout << "sell the " << 20-ticket_num+1 << "th" <<std::endl;
			ticket_num--;
		}
		pthread_mutex_unlock(&mutex_x);
	}
	return 0;
}
int main()
{
	int ret;
	pthread_t pid[4];
	
	for(int i=0; i<4; i++)
	{
		ret = pthread_create(&pid[i], NULL, &sell_ticket, NULL);
		if(ret < 0)
		{
			std::cout << "thread create err, ret = " << ret << std::endl;
			return ret;
		}
	}
	
	sleep(20);
	
	void *ans;
	for(int i=0; i<4; i++)
	{
		ret = pthread_join(pid[i], &ans);
		if(ret < 0)
		{
			std::cout << "thread join err, ret = " << ret << std::endl;
			return ret;
		}
		std::cout << "ans:" << ans <<std::endl;
	}
	
	return 0;
}
```

互斥锁的初始化、属性、分类、加锁解锁

缺点：

加锁函数在锁已经被占据时返回EBUSY而不是挂起等待

互斥量不是万能的，比如某个线程正在等待共享数据内某个条件出现，可可能需要重复对数据对象加锁和解锁（轮询），但是这样轮询非常耗费时间和资源，而且效率非常低，所以互斥锁不太适合这种情况

### 2.条件变量

当线程在等待满足某些条件时使线程进入睡眠状态，一旦条件满足，就换线因等待满足特定条件而睡眠的线程

```c
#include <pthread.h>
#include <unistd.h>
#include <iostream>
pthread_cond_t qready=PTHREAD_COND_INITIALIZER;  //cond
pthread_mutex_t qlock=PTHREAD_MUTEX_INITIALIZER; //mutex
int x=10,y=20;
void* f1(void* arg)
{
	std::cout << "f1 start\n";
	pthread_mutex_lock(&qlock);
	while(x<y)
	{
		pthread_cond_wait(&qready,&qlock);
	}
	pthread_mutex_unlock(&qlock);
	sleep(3);
	std::cout << "f1 end\n";
	return 0;
}
void* f2(void* arg)
{
	std::cout << "f2 start\n";
	pthread_mutex_lock(&qlock);
	x = 20;
	y = 10;
	std::cout << "has a change,x=" << x << " y=" << y << std::endl;
	pthread_mutex_unlock(&qlock);
	if(x > y)
	{
		pthread_cond_signal(&qready);
	}
	std::cout << "f2 end\n";
	return 0;
}
int main()
{
	int ret;
	pthread_t pid[2];
	
	ret = pthread_create(&pid[0], NULL, f1, NULL);
	if(ret < 0)
	{
		std::cout << "thread f1 create err, ret = " << ret << std::endl;
		return ret;
	}
	
	sleep(2);
	
	ret = pthread_create(&pid[1], NULL, f2, NULL);
	if(ret < 0)
	{
		std::cout << "thread f2 create err, ret = " << ret << std::endl;
		return ret;
	}
	
	sleep(5);
	
	return 0;
}
```

条件变量的创建、销毁、条件等待、计时等待、通知

需要互斥锁配合，在调用pthread_cond_wait前必须由本线程加锁

激发一个等待线程：pthread_cond_signal(&cond)

激发所有等待线程：pthread_cond_broadcast(&cond)

重要的是，pthread_cond_signal不会存在惊群效应，也就是是它最多给一个等待线程发信号，不会给所有线程发信号唤醒提他们，然后要求他们自己去争抢资源！

pthread_cond_signal会根据等待线程的优先级和等待时间来确定激发哪一个等待线程

### 3.读写锁

可以多个线程同时读，但是不能多个线程同时写

### 4.信号量

信号量（sem）和互斥锁的区别：互斥锁只允许一个线程进入临界区，而信号量允许多个线程进入临界区



# 问题

一、条件变量为什么不直接在cond类中和互斥量配合使用呢？

个人理解：这里只是单独封装，具体使用时再配合使用



