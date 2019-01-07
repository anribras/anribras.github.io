---
layout: post
title:
modified:
categories: Tech
 
tags: [cplusplus]

  
comments: true
---
<!-- TOC -->

- [vector印象](#vector印象)
- [Buffer.h](#bufferh)

<!-- /TOC -->

### vector印象
自增长内存，连续存储，这两点就足够代替c中是malloc数组了。

也是遇到了个实际的例子，音频解码接收buffer,需要自增长内存，同时又要很方便的取出。

下面直接贴上写好的bufffer，有几点：

* 自增长内存
* 读取时不需要pop, 因内存连续可用&vec[i]直接寻访地址;
* 需要取出时，仍然可以用memcpy直接取出,然后erase掉。
* 用shrink_to_fit减少富余的分配空间。
* 批量存入时可采样的方法:
```
std::copy(static_cast<const T*>(raw), 
					static_cast<const T*>((const T*)raw+len), 
					back_inserter(inbuff)
					);
```

### Buffer.h
```
#ifndef __SPECIALIZED_BUFFER_H
#define __SPECIALIZED_BUFFER_H

#include <vector>
#include <algorithm>
#include <iterator>
#include <memory>

extern "C"
{
#include <pthread.h>
#include <stdio.h>
#include <string.h>
#include "ecolink_log.h"
}

#define CAPACITY_MAX (20*1024)

/** @brief SpecializedBuffer  */
//添加数据 vector无限增长，确保不丢失数据
//锁的范围控制在最小，有最好的速度
//refill机制，
template<typename T>
class Buffer 
{
	private:
		void lock() {
			pthread_mutex_lock(&inbuff_mutex);
		}
		void unlock() {
			pthread_mutex_unlock(&inbuff_mutex);
		}
		void notify() {
			pthread_cond_signal(&inbuff_cond);
		}
		void wait() {
			pthread_cond_wait(&inbuff_cond, &inbuff_mutex);
		}
		std::vector<T> inbuff;
		pthread_mutex_t inbuff_mutex;
		pthread_cond_t inbuff_cond;
public:
		T& operator[](long i) {
			return inbuff[i];
		}
		Buffer(){
			inbuff_mutex = PTHREAD_MUTEX_INITIALIZER;
			inbuff_cond =  PTHREAD_COND_INITIALIZER ;
		}
		~Buffer(){
			free();
		}

		std::shared_ptr<std::vector<T>> get_vec() {
			return std::make_shared<std::vector<T>>(inbuff);
		}

		long capacity(){
			return inbuff.capacity();
		}
		long size() {
			return inbuff.size();
		}

		int put(const void* raw, long len) {
			lock();

			//LOGI("put into (%d) ",len);
			std::copy(static_cast<const T*>(raw), 
					static_cast<const T*>((const T*)raw+len), 
					back_inserter(inbuff)
					);

			unlock();
			notify();
			return 0;
		}


		void free() {
			inbuff.clear();
			inbuff.shrink_to_fit();
		}

		// drain_flag 决定read后是否擦除容器数据
		int fetch(long pos, T* data,long len , int drain_flag) {
			if( pos < 0 || len < 0 ) {
				LOGI("set pos or len error");
				return -1;
			}
			if(inbuff.size() < (len)  || 
					inbuff.size() < (pos+len)) {
				LOGI("wana from(%ld) get (%ld) but only(%ld) total, wait",
						pos, len, inbuff.size());
				wait();
			}
			
			memcpy(data, &inbuff[pos], len * sizeof(T)); 
			if(drain_flag)
				inbuff.erase(inbuff.begin()+(long)pos,inbuff.begin() + (long)(pos+len));
			return 0;
		}


		int fetch_from_head(T* data,long len ) {
			lock();
			int ret = fetch(0,data, len,0);
			unlock();
			return ret;
		}

		int fetch_from(int pos, T* data,long len ) {
			lock();
			int ret = fetch(pos,data, len,0);
			unlock();
			return ret;
		}

		int drain_from_head(T* data,long len ) {
			lock();
			int ret = fetch(0,data, len,1);
			unlock();
			return ret;
		}

		int drain_from(int pos, T* data,long len ) {
			lock();
			int ret = fetch(pos,data, len,1);
			unlock();
			return ret;
		}

		int drain_out(T* data, long* len) {
			lock();
			*len =  inbuff.size();
			int ret =fetch(0,data,*len,1);
			unlock();
			return ret;
		}

		int erase(long start, long end) {
			lock();
			if(start > end) {
				LOGI("start > end when erase");
				return -1;
			}
			inbuff.erase(inbuff.begin() + start,
					inbuff.begin() + end);

			//LOGI("going to shrinking cap(%d) size(%d)",inbuff.capacity(), inbuff.size());
			if(inbuff.capacity() > CAPACITY_MAX && 
				inbuff.capacity() > inbuff.size() * 2){
				LOGI("shrinking");
				inbuff.shrink_to_fit();
			}
			unlock();
			return 0;
		}

		int wait_till_size(long size) {
			lock();
			if(inbuff.size() <= size)  {
			   //LOGI("wait till ( size > %d)",size);
			   wait();	
			}
			unlock();
			return 0;
		}

		int wait_not_empty() {
			return wait_till_size(1);
		}
};
#endif
```
