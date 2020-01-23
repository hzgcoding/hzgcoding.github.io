---
title: 全局唯一ID生成算法优化
date: 2020-01-23 15:12:42
tags: uuid
categories: C/C++
---

<!-- toc -->

- [前言](#前言)
- [Snowflake算法介绍](#Snowflake算法介绍)
- [SnowFlake算法优化](#SnowFlake算法优化)
  * [目前算法设计缺陷](#目前算法设计缺陷)
  * [设计优化](#设计优化)
- [是否这样就完美了呢？](#是否这样就完美了呢？)
- [Talk is cheap, show you the code](#Talk-is-cheap-show-you-the-code)
- [参考文章](#参考文章)

<!-- tocstop -->

## 前言

在进程启动前，一般会给每个进程静态分配一个唯一标识ID(ServerID或者PipeID)
```
struct SServerID
{
	UINT16 wPlatform = 0;	 ///< Platform   平台id 
	UINT16 wArea = 0;        ///< Area       区服id
	UINT16 wType = 0;        ///< Type       服务器App类型
	UINT16 wIndex = 0;       ///< Index      服务器App编号索引
};
```

每个Role/Item/Hero/Mail 等创建的时候都会创建一个UUID来唯一标识，其中Mail_Uuid还有趋势递增的需求

## Snowflake算法介绍

*SnowFlake分布式生成Id算法由Twitter开源*

SnowFlake算法生成id的结果是一个64bit大小的UINT64整数，它的结构如下图：
![snowflake uuid-64bit](:storage\1a51b319-5d9b-40f9-a828-cfc88309cea2\4dbfceed.png)

SnowFlake的优点: 
	- 整体上按照时间自增排序
	- 整个分布式系统内不会产生ID碰撞(时间戳和自增序列以外的字段作区分), 并且效率较高(位运算),

SnowFlake每秒能够产生6.4万ID左右.(和5段位的配置位数有关)
UUID从高位到低位依次排列：
- 第一段：39位, 当对于某一个时间点的时间戳差值（至少10年可用）
- 第二段：3 位, 平台platform或大区id，比如QQ Android/QQ IOS/Wechat IOS 等（8）
- 第三段：11位, 区服area id，对应的就是服务器小区的ID（2048）
- 第四段：5 位, 服务器APP实例id index（32), Notice：appid需要在area范围内唯一
- 第五段：6 位, 自增长id，也就是说在一个完整的毫秒时间内最多可以生成64个UUID

## SnowFlake算法优化

### 目前算法设计缺陷
在实际服务器运行过程中，尤其在游戏服务器开发期间，大量用户注册，会有瞬间生成大量UUID的需求

之前服务器的做法是：
```cpp
//遍历了一圈64个，等下一个毫秒生成
if (m_nGlobalSeq == m_nMliSeq)
{
	// 毫秒内序列溢出, 阻塞到下一个毫秒,获得新的时间戳
	nCurTimestamp = WaitForNextMilli(m_nLastTimestamp);
}

/**
* 阻塞到下一个毫秒，直到获得新的时间戳
*
* @param lastTimestamp 上次生成ID的时间截
* @return 当前时间戳
*/
UINT64 WaitForNextMilli(UINT64 lastTimestamp) const
{
	UINT64 nTimestamp = _GetNowMliTime();
	while (nTimestamp <= lastTimestamp)
	{
		nTimestamp = _GetNowMliTime();
	}
	return nTimestamp;
}

```
这里使用while强行等待到下一毫秒，相当于阻塞当前线程，从而成为热点函数，需要去优化

### 设计优化
- 合理分配各字段占用bit（图中已经是调整过的）
- 将生成uuid单独出一个独立全局Server组件，提供全局唯一UUID服务，这样除去高位39bit外理论上其他bit都可以当作自增位，而且可以提前生成很多uuid放在pool当中，有需要的进程从当中取，可以满足需求，Baidu在github上开源的UUID生成算法就是这样处理的
- 我们取一个折中方案，只是在server中加一个Pool，存放提前生成好的UUIDs，每次业务端需要UUID的时候首先从Pool中取，如果取不到就走原来的流程

```cpp
UINT64 GenId()
{
	if (!IsEmpty())
	{
		//从Pool中取
		return PopFrontElement();
	}

	return NextId();
}
```

既然存在Pool，那么就需要设计Pool中元素填充方案
首先将Pool设定一个合适的固定最大值 const UINT32 UUID_POOL_MAX_SIZE = 1 << 13;   //		uuid pool 最大数 8192
然后根据当前Pool的状态，来定时填充，每次填充的数量为当前毫秒内所有可以生成的UUID数
![Fill_Pool.png](:storage\1a51b319-5d9b-40f9-a828-cfc88309cea2\928daf4a.png)


```cpp
#include "uuid_pool_mgr.h"
#include "uuid_generator.h"


bool CUUIDPool::Init()
{
	FillUuidPool();
	return true;
}

void CUUIDPool::OnTimer(unsigned int nId)
{
	EXLOG_DEBUG << "[RyzUuid]CUUIDPool::OnTimer Old EUuidPoolState : " << nId;
	CUUIDMaker::Instance()->FillPoolWithInMli();
	FillUuidPool();
}

void CUUIDPool::FillUuidPool()
{
	EUuidPoolState eUuidDequeState = CUUIDMaker::Instance()->GetUuidDequeState();
	uint32 nUpdateInterval = 5 * 60 * 1000;
	switch (eUuidDequeState)
	{
	case EUuidPoolState_Empty:
		nUpdateInterval = 1 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_0_30_Per:
		nUpdateInterval = 2 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_30_70_Per:
		nUpdateInterval = 3 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_70_100_Per:
		nUpdateInterval = 4 * 60 * 1000;
		break;
	case EUuidPoolState_Full:
		nUpdateInterval = 5 * 60 * 1000;
		break;
	default:
		nUpdateInterval = 5 * 60 * 1000;
		break;
	}

	EXLOG_DEBUG << "[RyzUuid]CUUIDPool::OnTimer Now EUuidPoolState : " << eUuidDequeState;
	SetTimer(eUuidDequeState, nUpdateInterval, nUpdateInterval, ETIMER_ONCE);
}


enum EUuidPoolState
{
	EUuidPoolState_Empty				= 1,
	EUuidPoolState_Not_Full_0_30_Per	= 2,
	EUuidPoolState_Not_Full_30_70_Per	= 3,
	EUuidPoolState_Not_Full_70_100_Per	= 4,
	EUuidPoolState_Full					= 5,
};

//uuid_generator.h
//将当前毫秒内的UUID全部生成并存到Pool中
void FillPoolWithInMli()
{
	EUuidPoolState nCurState = GetUuidDequeState();
	AtomicUInt64 nCurTimestamp = _GetNowMliTime();
	while (nCurState != EUuidPoolState_Full)
	{
		UINT64 nNextUUId = NextId(false);
		if (nNextUUId == 0)
		{
			break;
		}

		PushBackElement(nNextUUId);

		if (nCurTimestamp != _GetNowMliTime())
		{
			break;
		}
	}
}

```

## 是否这样就完美了呢？

并没有呢！！！
这个算法强制依赖时间递增，如果时间回拨怎么办？
目前的做法是直接throw new exception
分析时间回拨产生原因
第一：人物操作，在真实环境一般不会有那个傻逼干这种事情，所以基本可以排除。 
第二：由于有些业务等需要，机器需要同步时间服务器（在这个过程中可能会存在时间回拨，查了下我们服务器一般在10ms以内（2小时同步一次））。 Ntp过程可能产生时间回拨。
第三：QA和策划测试过程中有需求怎么办？
解决办法：
1. 将uuid_generation独立出来给其他server提供服务
2. 当回拨时间小于XXms，就等时间追上来之后继续生成。 (XXms对业务没有什么影响)
3. 当时间大于XXms时间我们通过更换AppId位来来解决回拨问题。

## Talk is cheap, show you the code
```cpp
#ifndef __UUID_GENERATOR_H__
#define __UUID_GENERATOR_H__

#include "gnsingleton.h"
#include "gntype.h"
#include "gnpipe.h"
#include "gntime.h"
#include "noncopy.h"
#include "gnserverid.h"

#include <assert.h>
#include <mutex>
#include <atomic>
#include <chrono>
#include <exception>
#include <sstream>
#include <deque>

enum EUuidPoolState
{
	EUuidPoolState_Empty				= 1,
	EUuidPoolState_Not_Full_0_30_Per	= 2,
	EUuidPoolState_Not_Full_30_70_Per	= 3,
	EUuidPoolState_Not_Full_70_100_Per	= 4,
	EUuidPoolState_Full					= 5,
};

//#define SNOWFLAKE_ID_MAKER_NO_LOCK
class CSnowflakeIdMaker : private CNoncopy
{
public:
#ifdef SNOWFLAKE_ID_MAKER_NO_LOCK
	typedef std::atomic<UINT32> AtomicUInt;
	typedef std::atomic<UINT64> AtomicUInt64;
#else
	typedef UINT32 AtomicUInt;
	typedef UINT64 AtomicUInt64;
#endif

	const UINT32 UUID_POOL_MAX_SIZE = 1 << 13;   //		uuid pool 最大数 8192
	const UINT64 START_EPOCH			= 1541001600000LL;		//开始时间截 (2018-11-01 00:00:00.000)，修改此时间可调整可用时长
							   
	const UINT32 A_TIMESTAMP_BITS		= 39;					//时间戳所占的位数
	const UINT32 B_PLATFORM_BITS		= 3;					//平台id所占的位数
	const UINT32 C_AREA_BITS			= 11;					//区服id所占的位
	const UINT32 D_APP_ID_BITS			= 5;					//app id所占的位数
	const UINT32 E_INCR_SEQUENCE_BITS	= 6;				    //自增序列所占的位数

	const UINT32 APP_ID_SHIFT			= E_INCR_SEQUENCE_BITS;		//APPID向左移位数
	const UINT32 AREA_ID_SHIFT			= E_INCR_SEQUENCE_BITS + D_APP_ID_BITS;											//小区id向左移位数
	const UINT32 PLATFORM_ID_SHIFT		= E_INCR_SEQUENCE_BITS + D_APP_ID_BITS + C_AREA_BITS;						//大区id向左移位数
	const UINT32 TIME_STAMP_SHIFT		= E_INCR_SEQUENCE_BITS + D_APP_ID_BITS + C_AREA_BITS + B_PLATFORM_BITS;		//时间戳向左移位数
	const UINT32 SEQUENCE_MASK			= (1 << E_INCR_SEQUENCE_BITS) - 1;												//生成序列的掩码


	CSnowflakeIdMaker() : m_nPlatformId(0), m_nAreaId(0), m_nGlobalSeq(0), m_nLastTimestamp(0) {}

	CSnowflakeIdMaker(const UINT32 nPlatId, const UINT32 nAreaId, const UINT32 nAppId)
	{
		Init(nPlatId, nAreaId, nAppId);
	}

	void Init(const UINT32 nPlatId, const UINT32 nAreaId, const UINT32 nAppId)
	{
		m_nPlatformId = nPlatId;
		m_nAreaId = nAreaId;
		m_nAppId = nAppId;
	}

	UINT64 GenId()
	{
		if (!IsEmpty())
		{
			return PopFrontElement();
		}

		return NextId();
	}

	/**
	* 获得下一个ID (该方法是线程安全的)
	* @param  bCanBlock 参数指定当前函数是否可以阻塞, 默认为true
	* @return SnowflakeId
	*/
	UINT64 NextId(bool bCanBlock = true)
	{
		using namespace  std;

#ifdef SNOWFLAKE_ID_MAKER_NO_LOCK
		static AtomicUInt64 nCurTimestamp{ 0 };
#else
		std::unique_lock<std::mutex> oLock{ m_oMutex };
		AtomicUInt64 nCurTimestamp{ 0 };
#endif

		nCurTimestamp = _GetNowMliTime();

		// 如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
		if (nCurTimestamp < m_nLastTimestamp)
		{
			std::ostringstream oSS;
			oSS << "clock moved backwards.  Refusing to generate id for " << m_nLastTimestamp - nCurTimestamp << " milliseconds";
			throw std::exception(std::runtime_error(oSS.str()));
		}

		m_nGlobalSeq = (m_nGlobalSeq + 1) & SEQUENCE_MASK;

		// 为了使id递增+1均匀分布，这里seq跨毫秒也不清0
		if (m_nLastTimestamp == nCurTimestamp)
		{
			//遍历了一圈64个，等下一个毫秒生成
			if (m_nGlobalSeq == m_nMliSeq)
			{
				if (bCanBlock)
				{
					// 毫秒内序列溢出, 阻塞到下一个毫秒,获得新的时间戳
					nCurTimestamp = WaitForNextMilli(m_nLastTimestamp);
				}
				else
				{
					return 0;
				}
			}
		}
		else
		{	
			m_nMliSeq = m_nGlobalSeq;
		}

#ifdef SNOWFLAKE_ID_MAKER_NO_LOCK
		m_nLastTimestamp = nCurTimestamp.load();
#else
		m_nLastTimestamp = nCurTimestamp;
#endif

		// 移位并通过或运算拼到一起组成64位的ID
		return ((nCurTimestamp - START_EPOCH) << TIME_STAMP_SHIFT)
			| (m_nPlatformId << PLATFORM_ID_SHIFT)
			| (m_nAreaId << AREA_ID_SHIFT)
			| (m_nAppId << APP_ID_SHIFT)
			| (m_nGlobalSeq);
	}

	EUuidPoolState GetUuidDequeState() 
	{
		if (m_deUuidPool.empty())
		{
			return EUuidPoolState_Empty;
		}

		size_t nCurPoolSize = m_deUuidPool.size();
		if (nCurPoolSize >= UUID_POOL_MAX_SIZE)
		{
			return EUuidPoolState_Full;
		}

		uint32 nCurPercent = nCurPoolSize * 100 / UUID_POOL_MAX_SIZE;
		if (nCurPercent < 30)
		{
			return EUuidPoolState_Not_Full_0_30_Per;
		}
		else if (nCurPercent < 70)
		{
			return EUuidPoolState_Not_Full_30_70_Per;
		}
		else
		{
			return EUuidPoolState_Not_Full_70_100_Per;
		}

		return EUuidPoolState_Full;
	} 

	void FillPoolWithInMli()
	{
		EUuidPoolState nCurState = GetUuidDequeState();
		AtomicUInt64 nCurTimestamp = _GetNowMliTime();
		while (nCurState != EUuidPoolState_Full)
		{
			UINT64 nNextUUId = NextId(false);
			if (nNextUUId == 0)
			{
				break;
			}
			
			PushBackElement(nNextUUId);

			if (nCurTimestamp != _GetNowMliTime())
			{
				break;
			}
		}
	}

protected:

	//判断pool是否为空
	bool IsEmpty()
	{
		return m_deUuidPool.empty();
	}

	//返回pool队列中front 元素
	UINT64 PopFrontElement()
	{
#ifdef SNOWFLAKE_ID_MAKER_NO_LOCK
		
#else
		std::unique_lock<std::mutex> oLock{ m_oMutex };
#endif

		UINT64 nFrontElement = m_deUuidPool.front();
		m_deUuidPool.pop_front();
		return nFrontElement;
	}

	void PushBackElement(UINT64 nNextUUId)
	{
#ifdef SNOWFLAKE_ID_MAKER_NO_LOCK
#else
		std::unique_lock<std::mutex> oLock{ m_oMutex };
#endif
		m_deUuidPool.push_back(nNextUUId);
	}

	/**
	* 返回以毫秒为单位的当前时间
	*
	* @return 当前时间(毫秒)
	*/
	UINT64 _GetNowMliTime() const
	{
		if (0)
		{
			Storm::CSTDateTime oDateTime;
			oDateTime.Now();
			return oDateTime.EpochMilliSecs();
		}
		else
		{
			using namespace std;
			auto nTimeNow = chrono::system_clock::now();
			auto nDurationInMs = chrono::duration_cast<chrono::milliseconds>(nTimeNow.time_since_epoch());
			return nDurationInMs.count();
		}
	}

	/**
	* 阻塞到下一个毫秒，直到获得新的时间戳
	*
	* @param lastTimestamp 上次生成ID的时间截
	* @return 当前时间戳
	*/
	UINT64 WaitForNextMilli(UINT64 lastTimestamp) const
	{
		UINT64 nTimestamp = _GetNowMliTime();
		while (nTimestamp <= lastTimestamp)
		{
			nTimestamp = _GetNowMliTime();
		}
		return nTimestamp;
	}

private:

#ifndef SNOWFLAKE_ID_MAKER_NO_LOCK
	std::mutex		m_oMutex;
#endif

	UINT32			m_nPlatformId = 0;		//平台id
	UINT32			m_nAreaId = 0;			//区服id
	UINT32			m_nAppId = 0;			//Appid
	AtomicUInt		m_nGlobalSeq{ 0 };		//全局序列
	AtomicUInt		m_nMliSeq{ 0 };			//每毫秒序列
	AtomicUInt64	m_nLastTimestamp{ 0 };	//上次生成ID的时间截
	std::deque<UINT64> m_deUuidPool;			//uuid池 用于存放预生成uuids
};



/************************************************************************/
/* 负责生成全局唯一id												*/
/************************************************************************/
class CUUIDMaker : public Storm::TSingleton<CUUIDMaker>
{
	friend class Storm::TSingleton<CUUIDMaker>;
public:
	bool Init(const UINT32 nParaA, const UINT32 nParaB, const UINT32 nParaC)
	{
		m_oIdMaker.Init(nParaA, nParaB, nParaC);
		return true;
	}
	
	/// init with pipeid
	bool Init(const UINT64 nPipeId)
	{
		using namespace Storm;
		CServerID oServID(nPipeId);
		m_oIdMaker.Init(oServID.GetPlat(), oServID.GetArea(), oServID.GetIndex());
		return true;
	}

	UINT64 GenId()
	{
		return m_oIdMaker.GenId();
	}

	EUuidPoolState GetUuidDequeState()
	{
		return m_oIdMaker.GetUuidDequeState();
	}

	void FillPoolWithInMli()
	{
		m_oIdMaker.FillPoolWithInMli();
	}

	UINT64 GetCompareIdFromTime(UINT32 nTimeVal);

	UINT64 GetTimeAddVal(UINT32 nTimeVal);

	UINT32 GetTimeValFromUuid(UINT64 nUuid);

private:
	CSnowflakeIdMaker	m_oIdMaker;
};

#define  GEN_GLOBAL_UUID()  CUUIDMaker::Instance()->GenId()


#endif


//gameserver 
#include "uuid_pool_mgr.h"
#include "uuid_generator.h"


bool CUUIDPool::Init()
{
	FillUuidPool();
	return true;
}

void CUUIDPool::OnTimer(unsigned int nId)
{
	EXLOG_DEBUG << "[RyzUuid]CUUIDPool::OnTimer Old EUuidPoolState : " << nId;
	CUUIDMaker::Instance()->FillPoolWithInMli();
	FillUuidPool();
}

void CUUIDPool::FillUuidPool()
{
	EUuidPoolState eUuidDequeState = CUUIDMaker::Instance()->GetUuidDequeState();
	uint32 nUpdateInterval = 5 * 60 * 1000;
	switch (eUuidDequeState)
	{
	case EUuidPoolState_Empty:
		nUpdateInterval = 1 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_0_30_Per:
		nUpdateInterval = 2 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_30_70_Per:
		nUpdateInterval = 3 * 60 * 1000;
		break;
	case EUuidPoolState_Not_Full_70_100_Per:
		nUpdateInterval = 4 * 60 * 1000;
		break;
	case EUuidPoolState_Full:
		nUpdateInterval = 5 * 60 * 1000;
		break;
	default:
		nUpdateInterval = 5 * 60 * 1000;
		break;
	}

	EXLOG_DEBUG << "[RyzUuid]CUUIDPool::OnTimer Now EUuidPoolState : " << eUuidDequeState;
	SetTimer(eUuidDequeState, nUpdateInterval, nUpdateInterval, ETIMER_ONCE);
}

```
## 参考文章
[分布式唯一id：snowflake算法思考 - 掘金](https://juejin.im/post/5a7f9176f265da4e721c73a8)
[https://tech.meituan.com/2017/04/21/mt-leaf.html](https://tech.meituan.com/MT_Leaf.html)
[GitHub - baidu/uid-generator: UniqueID generator](https://github.com/baidu/uid-generator)