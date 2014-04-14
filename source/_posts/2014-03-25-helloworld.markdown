---
layout: post
title: "Linux下使用hugepage"
date: 2014-03-25 18:15:37 +0800
comments: true
categories: 
---

如果应用程序使用的内存比较大的时候，就可以考虑使用大内存。好处有可以减少tlb的miss，还有缺页中断次数会大幅减少。

2.6.32以后的内核支持使用MAP_HUGETLB方式来使用内存,
首先使用如下命令分配和查看大内存页:
```
sjx$ sysctl vm.nr_hugepages=192
sjx$ cat /proc/meminfo | grep Huge

AnonHugePages:         0 kB
HugePages_Total:     192
HugePages_Free:      192
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```
接着写一段C++代码验证一下是否可以正确使用mmap来分配大内存页，以及是否有效果。
```c++
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<sys/time.h>
class timer{
public:
  struct  timeval t;
  timer(){
    gettimeofday(&t,NULL);
  }
};
int get_time_diff(timer& t1, timer& t2){
  return (1000000 * (t2.t.tv_sec-t1.t.tv_sec)+ t2.t.tv_usec-t1.t.tv_usec)/1000;
}


int main(int argc, char **argv) {
  char *m;
  size_t s = (800 * 1024 * 1024);
  m =(char* ) mmap(NULL, s, PROT_READ | PROT_WRITE, 
        MAP_PRIVATE /*MAP_SHARED*/ | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
  if (m == MAP_FAILED) {
    perror("map mem");
    m = NULL;
    return 1;
  }
  timer t1;
  for(int i=0;i<s;i+=4096){
    m[i]=0;
  }
  timer t2;
  munmap(m, s);
  
  m =(char* )malloc(s);
  timer t3;
  for(int i=0;i<s;i+=4096){
    m[i]=0;
  }
  timer t4;
  free(m);

  printf("mmap:%d\n",get_time_diff(t1,t2));
  printf("malloc:%d\n",get_time_diff(t3,t4));
 
  return 0;
}
```
需要注意的是s最好是页面大小的整数倍，有的系统下如果非整数倍会报错。

最后在我的台式机上输出结果如下：主要原因应该是mmap的pagefault次数比较少。
```
mmap:47
malloc:164
```






