---
title: Timer 定时器技术分享
date: 2020-01-23 15:12:45
tags: timer
categories: Linux
---

## 说点废话
> 不管是客户端`Client`还是服务器`Server`，不论你是从事游戏行业还是互联网行业，在技术上总会涉及到定时器。虽然有的框架系统已经帮你实现，并且提供完美API供你使用，但你真的了解定时器吗？我们不仅要知道如何使用正确的Timer，还得明白定时器的实现原理，要知其所以然。  

## 理解定时器
使用者角度分类：
1. 周期性定时器
    1. 使用 `TCP` 长连接时，客户端需要定时向服务端发送心跳请求
    2. 游戏内系统每日重置功能
    3. 体力回复
    4. ....
   
2. 非周期性定时器
    1.  玩法活动定时开启、关闭
    2.  ...

**当然，大部分非周期性定时器都可以使用周期性定时器实现，即执行一次后立即调用Remove接口即可**

定时器像水和空气一般，普遍存在于各个场景中，一般定时任务的形式表现为：经过固定时间后触发、按照固定频率周期性触发、在某个时刻触发。定时器是什么？可以理解为这样一个数据结构：**存储一系列的任务集合，并且 `Deadline` 越接近的任务，拥有越高的执行优先级**

支持以下几种操作：
1. `Add New TimerTask` 添加新的定时器
2. `Kill Or Remove TimerTask` 取消或者移除既有定时器任务
3. `Run` 执行

判断一个`TimerTask`是否到期，基本会采用轮询的方式，每隔一个**时间片**`tickDuration`去检查最近的任务是否到期。

> **说到底，定时器还是靠线程轮询实现的。**

现在知道`Timer`是靠轮询来实现的，那么中间应该采用那种数据结构呢？采用不同的数据结构实现，其性能也大不一样！
现在主要有如下几种：`List`链表、`Heap`最小堆、时间轮、分级时间轮，其中时间轮的实质为Hash表。

## 数据结构

### 双向有序链表
`AddTimer O(N) `很容易理解，按照 `expireTime` 查找合适的位置即可；`KillTimer O(1)` ，任务在 `Kill` 时，会持有自己节点的引用，所以不需要查找其在链表中所在的位置，即可实现当前节点的删除;`RunTimer O(1)`，由于整个双向链表是基于 expireTime 有序的，所以调度器只需要轮询第一个任务即可。

