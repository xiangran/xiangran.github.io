---
layout: post
title: STM32 & FreeRTOS学习
date: 2018-05-12
categories: blog
tags: [STM32,FreeRTOS]
description: 

---
#系统时钟节拍
configTICK_RATE_HZ  1000 表示1KHZ节拍，1ms
configUSE_TIME_SLICING 打开task时间片管理

同优先级根据时间片管理，运行指定的时间片(systick)
不同优先级根据抢占式管理

>>时钟周期： 72M晶振，一个时钟周期就是1/72us
机器周期： 完成一个指令需要若干工作，没一个工作为一个基本操作，完成一个基本操作的时间 是机器周期
指令周期： 执行一条指令所需要的时间，一般由若干个机器周期组成。

由于 Cortex-M3 和 M4 内核具有双堆栈指针，MSP 主堆栈指针和 PSP 进程堆栈指针，或者叫 PSP
任务堆栈指针也是可以的。在 FreeRTOS 操作系统中，主堆栈指针 MSP 是给系统栈空间使用的，进
程堆栈指针 PSP 是给任务栈使用的。 也就是说，在 FreeRTOS 任务中，所有栈空间的使用都是通过
PSP 指针进行指向的。 一旦进入了中断函数以及可能发生的中断嵌套都是用的 MSP 指针。

#优先级
当所有task都高于timer 和idle时，如果一直有未完成的工作，timer不会运行

#Timer
考虑平台硬件定时器个数限制的， FreeRTOS 通过一个 Daemon 任务（启动调度器时自动创建）管理软定时器， 满足用户定时需求. Daemon 任务会在其执行期间检查用户启动的时间周期溢出的定时器，并调用其回调函数。

对于硬件定时器的中断服务程序， 我们知道不应该在里面执行复杂，可能导致阻塞的工作，相应的， 虽然软定时器实际是在定时Daemon 任务中执行，但是阻塞的话会导致其他定时器调用被延时， 所以实际使用也应该避免。

软定时器是通过一个任务来辅助实现，该功能时刻裁剪的 ， 只有设置 FreeRTOSConfig.h 中configUSE_TIMERS == 1 将相关代码编译进来, 才能正常使用相关功能。

##配置定时器服务任务
程序中需要使用到软件定时器， 需要先在 FreeRTOSConfig.h 中正确配置如下宏 ： 
* configUSE_TIMERS 
是否编译定时器相关代码， 如需要使用定时器， 设置为 1 
* configTIMER_TASK_PRIORITY 
设置定时器Daemon 任务优先级， 如果优先级太低， 可能导致定时器无法及时执行 
* configTIMER_QUEUE_LENGTH 
设置定时器Daemon 任务的命令队列深度， 设置定时器都是通过发送消息到该队列实现的。 
* configTIMER_TASK_STACK_DEPTH 
设置定时器Daemon 任务的栈大小

##创建 启动 停止定时器

```c
TimerHandle_t xTimerUser; // 定义句柄

// 定时器回调函数格式
void vTimerCallback( TimerHandle_t xTimer )
{
    // do something no block
    // 获取溢出次数
    static uin32_t ulCount = ( uint32_t ) pvTimerGetTimerID( xTimer );
    // 累积溢出次数
    ++ulCount; 
    // 更新溢出次数
    vTimerSetTimerID( xTimer, ( void * ) ulCount );

    if （ulCount == 10） {
        // 停止定时器
        xTimerStop( xTimer, 0 );
    }
}

void fun()
{
    // 申请定时器， 配置
    xTimerUser = xTimerCreate
                   /*调试用， 系统不用*/
                   ("Timer's name",
                   /*定时溢出周期， 单位是任务节拍数*/
                   100,   
                   /*是否自动重载， 此处设置周期性执行*/
                   pdTRUE,
                   /*记录定时器溢出次数， 初始化零, 用户自己设置*/
                  ( void * ) 0,
                   /*回调函数*/
                  vTimerCallback);

     if( xTimerUser ！= NULL ) {
        // 启动定时器， 0 表示不阻塞
        xTimerStart( xTimerUser, 0 );
    }
}
```
如上所示， 调用函数 xTimerCreate申请，配置定时器， 通过 xTimerStart 启动定时器， 当定时器计数溢出时， 系统回调注册的函数。

