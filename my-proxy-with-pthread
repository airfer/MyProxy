#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <netdb.h>
#include <pthread.h>

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include "list.h"
#include "logmsg.h"
#include "dns.h"

#define MAX_BUF_SIZE    	10000
#define MAX_CONCURRENT		1000

typedef struct  {
	char dns[DNS_MAX_SIZE];
	struct sockaddr_in addr;
	list_t list;
} dns_addr_t;

typedef struct thread_handler_s {
	int sockfd;
	pthread_t pid;
	pthread_mutex_t lock;
	void *buf;
	struct thread_handler_s *next;
} thread_handler_t;

static thread_handler_t *thread_pool;
static thread_handler_t *next_thread_handler = NULL;
static int max_concurrent, cur_thread_count = 0;
static pthread_mutex_t pool_lock;
//static pthread_mutex_t dns_lock;
static LIST_HEAD(dns_head);

static void * get_buffer(void)
{
	return malloc(MAX_BUF_SIZE);
}

static void free_buffer(void *buf)
{
	free(buf);
}

static int init_thread_pool(int maxcc)//初始化线程池，并且加锁
{
	thread_handler_t *tp;
	int i;

	max_concurrent = maxcc;
	thread_pool = (thread_handler_t *)malloc(sizeof(thread_handler_t) * maxcc);
	if (!thread_pool) {
		LOG_MSG(0, errno, "init_pthread_pool");
		return -1;
	}
	for (i = 0; i < maxcc; i++) {
		tp = thread_pool + i;
		pthread_mutex_init(&tp->lock, NULL);
		pthread_mutex_lock(&tp->lock);
		tp->sockfd = -1;
		tp->pid = -1;
		tp->buf = NULL;
	}
	pthread_mutex_init(&pool_lock, NULL);

	return 0;
}

static void * transmit_handler(void *data);
static int alloc_thread_pool(int num)//根据线程池创建线程
{
	pthread_t pid;
	int i;
        	
	for (i = 0; cur_thread_count+1 < max_concurrent && i < num; i++) {
		thread_handler_t *tp = &thread_pool[cur_thread_count];

		if (pthread_create(&pid, NULL, transmit_handler, tp) < 0) {
			LOG_MSG(0, errno, "failed to create tunnel_handler thread");
			break;
		}

		tp->next = next_thread_handler;//将线程结构用指针相互连接
		next_thread_handler = tp;//这里插入的方式采用的是头插法，最后next_thread_handler指向第一个元素
		tp->pid = pid;

		cur_thread_count++;
	}

	return i;
}

static int start_transmit(int sockfd)
{
	thread_handler_t *tp;

	pthread_mutex_lock(&pool_lock);//对线程池进行加锁

	//FIXME socket reset
	while (!next_thread_handler) {//如果线程池为空，则分配线程池，如果分配线程池失败，则睡眠后重试
		if (!alloc_thread_pool(100) && !next_thread_handler)
			sleep(5);//在第一次的时候肯定是要分配的
	}

	tp = next_thread_handler;
	next_thread_handler = tp->next;

	tp->buf = get_buffer();
	if (!tp->buf) {//如果分配内存失败，则next_thread_handler不发生移动，并对线程池解锁返回
		next_thread_handler = tp;
		pthread_mutex_unlock(&pool_lock);
		return -1;
	}

	tp->sockfd = sockfd;//如果buffer分配成功则分配套接字并解锁，这里的sockfd为accept之后的套接字
	pthread_mutex_unlock(&tp->lock);
	pthread_mutex_unlock(&pool_lock);

	return 0;
}

static void end_transmit(thread_handler_t *tp)//释放套接字，也释放分配的内存
{
	pthread_mutex_lock(&pool_lock);

	close(tp->sockfd);
	tp->sockfd = -1;

	free_buffer(tp->buf);
	tp->buf = NULL;

	tp->next = next_thread_handler;
	next_thread_handler = tp;

	pthread_mutex_unlock(&pool_lock);
}