### 最小堆
最小堆指的是满足除了根节点以外的每个节点都不小于其父节点的堆。这样，堆中的最小值就存放在根节点中，并且在以某个结点为根的子树中，各节点的值都不小于该子树根节点的值。一个最小堆的例子如下图：
![最小堆](https://www.ibm.com/developerworks/cn/linux/l-cn-timers/image002.jpg)

明显的，最小堆添加新元素或者删除节点效率为`O(lgn)`, `root`节点`expireTime`最小，执行优先级最高，因此复杂度为O(1)

**如果程序中的定时器数量比较少，基于最小堆的定时器一般可以满足需求，且实现简单。**

### 时间轮
时间轮的实质为哈希环`HashTable`,每个定时器任务根据对其`expireTime`哈希，得到对应的位置`index`，复杂度为`O(1)`
![时间轮](http://kirito.iocoder.cn/201807171109599678a80c-075a-40ee-b25f-10fd82c1025c.png)

**性能比较：**
| 实现方式 |	AddTimer	| KillTimer	 | RunTimer|
|-------- |------------| -----------| -----------|
| 基于链表  |	O(1)	| O(n)	| O(n)|
|基于排序链表	|O(n)	|O(1)	|O(1)|
|基于最小堆     |	O(lgn)|	O(lgn)|	O(1)|
|基于时间轮	|O(1)	|O(1)	|O(1)|


*现在看起来我们选择时间轮来实现就行了，是否这样就完事了？*

## 着重分析时间轮

如果需要支持的定时器范围非常的大，上面的实现方式则不能满足这样的需求。因为这样将消耗非常可观的内存，假设需要表示的定时器范围为：0 – 2^3-1ticks，则简单时间轮需要 2^32 个元素空间，这对于内存空间的使用将非常的庞大。也许可以降低定时器的精度，使得每个 Tick 表示的时间更长一些，但这样的代价是定时器的精度将大打折扣。

现在的问题是，度量定时器的粒度，只能使用唯一粒度吗？想想日常生活中常遇到的水表，如下图：
![水表](https://www.ibm.com/developerworks/cn/linux/l-cn-timers/image004.jpg)

钟表：
![水表](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1556440651&di=0c604969bfdf6b335dd78462f862743e&imgtype=jpg&er=1&src=http%3A%2F%2Famuseum.cdstm.cn%2FAMuseum%2Ftime%2F01gzsj%2Fimages%2F0102_b.jpg)

分级时间轮同样如此，每级时间轮所代表的粒度精度都不一样，这样结合起来，既能够保证定时器的精度，也能以较小内存代价表示范围更大更多的定时器。

**简单时间轮**： 一个齿轮，每个齿轮保存一个超时的node链表。一个齿轮表示一个时间刻度，比如钟表里面一小格代表一秒，钟表的秒针每次跳一格。假设一个刻度代表10ms，则2^32 个格子可表示1.36年，2^16个格子可表示10.9分钟。当要表示的时间范围较大时，空间复杂度会大幅增加。

**分级时间轮**： 类似于水表，当小轮子里的指针转动满一圈后，上一级轮子的指针进一格。  采用五个轮子每个轮子为一个简单时间轮，大小分别为 2^8， 2^6， 2^6， 2^6， 2^6，所需空间：2^8 + 2^6 + 2^6 + 2^6 + 2^6 = 512， 可表示的范围为 0  --  2^8 * 2^6 * 2^6* 2^6* 2^6 = 2^32 。

**分级时间轮简洁图**：
![分级时间轮](http://kirito.iocoder.cn/7f03c027b1de345a0b1e57239d73de74.png)

*熟知的Linux系统内核，定时器实现方式就是分级时间轮*
![Linux内核](https://images0.cnblogs.com/i/205989/201405/281755599318276.jpg)

## 具体实现
wheel_timer_mgr.h
```cpp
class CWheelTimerModule;
class ITimerMgr;
class CWheelTimer;

typedef std::list<CWheelTimer*> TListTimer;

enum ETimerType 
{ 
	ETIMER_ONCE, 
	ETIMER_CIRCLE 
};

///定时器最小精度 1/10秒 （100毫秒）
static const int  WHEEL_TIMER_MIN_PRECISION = 100;

/**
* @brief 基于分级的时间轮定时器, 精度约定为十分之一秒
* 注意，注册的Timer精度必须为约定的精度倍数
*/
class CWheelTimer
{
public:
	CWheelTimer();
	CWheelTimer(CWheelTimerModule& oModule);
	~CWheelTimer();

	/**
	* @brief 启动定时器
	* @param nInterval: 传入的是毫秒, 约定必须是十分之一秒（100ms）的倍数
	* @return 
	*/
	void Start(ITimerMgr* pITimer, unsigned int nId, unsigned nInterval, int nDelay, ETimerType eTimerType = ETIMER_CIRCLE);

	/**
	* @brief 停止定时器
	*/
	void Stop();
private:
	/**
	* @brief 定时器被触发
	*/
	void OnTrigger(const UINT64 nNow);

private:
	friend class CWheelTimerModule;

	CWheelTimerModule&		m_oModule;
	ITimerMgr*				m_pTimerMgr = nullptr;
	ETimerType				m_eTimerType;
	unsigned int			m_nTimerId = 0;
	unsigned int			m_nInterval = -1;    //ms
	unsigned long long		m_llExpireTime = 0;  //ms
	int						m_nVecIndex = 0;
	TListTimer::iterator	m_listIter;
};

/**
* @brief 定时器管理器接口， 派生类继承使用
*/
class ITimerMgr
{
public:
	virtual		  ~ITimerMgr();
	virtual void  OnTimer(unsigned int nId) = 0;
	//interval 时间精度 ms
	void		  SetTimer(unsigned int nId, int nInterval, int nDelay = 0, ETimerType eTimeType = ETIMER_CIRCLE);
	void		  KillTimer(unsigned int nId);
	bool		  IsTimerExist(unsigned int nId);;

protected:
	std::unordered_map<unsigned int, CWheelTimer*> m_mapTimer;
};

/**
* @brief 全局定时器管理模块， 负责管理所有的定时器
*/
class CWheelTimerModule : public Storm::TSingleton<CWheelTimerModule>
{
	friend class Storm::TSingleton<CWheelTimerModule>;
	CWheelTimerModule();
public:
	void			AddTimer(CWheelTimer* pTimer);
	void			RemoveTimer(CWheelTimer* pTimer);
	/**
	* @brief 驱动所有的定时器
	*/
	void			Run();

	static UINT64	GetCurMillisecs();
	/**
	* @brief 修正精度 
	* @param nSrcTime 
	* @return 传入的时间除以当前约定最小定时器精度
	*/
	static UINT64	HandlePrecision(const UINT64 nSrcTime);

private:
	int				_Cascade(int nOffset, int nIndex);

private:
	std::vector<TListTimer>		m_vecTimerList;

	//notice: precision=100ms not 1ms
	UINT64						m_llCheckTime;
};

/**
* @brief 定时器工厂 
*/
class CTimerFactory: public Storm::TSingleton<CTimerFactory>, public CNoncopy
{
	friend class Storm::TSingleton<CTimerFactory>;
	CTimerFactory();
	virtual			~CTimerFactory();
public: 
	CWheelTimer*	CreateCTimer();
	void			ReleaseCTimer(CWheelTimer* pTimer);
private:

	CSTObjectPool<CWheelTimer>    m_oCTimerPool;
}
```

wheel_timer_mgr.cpp
```cpp
#if 0 
	#define TVN_BITS 6
	#define TVR_BITS 8
	#define TVN_SIZE (1 << TVN_BITS)
	#define TVR_SIZE (1 << TVR_BITS)
	#define TVN_MASK (TVN_SIZE - 1)
	#define TVR_MASK (TVR_SIZE - 1)
	#define OFFSET(N) (TVR_SIZE + (N) *TVN_SIZE)
	#define INDEX(V, N) ((V >> (TVR_BITS + (N) *TVN_BITS)) & TVN_MASK)
#else 
	static const int TVN_BITS = 6;
	static const int TVR_BITS = 8;
	static const int TVN_SIZE = (1 << TVN_BITS);
	static const int TVR_SIZE = (1 << TVR_BITS);
	static const int TVN_MASK = (TVN_SIZE - 1);
	static const int TVR_MASK = (TVR_SIZE - 1);
	int OFFSET(int N) { return (TVR_SIZE + (N)*TVN_SIZE); }
	int INDEX(unsigned long long V, int N)
	{
		return ((V >> (TVR_BITS + (N)*TVN_BITS)) & TVN_MASK);
	}

#endif



CWheelTimer::CWheelTimer()
	:m_oModule(CWheelTimerModule::GetInstance())
	, m_nVecIndex(-1)
{
}

CWheelTimer::CWheelTimer(CWheelTimerModule& oModule)
	: m_oModule(oModule)
	, m_nVecIndex(-1)
{
}

CWheelTimer::~CWheelTimer()
{
	Stop();
}

void CWheelTimer::Start(ITimerMgr* pTimerMgr, unsigned int nId, unsigned nInterval, int nDelay, ETimerType eTimerType)
{
	Stop();

	//时间都修正为最小精度
	if (nInterval < WHEEL_TIMER_MIN_PRECISION)
	{
		nInterval = WHEEL_TIMER_MIN_PRECISION;
	}

	m_nInterval = CWheelTimerModule::HandlePrecision(nInterval);
	m_eTimerType = eTimerType;
	m_pTimerMgr = pTimerMgr;
	m_nTimerId = nId;
	m_llExpireTime = CWheelTimerModule::HandlePrecision(nDelay + CWheelTimerModule::GetCurMillisecs());
	m_oModule.AddTimer(this);
}


void CWheelTimer::Stop()
{
	if (m_nVecIndex != -1)
	{
		m_oModule.RemoveTimer(this);
		m_nVecIndex = -1;
	}
}

void CWheelTimer::OnTrigger(const UINT64 nNow)
{
	if (m_eTimerType == ETIMER_CIRCLE)
	{
		m_llExpireTime = m_nInterval + nNow;
		m_oModule.AddTimer(this);
	}
	else
	{
		m_nVecIndex = -1;
	}

	if (m_pTimerMgr != nullptr)
	{
		m_pTimerMgr->OnTimer(m_nTimerId);
	}
}

//--------------------------------------------------------------------------------------------------------

CWheelTimerModule::CWheelTimerModule()
{
	m_vecTimerList.resize(TVR_SIZE + 4 * TVN_SIZE);
	m_llCheckTime = HandlePrecision(GetCurMillisecs());
}

void CWheelTimerModule::AddTimer(CWheelTimer* pTimer)
{
	UINT64 llExpireTime = pTimer->m_llExpireTime;
	INT64 llTimeDiff = pTimer->m_llExpireTime - m_llCheckTime;

	if (llTimeDiff < 0)
	{
		pTimer->m_nVecIndex = m_llCheckTime & TVR_MASK;
	}
	else if (llTimeDiff < TVR_SIZE)
	{
		pTimer->m_nVecIndex = llExpireTime & TVR_MASK;
	}
	else if (llTimeDiff < 1 << (TVR_BITS + TVN_BITS))
	{
		pTimer->m_nVecIndex = OFFSET(0) + INDEX(llExpireTime, 0);
	}
	else if (llTimeDiff < 1 << (TVR_BITS + 2 * TVN_BITS))
	{
		pTimer->m_nVecIndex = OFFSET(1) + INDEX(llExpireTime, 1);
	}
	else if (llTimeDiff < 1 << (TVR_BITS + 3 * TVN_BITS))
	{
		pTimer->m_nVecIndex = OFFSET(2) + INDEX(llExpireTime, 2);
	}
	else
	{
		if (llTimeDiff > 0xffffffffUL)
		{
			llTimeDiff = 0xffffffffUL;
			llExpireTime = llTimeDiff + m_llCheckTime;
		}
		pTimer->m_nVecIndex = OFFSET(3) + INDEX(llExpireTime, 3);
	}

	TListTimer& listTimer = m_vecTimerList[pTimer->m_nVecIndex];
	listTimer.push_back(pTimer);
	pTimer->m_listIter = listTimer.end();
	--pTimer->m_listIter;
}

void CWheelTimerModule::RemoveTimer(CWheelTimer* pTimer)
{
	TListTimer& listTimer = m_vecTimerList[pTimer->m_nVecIndex];
	listTimer.erase(pTimer->m_listIter);
}

void CWheelTimerModule::Run()
{
	UINT64 nNow = HandlePrecision(GetCurMillisecs());
	while (m_llCheckTime <= nNow)
	{
		//for every tick
		int index = m_llCheckTime & TVR_MASK;
		if (!index &&
			!_Cascade(OFFSET(0), INDEX(m_llCheckTime, 0)) &&
			!_Cascade(OFFSET(1), INDEX(m_llCheckTime, 1)) &&
			!_Cascade(OFFSET(2), INDEX(m_llCheckTime, 2)))
		{
			_Cascade(OFFSET(3), INDEX(m_llCheckTime, 3));
		}

		
		++m_llCheckTime;

		TListTimer& listTimer = m_vecTimerList[index];
		TListTimer listTmp;
		listTmp.splice(listTmp.end(), listTimer);
		for (auto itr = listTmp.begin(); itr != listTmp.end(); ++itr)
		{
			auto* pTimer = *itr;
			if (pTimer != nullptr)
			{
				pTimer->OnTrigger(nNow);
			}
		}
	}
}

int CWheelTimerModule::_Cascade(int nOffset, int nIndex)
{
	TListTimer& listTimer = m_vecTimerList[nOffset + nIndex];
	TListTimer listTemp;
	listTemp.splice(listTemp.end(), listTimer);

	for (auto itr = listTemp.begin(); itr != listTemp.end(); ++itr)
	{
		auto* pTimer = *itr;
		if (pTimer != nullptr)
		{
			AddTimer(pTimer);
		}
	}

	return nIndex;
}

UINT64 CWheelTimerModule::HandlePrecision(const UINT64 nSrcTime)
{
	return nSrcTime / WHEEL_TIMER_MIN_PRECISION;
}

UINT64 CWheelTimerModule::GetCurMillisecs()
{
	auto llCurTime = CSysTime::Instance()->GetNowMliTime();
	return llCurTime;
}

ITimerMgr::~ITimerMgr()
{
	for (auto it : m_mapTimer)
	{
		if (it.second != nullptr)
		{
			it.second->Stop();
			CTimerFactory::Instance()->ReleaseCTimer(it.second);
		}
	}
	m_mapTimer.clear();
}

void ITimerMgr::SetTimer(unsigned int nId, int nInterval, int nDelay ,ETimerType eTimeType)
{
	if (IsTimerExist(nId))
	{
		EXLOG_DEBUG << "[RyzTimer]Timer Has Existed, Not Repeat Add, nId:" << nId;
		return;
	}

	CWheelTimer* pTimer = CTimerFactory::Instance()->CreateCTimer();
	if (nullptr == pTimer)
	{
		return;
	}

	m_mapTimer[nId] = pTimer;
	pTimer->Start(this, nId, nInterval, nDelay, eTimeType);
	EXLOG_DEBUG << "[RyzTimer]Add Timer nId:" << nId << ",nInterval:" << nInterval << ",nDelay:" << nDelay << ",eTimeType:" << eTimeType;
}

void ITimerMgr::KillTimer(unsigned int nId)
{
	auto it = m_mapTimer.find(nId);
	if (it != m_mapTimer.end())
	{
		// 释放后 在TimerManager中 不会再执行 不需要做其他的操作
		it->second->Stop();
		CTimerFactory::Instance()->ReleaseCTimer(it->second);
		m_mapTimer.erase(it);
	}
}

bool ITimerMgr::IsTimerExist(unsigned int nId) 
{ 
	return m_mapTimer.find(nId) != m_mapTimer.end(); 
}

CTimerFactory::CTimerFactory()
{
	m_oCTimerPool.Init(32, 8);
}

CTimerFactory::~CTimerFactory()
{
}

CWheelTimer * CTimerFactory::CreateCTimer()
{
	CWheelTimer* pTimer = m_oCTimerPool.FetchObj();
	return pTimer;
}

void CTimerFactory::ReleaseCTimer(CWheelTimer* pTimer)
{
	if (nullptr != pTimer)
	{
		m_oCTimerPool.ReleaseObj(pTimer);
	}
}


```