定时器可以设置为一次性 One-shot 或者自动重载 Auto-reload 两种， 第一种溢出后停止定时器， 第二种溢出后会再次启动定时器。
![image_1cb90la41184u1agv1fgmve15f59.png-38.1kB][1]


##修改定时器
在申请定时器的时候设置的定时器周期， 可以通过函数 xTimerChangePeriod 修改， 如下示例 ：
```c
void vAFunction_2( TimerHandle_t xTimer )
 {
     // 判断定时器是否处于运行状态
     if( xTimerIsTimerActive( xTimer ) != pdFALSE )
     {
         /* xTimer is active, do something. */
     }
     else
     {
         // 处于这个状态的定时器， 可能由于 ： 
         // 1 定时器 create 后没有start
         // 2 一次性定时器执行溢出后

         // 修改定时器周期
         if( xTimerChangePeriod( xTimer, 
                /*修改定时周期*/
                500 / portTICK_PERIOD_MS, 
            /*允许阻塞最大时间 100 ticks*/
            100 ) == pdPASS )
         {
             // update fail
             // 阻塞 100 tick 仍然无法发送命令

             // 删除定时器 释放对应内存！
             xTimerDelete( xTimer );
         }
         else 
         {
             // 定时器配置更新成功， 并已经启动 ！！
         }
    } 
 }
```

如上， 该函数会修改定时器并使定时器 开始运行！！！

另外， 可以通过函数 xTimerReset 重启定时器， 如果已经启动计数， 重新开始计数; 如果没有启动，启动定时器。
>定时器使用系统提供 API，涉及 Queue 操作， 如果是在中断程序中调用，需要调用对应带 FromISR的接口。

##获取定时器状态
其他获取定时器信息的函数
```c
// 获取名称 ， 申请是设置的字符串
pcTimerGetName()
// 定时器溢出周期
xTimerGetPeriod()
// 返回定时器溢出的时间点 （--> xTaskGetTickCount()）
xTimerGetExpiryTime()
```

##定时器实现
FreeRTOS 软定时器的实现在源码目录 Source/include/timers.h, 涉及 链表 和 消息队列(后续文章分析)。

##数据结构
使用定时器前，需要先申请定时器， 见 配置定时器服务任务 中， 通过函数 xTimerCreate获取一个定时器， 实际上是向系统申请了一块内存存储定时器控制块的数据结构, 并将参数填写到该结构体中。

##定时器控制块
![image_1cb90q1qv1tdlctn11hq1ncj1kk416.png-29.6kB][2]
```c
typedef struct tmrTimerControl
{
    // 定时器名 方便调试
    const char *pcTimerName;
    // 链表项 用于插入定时链表
    ListItem_t xTimerListItem;
    // 定时器中断周期
    TickType_t xTimerPeriodInTicks;
    // 是否自动重置， 如果 =pdFalse 为一次性
    UBaseType_t uxAutoReload;
    // 溢出计数 需自己设置
    void *pvTimerID;
    // 定时器溢出回调函数
    TimerCallbackFunction_t pxCallbackFunction;
    #if( configUSE_TRACE_FACILITY == 1 )
        UBaseType_t uxTimerNumber;
    #endif
    #if( ( configSUPPORT_STATIC_ALLOCATION == 1 ) && 
        ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) )
        // 标记定时器使用的内存， 删除时判断是否需要释放内存
        uint8_t ucStaticallyAllocated;  #endif
} xTIMER;
```
成功申请定时器后， 定时器并没有开始工作， 需要调用函数将该定时器中的 xTimerListItem 插入到定时器管理链表中， Daemon 任务才能在该定时器设定的溢出时刻调用其回调函数。

##定时器管理链表
timers.c 中定义了如下几个链表变量用于管理定时器， 定时器根据其溢出时刻从小到大插入链表进行管理。 
使用两个链表是为了应对系统 TickCount 溢出的问题，在 FreeRTOS 任务调度 系统节拍 介绍过。

```c
PRIVILEGED_DATA static List_t xActiveTimerList1;
PRIVILEGED_DATA static List_t xActiveTimerList2;
// 当前节拍计数器对应的定时器管理链表指针
PRIVILEGED_DATA static List_t *pxCurrentTimerList;
// 溢出时间到了下一个节拍计数阶段(当前节拍计数器溢出后)的定时器管理链表指针 
PRIVILEGED_DATA static List_t *pxOverflowTimerList;
```