static int make_connect_socket(struct sockaddr_in *addrp)//用作客户端，不用进行绑定
{
	int sfd;

	if ((sfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
		LOG_MSG(0, errno, "socket");
		return -1;
	}

	if (connect(sfd, (struct sockaddr *)addrp, sizeof(*addrp)) < 0) {
		LOG_MSG(0, errno, "connect");
		close(sfd);
		return -1;
	}

	return sfd;
}

static int make_listen_socket(int port)//用作服务器端，进行监听
{
	int sfd;
	struct sockaddr_in addr;
	int addrlen = sizeof(addr);

	if ((sfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
		LOG_MSG(0, errno, "socket");
		return -1;
	}

	addr.sin_addr.s_addr = htonl(INADDR_ANY);
	addr.sin_family = AF_INET;
	addr.sin_port = htons(port);
	if (bind(sfd, (struct sockaddr *)&addr, addrlen) < 0) {
		LOG_MSG(0, errno, "bind");
		return -1;
	}

	if (listen(sfd, 1) < 0) {
		LOG_MSG(0, errno, "listen");
		return -1;
	};

	return sfd;
}

static int write_data(int fd, const char *buffer, unsigned short size)
{                                //发送一定字节的数据，并且可能分几次发送完
    int num = 0;
    const char *buf = NULL;
    unsigned short total = 0;

    buf = (char *)buffer;
	do {
		if ((num = send(fd, buf + total, size - total, 0)) <= 0) {
			LOG_MSG(3, errno, "writeData: EOF or error");
			return num;
		}
		total += (unsigned short)num;
	} while (total < size);

    return total;
}

static int read_data(int fd, char *buffer, unsigned short size)
{                              //从buffer中读取数据
    int num;

	if ((num = recv(fd, buffer, size, 0)) == 0) {
		LOG_MSG(3, 0, "readData: EOF");
		return -1;
	}
	else if (num < 0) {
		LOG_MSG(0, errno, "readData");
		return -1;
	}

    return num;
}

static int transmit_loop(int tunnel_fd, int web_fd, char *buffer, int max_size)
{
    fd_set testset;
    int ready = 0;
    int maxfd = (web_fd > tunnel_fd ? web_fd+1 : tunnel_fd+1);
    int size = 0;

	while (1) {  //主题不发生变化，都是调用select，来进行套接子的选择，从一个套接字转发到另一个套接字
		FD_ZERO(&testset);
		FD_SET(web_fd, &testset);
		FD_SET(tunnel_fd, &testset);

		if ((ready = select(maxfd, &testset, 0, 0, NULL)) < 0) {
			LOG_MSG(0, errno, "error in select");
			return -1;
		}

		if (FD_ISSET(web_fd, &testset)) {
			if ((size = read_data(web_fd, buffer, max_size)) > 0) {

				if (write_data(tunnel_fd, buffer, size) != size) {
					LOG_MSG(0, 0, "write web err");
					return -1;
				}
			}
			else
				return -1;
		}

		if (FD_ISSET(tunnel_fd, &testset)) {
			if ((size = read_data(tunnel_fd, buffer, max_size)) > 0) {

				if (write_data(web_fd, buffer, size) != size) {
					LOG_MSG(0, 0, "write tunnel err");
					return -1;
				}
			}
			else
				return -1;
		}
	}

    LOG_MSG(1, 0, "connection closed");

    return 0;
}

static dns_addr_t * find_dns_addr(const char *dns)
{
	list_t *pos;
	dns_addr_t *dp = NULL;

	//lock()
	list_for_each(pos, &dns_head) {
		dp = list_entry(pos, dns_addr_t, list);

		if (!strcmp(dns, dp->dns))
			return dp;
	}
	//ulock()

	return NULL;
}

static int get_dns_addr(int fd, struct sockaddr_in *addr)//与DNS相关问题
{
	char buf[2000], *p;
	fd_set testset;
	int size, i;
	char label[100], dns[100];
	dns_addr_t *dp;

	FD_ZERO(&testset);
	FD_SET(fd, &testset);
	if (select(fd + 1, &testset, 0, 0, NULL) <= 0) {//这里为何是fd+1？
		LOG_MSG(0, errno, "timed out or error");//因为第一个参数表示的是范围
		return -1;
	}

	if ((size = recv(fd, buf, 2000, MSG_PEEK)) <= 0) {
		LOG_MSG(1, errno, "MSG_PEEK");
		return -1;
	}

	for (i = 0; i < size; i++) {
		if (buf[i] == '\n' || buf[i] == '\r')
			buf[i] = 0;
	}

	p = buf;
	while (p - buf < size) {
		sscanf(p, "%s %[^:]", label, dns);

		if (!strcmp(label, "Host:")) {//由域名来查找ipv4的地址
			if ((dp = find_dns_addr(dns)) != NULL) {
				*addr = dp->addr;
				return 0;
			}
			return -1;
		}

		p += strlen(p) + 1;
	}

	return -1;
}

static void * transmit_handler(void *data)
{
	thread_handler_t *tp = (thread_handler_t *)data;//传送处理句柄
	struct sockaddr_in addr;
	addr.sin_family=AF_INET;
	addr.sin_addr.s_addr=inet_addr("10.66.10.82");
	addr.sin_port=htons(3306);
	int wfd;

	while (1) {
		pthread_mutex_lock(&tp->lock);

		/*if (get_dns_addr(tp->sockfd, &addr) < 0) {
			LOG_MSG(1, 0, "no match dns\n");
			goto endh;
		}*/

		if ((wfd = make_connect_socket(&addr)) < 0) {//客户端建立连接，可以看出，addr的地址是从dns中获得的
			LOG_MSG(0, 0, "make_connect_socket");
			goto endh;
		}

		if (transmit_loop(wfd, tp->sockfd, tp->buf, MAX_BUF_SIZE) < 0)
			LOG_MSG(1, 0, "transmit_loop");//从这里可以看到，wfd为connect后的fd，也就是说客户端
                                                     //tp->sockfd为accept之后套接字
		close(wfd);
	endh:
		end_transmit(tp);
	}
}

static int del_dns_list(const char *dns)
{
	dns_addr_t *dp = find_dns_addr(dns);

	if (dp) {
		//lock()
		list_del(&dp->list);
		//ulock()
		free(dp);
		return 0;
	}

	return -1;
}

static void add_dns_list(dns_addr_t *dns)
{
	//lock()
	list_add(&dns->list, &dns_head);
	//ulock()
}

static char *ip2str(struct sockaddr_in *addr)
{
	static char ipbuf[24];
	inet_ntop(AF_INET, &addr->sin_addr, ipbuf, 16);
	sprintf(ipbuf, "%s:%d", ipbuf, ntohs(addr->sin_port));
	return ipbuf;
}

static void * dns_manager_handler(void *data)//DNS相关问题
{
	int sfd = (int)data;//可以看出如何用于pthread参数传递
	int len;
	char buf[1000];
	struct dnsmsg_s *dmsg;
	dns_addr_t *pdns;

	struct sockaddr_in addr;
	int addrlen = sizeof(addr);

	for (;;) {
		len = recvfrom(sfd, &buf, sizeof(buf), 0,//讲的是在固定的端口接受信息
				(struct sockaddr *)&addr, (socklen_t *)&addrlen);
		if (len <= 0) {
			LOG_MSG(1, errno, "dns_manager_handler recvfrom");
			continue;
		}
		if (len != sizeof(*dmsg)) {//如果接受到的没有dmsg的长度，则说明接收的并非是dns消息
			LOG_MSG(1, 0, "not's dns message");
			continue;
		}
		LOG_MSG(1, 0, "udp recvfrom: %s", ip2str(&addr));

		dmsg = (struct dnsmsg_s *)buf;
		if (!strcmp(dmsg->type, DNSMSG_ADDDNS)) {//判断DNS消息的类型
			if ((pdns = find_dns_addr(dmsg->dns)) != NULL) {
				pdns->addr = dmsg->addr;//这个是收到DNS添加消息时，更新DNS所对应的地址
				LOG_MSG(1, 0, "modfied dns %s[%s] succeed",
						dmsg->dns, ip2str(&pdns->addr));
				continue;
			}

			pdns = (dns_addr_t *)malloc(sizeof(dns_addr_t));
			strcpy(pdns->dns, dmsg->dns);
			pdns->addr = dmsg->addr;

			add_dns_list(pdns);
			LOG_MSG(1, 0, "add dns %s[%s] succeed",
					pdns->dns, ip2str(&pdns->addr));
			//SEND ADD SUCCEED
			continue;
		}

		if (!strcmp(dmsg->type, DNSMSG_DELDNS)) {//删除DNS消息
			if (del_dns_list(dmsg->dns) < 0) {
				LOG_MSG(1, 0, "del dns %s failed", dmsg->dns);
				//SEND DEL FAILED;
			}
			else {
				LOG_MSG(1, 0, "del dns %s succeed", dmsg->dns);
				//SEND DEL SUCCEED
			}
			continue;
		}
	}
}

static int init_dns_manager(void)//用于DNS查询
{
	int sfd;
	pthread_t pid;
	struct sockaddr_in addr;
	int addrlen = sizeof(addr);

	if ((sfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
		LOG_MSG(0, errno, "init_dns_manager socket failed");
		return -1;
	}

	addr.sin_addr.s_addr = htonl(INADDR_ANY);
	addr.sin_family = AF_INET;
	addr.sin_port = htons(DNS_BIND_PORT);
	if (bind(sfd, (struct sockaddr *)&addr, addrlen) < 0) {
		LOG_MSG(0, errno, "init_dns_manager bind failed");
		close(sfd);
		return -1;
	}

	if (pthread_create(&pid, NULL, dns_manager_handler, (void *)sfd) == -1) {
		LOG_MSG(0, errno, "failed to create dns_manager_handler thread");
		close(sfd);
		return -1;
	}

	return 0;
}

int main(int argc, char *argv[])
{
	int listen_fd, webfd;
	struct sockaddr_in addr;
	int addrlen = sizeof(addr);

	log_level = 3;
	if (argc < 2) {
		LOG_MSG(0, 0, "missing port");
		return 1;
	}

	//daemon(1, 0);
	if (init_thread_pool(MAX_CONCURRENT) < 0) {
		LOG_MSG(0, 0, "init_thread_pool failed");
		return 1;
	}

	if ((listen_fd = make_listen_socket(atoi(argv[1]))) < 0) {
		LOG_MSG(0, 0, "make_listen_socket failed");
		return 1;
	}

	/*if (init_dns_manager() < 0) {
		LOG_MSG(0, 0, "init_dns_manager failed");
		close(listen_fd);
		return 1;
	}*/

	while (1) {
		if ((webfd = accept(listen_fd, 
						(struct sockaddr *)&addr,
						(socklen_t *)&addrlen)) < 0) {
			LOG_MSG(0, errno, "accept");
			continue;
		}
		LOG_MSG(3, 0, "tcp connect form: %s", ip2str(&addr));

		if (start_transmit(webfd) < 0) { //开始转发
			LOG_MSG(0, 0, "start_transmit fail");
			close(webfd);
		}
	}

	return 0;
}