##命令队列
文章开头提到的使用定时器的函数， 大部分都带有一个参数，用于设置调用后允许阻塞的最大时间， 原因是， 这些函数并没有直接操作定时器管理链表， 而是向定时器Daemon 任务的消息队列 xTimerQueue 发送消息命令。 之后， 定时器Daemon 任务会从消息队列取出消息并响应操作。

##定时器服务任务
此处，从系统启动的定时器Daemon 任务展开分析 FreeRTOS 的软定时器的实现 。 
该任务主体的执行流程如下所示 ：
![image_1cb910kc018id396dnqhbo1e5f30.png-58.9kB][3]

永久循环部分的代码 :
```c
for( ;; )
{
    // 读取定时器队列第一个链表项的值 -> 即将溢出的定时器时间（ticks）
    // 如果链表空， 返回的是 0
    xNextExpireTime = prvGetNextExpireTime( &xListWasEmpty );

    // 处理溢出的定时器
    // 阻塞直到下一个定时器溢出 或 消息队列有新命令
    prvProcessTimerOrBlockTask( xNextExpireTime, xListWasEmpty );

    // 读取消息队列，执行命令
    prvProcessReceivedCommands();
}
```

##回调定时器
定时器任务中， 取出下一个定时器溢出的时间，并把它传递给函数prvProcessTimerOrBlockTask， 该函数负责处理溢出定时器， 应对节拍计数器溢出问题等， 并设置合适的时间阻塞 Daemon 任务， 让出 CPU 使用权直到下一个定时器溢出或者接收到新的命令。
```c
static void prvProcessTimerOrBlockTask(
    const TickType_t xNextExpireTime,
    BaseType_t xListWasEmpty )
{
    TickType_t xTimeNow;
    BaseType_t xTimerListsWereSwitched;
    // 挂起调度器 避免任务切换
    vTaskSuspendAll();
    {
        // 判断系统节拍计数是否溢出
        // 如果是，处理溢出定时器， 并切换定时器链表
        xTimeNow = prvSampleTimeNow( &xTimerListsWereSwitched );

        // 系统节拍计数器没有溢出
        if( xTimerListsWereSwitched == pdFALSE )
        {
            // 判断是否有定时器溢出
            if( ( xListWasEmpty == pdFALSE ) && ( xNextExpireTime <= xTimeNow ) )
            {
                // 恢复调度
                ( void ) xTaskResumeAll();
                //执行相应定时器的回调函数
                // 对于需要自动重载的定时器， 更新下一次溢出时间， 插回链表
                prvProcessExpiredTimer( xNextExpireTime, xTimeNow );
            }
            else
            {
                // 当前链表没有定时器
                if( xListWasEmpty != pdFALSE )
                {
                    // 判断溢出链表上是否有定时器
                    xListWasEmpty = listLIST_IS_EMPTY( pxOverflowTimerList );
                }
                // 阻塞挂起直到 ： 下一个定时器溢出 或 新命令消息
                // 下面这个queue函数是内核专用， 调用后不会直接阻塞，但是会把任务加入到阻塞链表中
                vQueueWaitForMessageRestricted( xTimerQueue, 
                        ( xNextExpireTime - xTimeNow ), /*转换阻塞时间*/
                        xListWasEmpty );

                if( xTaskResumeAll() == pdFALSE )
                {
                    // 触发任务切换
                    portYIELD_WITHIN_API();
                }
                else
                {
                    mtCOVERAGE_TEST_MARKER();
                }
            }
        }
        else
        {
            // 恢复调度
            ( void ) xTaskResumeAll();
        }
    }
}
```
##处理节拍计数器溢出
上面提到， 通过函数 prvSampleTimeNow判断节拍计数器是否发发生溢出， 并执行相应处理， 此处看看该函数内容 ：
```c
static TickType_t prvSampleTimeNow( BaseType_t * const pxTimerListsWereSwitched )
{
    TickType_t xTimeNow;
    // 静态变量 记录上一次调用时系统节拍值
    PRIVILEGED_DATA static TickType_t xLastTime = ( TickType_t ) 0U;
    // 获取本次调用节拍结束器值
    xTimeNow = xTaskGetTickCount();

    // 判断节拍计数器是否溢出过
    // 比如 8bit : 0xFF+1 -> 0
    if( xTimeNow < xLastTime )
    {
        // 发生溢出， 处理当前链表上所有定时器并切换管理链表
        prvSwitchTimerLists();
        *pxTimerListsWereSwitched = pdTRUE;
    }
    else
    {
        *pxTimerListsWereSwitched = pdFALSE;
    }
    // 更新记录
    xLastTime = xTimeNow;
    return xTimeNow;
}
```
可以看到， 该函数每次调用都会记录节拍值， 下一次调用，通过比较相邻两次调用的值判断节拍计数器是否溢出过。 
当节拍计数器溢出， 需要处理掉当前链表上的定时器（应为这条链表上的定时器都已经溢出了）， 然后切换链表。

对于处理这部分任务的函数， 主要要注意其对于需要重载的定时器的处理 ：
>类比一下 ， 一个自动重载的定时器， 每月需要执行一次， 上次调用是2016 年6月， 之后由于优先级问题，导致下一次调用时间等到第二年2017年 1月了，也就是跨年了（节拍计数器溢出了）， 切换日历（链表）前， 需要把旧的先处理掉， 那么实际该定时器在2016年 7～ 12月每月都需要执行一次，所以要补偿回来，直到第二年1月， 才发送消息，插到新日历里面（链表）。

即使时间延迟了，但是该调用几次，是保证的！！
```c
static void prvSwitchTimerLists( void )
{
    TickType_t xNextExpireTime, xReloadTime;
    List_t *pxTemp;
    Timer_t *pxTimer;
    BaseType_t xResult;

    // 切换链表前， 需要先处理当前链表上的所有执行定时器
    while( listLIST_IS_EMPTY( pxCurrentTimerList ) == pdFALSE )
    {
        // 获取第一个定时器溢出时间
        xNextExpireTime = listGET_ITEM_VALUE_OF_HEAD_ENTRY( pxCurrentTimerList );
        // 取出定时器并从链表移除
        pxTimer = ( Timer_t * ) listGET_OWNER_OF_HEAD_ENTRY( pxCurrentTimerList );
        ( void ) uxListRemove( &( pxTimer->xTimerListItem ) );
        traceTIMER_EXPIRED( pxTimer );
        // 执行定时器回调函数
        pxTimer->pxCallbackFunction( ( TimerHandle_t ) pxTimer );

        // 对于自动重载的定时器 计算下一次溢出时间
        if( pxTimer->uxAutoReload == ( UBaseType_t ) pdTRUE )
        {
            // 如果重载后定时器的时间没有溢出， 还在当前链表范围内， 继续插回到当前链表
            // 保证执行的次数
            xReloadTime = ( xNextExpireTime + pxTimer->xTimerPeriodInTicks );
            if( xReloadTime > xNextExpireTime )
            {
                // 设置下一次溢出时间
                listSET_LIST_ITEM_VALUE( &( pxTimer->xTimerListItem ), xReloadTime );
                listSET_LIST_ITEM_OWNER( &( pxTimer->xTimerListItem ), pxTimer );
                vListInsert( pxCurrentTimerList, &( pxTimer->xTimerListItem ) );
            }
            else
            {
                // 重载后定时器的时间同节拍计数器一样溢出了
                // 需要插入到新的链表中， 通过消息发送
                // 等到处理消息时，链表已经切换了
                xResult = xTimerGenericCommand( 
                    pxTimer, 
                    tmrCOMMAND_START_DONT_TRACE, 
                    xNextExpireTime, 
                    NULL, 
                    tmrNO_DELAY );

                configASSERT( xResult );
                ( void ) xResult;
            }
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }

    // 切换链表
    pxTemp = pxCurrentTimerList;
    pxCurrentTimerList = pxOverflowTimerList;
    pxOverflowTimerList = pxTemp;
}
```
函数 prvProcessTimerOrBlockTask 中， 当节拍计数器没有溢出， 判断当前管理链表上溢出定时器并进行处理的函数 prvProcessExpiredTimer 整体和上面介绍差别不大， 执行函数回调， 判断是否需要重载等。

##命令处理
用户将需要处理的定时器命令发送到定时器的消息队列， Daemon 任务每次执行期间回去读取并执行， 这部分工作有任务主体中的函数 prvProcessReceivedCommands完成， 下面看看这个函数如何实现， 对应平时使用定时器控制函数更加有底。 
以下代码做了简化
```c
static void prvProcessReceivedCommands( void )
{
    DaemonTaskMessage_t xMessage;
    Timer_t *pxTimer;
    BaseType_t xTimerListsWereSwitched, xResult;
    TickType_t xTimeNow;

    while( xQueueReceive( xTimerQueue, &xMessage, tmrNO_DELAY ) != pdFAIL )
    {
        #if ( INCLUDE_xTimerPendFunctionCall == 1 )
        // 延期执行函数命令
        // 执行注册的函数  
        #endif

        // 定时器命令消息
        if( xMessage.xMessageID >= ( BaseType_t ) 0 )
        {
            // 命令处理的定时器
            pxTimer = xMessage.u.xTimerParameters.pxTimer;

            if( listIS_CONTAINED_WITHIN( NULL, &( pxTimer->xTimerListItem ) ) == pdFALSE )
            {
                // 如果定时器已经在链表中， 不管37 21， 移除
                ( void ) uxListRemove( &( pxTimer->xTimerListItem ) );
            }

            // 判断节拍计数器是否溢出过 处理 切换
            // 因为下面可能有新项插入 确保链表对应 
            xTimeNow = prvSampleTimeNow( &xTimerListsWereSwitched );

            switch( xMessage.xMessageID )
            {
                case tmrCOMMAND_START :
                case tmrCOMMAND_START_FROM_ISR :
                case tmrCOMMAND_RESET :
                case tmrCOMMAND_RESET_FROM_ISR :
                case tmrCOMMAND_START_DONT_TRACE :
                    // 以上 ，都是让定时器跑起来
                    // 设置定时器溢出时间并插到链表中
                    if( prvInsertTimerInActiveList( pxTimer,
                        xMessage.u.xTimerParameters.xMessageValue +
                        pxTimer->xTimerPeriodInTicks, xTimeNow,
                         xMessage.u.xTimerParameters.xMessageValue ) 
                         != pdFALSE )
                    {
                        // 处理定时器慢了， 该定时器已经溢出
                        // 赶紧执行其回调就看看函数
                        pxTimer->pxCallbackFunction( ( TimerHandle_t ) pxTimer );
                        // 重载定时器 重新启动
                        if( pxTimer->uxAutoReload 
                            == ( UBaseType_t ) pdTRUE )
                        {
                            xResult = xTimerGenericCommand( pxTimer,
                                 tmrCOMMAND_START_DONT_TRACE,
                                  xMessage.u.xTimerParameters.xMessageValue+pxTimer->xTimerPeriodInTicks,
                                  NULL, tmrNO_DELAY );
                            configASSERT( xResult );
                            ( void ) xResult;
                        }
                    }
                    break;

                case tmrCOMMAND_STOP :
                case tmrCOMMAND_STOP_FROM_ISR :
                    // 停止定时器 开头已经从链表移除
                    // 不需要做其他
                    break;

                case tmrCOMMAND_CHANGE_PERIOD :
                case tmrCOMMAND_CHANGE_PERIOD_FROM_ISR :
                    // 更新定时器配置
                    pxTimer->xTimerPeriodInTicks =
                         xMessage.u.xTimerParameters.xMessageValue;
                    // 插入到管理链表 也就启动了定时器
                    ( void ) prvInsertTimerInActiveList( pxTimer,
                         ( xTimeNow + pxTimer->xTimerPeriodInTicks ),
                         xTimeNow, xTimeNow );
                    break;
                case tmrCOMMAND_DELETE :
                    // 删除定时器
                    // 判断定时器内存是否需要释放（动态的释放）
                    break;
                default :
                    /* Don't expect to get here. */
                    break;
            }
        }
    }
}
```
函数处理定时器，开头不管后面命令是什么，如果定时器原本在运行， 直接移除。



  [1]: http://static.zybuluo.com/xiangran/6214l0xo0lr4mjkv3eh51kys/image_1cb90la41184u1agv1fgmve15f59.png
  [2]: http://static.zybuluo.com/xiangran/g7j0ixccgilqbd3p6k9q56zn/image_1cb90q1qv1tdlctn11hq1ncj1kk416.png
  [3]: http://static.zybuluo.com/xiangran/x4s2zcy0zmpzspag42c4xm6r/image_1cb910kc018id396dnqhbo1e5f30.png