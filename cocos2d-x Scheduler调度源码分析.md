# cocos2d-x Scheduler调度 源码分析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-21 11:21  -->

Scheduler能够以一定的频率循环调用某个方法，比如说Action动画就得益于此。

在`Director::drawScene`中，就一直循环地调用`Scheduler::update`
```c
// Draw the Scene
void Director::drawScene()
{
    ...
    if (! _paused)
    {
        //_deltaTime指的是上次调用mainLoop和这次之间经过的时间
        _scheduler->update(_deltaTime); //这里是调度系统驱动的动力
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
    }
    ...
}
```
我们来看看`CCScheduler.h`这个文件，这个文件包括这些类：
- `Timer`
- `TimerTargetSelector`
- `TimerTargetCallback`
- `TimerScriptHandler`
- `Scheduler`

我们首先从`Scheduler::update`切入：
```c
void Scheduler::update(float dt)
{
    _updateHashLocked = true;

    if (_timeScale != 1.0f)
    {
        dt *= _timeScale; //默认为1，将时间进行缩放 <0放慢 >0加速
    }

    // 遍历所有update回调
    tListEntry *entry, *tmp;

    // 遍历优先级<0的list，将值保存在entry中
    DL_FOREACH_SAFE(_updatesNegList, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);//执行回调
        }
    }

    // 遍历优先级=0的list，将值保存在entry中
    DL_FOREACH_SAFE(_updates0List, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);//执行回调
        }
    }

    // 遍历优先级>0的list，将值保存在entry中
    DL_FOREACH_SAFE(_updatesPosList, entry, tmp)
    {
        if ((! entry->paused) && (! entry->markedForDeletion))
        {
            entry->callback(dt);//执行回调
        }
    }

    //以上list保存的都是update selector，执行周期本身和Scheduler::update一致
    //下面是用户自定义的selector，间隔时间并不是dt，而是用户自定义的间隔时间
    for (tHashTimerEntry *elt = _hashForTimers; elt != nullptr; )
    {
        //Hash表中以target为key，一个target可以同时拥有多个Timers
        _currentTarget = elt;
        _currentTargetSalvaged = false;

        if (! _currentTarget->paused)
        {
            //遍历所有Timers
            // The 'timers' array may change while inside this loop
            for (elt->timerIndex = 0; elt->timerIndex < elt->timers->num; ++(elt->timerIndex))
            {
                elt->currentTimer = (Timer*)(elt->timers->arr[elt->timerIndex]);
                elt->currentTimerSalvaged = false;
                //核心函数，后面会讲
                elt->currentTimer->update(dt);

                if (elt->currentTimerSalvaged)
                {
                    // The currentTimer told the remove itself. To prevent the timer from
                    // accidentally deallocating itself before finishing its step, we retained
                    // it. Now that step is done, it's safe to release it.
                    elt->currentTimer->release();
                }

                elt->currentTimer = nullptr;
            }
        }

        // elt, at this moment, is still valid
        // so it is safe to ask this here (issue #490)
        elt = (tHashTimerEntry *)elt->hh.next;

        // only delete currentTarget if no actions were scheduled during the cycle (issue #481)
        if (_currentTargetSalvaged && _currentTarget->timers->num == 0)
        {
            removeHashElement(_currentTarget);
        }
    }

    // 删除标记为deleted的update selector
    // 先从优先级<0的开始
    DL_FOREACH_SAFE(_updatesNegList, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    // 优先级=0
    DL_FOREACH_SAFE(_updates0List, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    // 优先级>0
    DL_FOREACH_SAFE(_updatesPosList, entry, tmp)
    {
        if (entry->markedForDeletion)
        {
            this->removeUpdateFromHash(entry);
        }
    }

    _updateHashLocked = false;
    _currentTarget = nullptr;

  ...

    //
    // 下面的函数是其他线程发送过来的，需要在主线程调用
    //
    if( !_functionsToPerform.empty() ) {
        _performMutex.lock();
        //在_performMutex.unlock()之前调用function()，如果在function()中有新的function加入到该_functionsToPerform
        //那岂不是死锁了，之前写代码我就遇到了这个问题，他们竟然也发现这个bug
        //话说回来，复制到一个temp的解决方案就没更好的方法了嘛
        //当然有更好的方法：使用std::recursive_mutex而不是std::mutex就能解决这种问题
        auto temp = _functionsToPerform;
        _functionsToPerform.clear();
        _performMutex.unlock();
        for( const auto &function : temp ) {
            function();
        }
    }
}
```
看完上述代码，基本上就了解了调度系统的基本原理了。

接下来我们来看看使用调度系统的方法，使用方法的基本思想就是将回调函数和一些参数包装成struct加入到list或hash表中。

调度某个函数：
```c
//每经过interval秒就执行一次callback
//paused - true 直到调用resume才执行
//interval如果等于0， 那么建议调用scheduleUpdate
//callback会执行repeat + 1次，如果是kRepeatForever就不停回调
//key是用来标示这个callback的
void schedule(const ccSchedulerFunc& callback, void *target, float interval, unsigned int repeat, float delay, bool paused, const std::string& key);
//同上 repeat - kRepeatForever，delay = 0
void schedule(const ccSchedulerFunc& callback, void *target, float interval, bool paused, const std::string& key);
//该函数唯一不同于上述的就是它的回调不是std::function，而是一个成员函数指针
void schedule(SEL_SCHEDULE selector, Ref *target, float interval, unsigned int repeat, float delay, bool paused);
//同上 repeat - kRepeatForever，delay = 0
void schedule(SEL_SCHEDULE selector, Ref *target, float interval, bool paused);

//因为我们还不知道包含update函数的类型，所以需要用模板
//对于update函数，它有所谓的优先级，优先级越低越先调用
template <class T>
void scheduleUpdate(T *target, int priority, bool paused)
{
    this->schedulePerFrame([target](float dt){
        target->update(dt);
    }, target, priority, paused);
}
```
`schedule`的基本实现步骤是根据target作为key在hash表中查找，找到的话，就往当前表中的timers数组中插入一个`TimerTargetCallback`类型的`Timer`，
再插入之前，先遍历已经存在的Timers，如果要插入的Timer的key(std::string类型)和其中一个相同，那么就当做一个Timer，只修改这个Timer的interval，
而不是插入新的Timer。

也就是说对于同一个target，即使调用同一个函数，只要key不相同，那么相互之间也没有影响。

对于没有key的SEL_SCHEDULE，selector本身就是一个指针地址，可以用来作为key，也就是说target和selector两者标示了唯一的Timer。

结束调度某个函数，就是add操作的逆操作，不再详细描述
```c
/**  需要target和key才能够定位到一个Timer
 */
void unschedule(const std::string& key, void *target);

/** 需要selector和key才能够定位到一个Timer
 */
void unschedule(SEL_SCHEDULE selector, Ref *target);

/** Unschedules the update selector for a given target
 */
void unscheduleUpdate(void *target);

/** Unschedules all selectors for a given target.
 This also includes the "update" selector.
 */
void unscheduleAllForTarget(void *target);

/** Unschedules all selectors from all targets.
 You should NEVER call this method, unless you know what you are doing.
 */
void unscheduleAll(void);

/** Unschedules all selectors from all targets with a minimum priority.
 You should only call this with kPriorityNonSystemMin or higher.
 */
void unscheduleAllWithMinPriority(int minPriority);
```

关于理解自定义调度事件，理解Timer是关键。
```c
class CC_DLL Timer : public Ref
{
protected:
    Timer();
public:
    /** get interval in seconds */
    inline float getInterval() const { return _interval; };
    /** set interval in seconds */
    inline void setInterval(float interval) { _interval = interval; };

    void setupTimerWithInterval(float seconds, unsigned int repeat, float delay);

    virtual void trigger() = 0;
    virtual void cancel() = 0;

    /** triggers the timer */
    void update(float dt);

protected:

    Scheduler* _scheduler; // weak ref
    float _elapsed;
    bool _runForever;
    bool _useDelay;
    unsigned int _timesExecuted;
    unsigned int _repeat; //0 = once, 1 is 2 x executed
    float _delay;
    float _interval;
};
```
get/set函数都不用看，主要是它记录了总共经历的时间_elapsed，间隔时间_interval，延迟时间_delay，
重复次数_repeat，已经执行了多少次_timesExecuted，已经延迟了多久时间_useDelay。

有了这些参数，我们再来看`Timer::update`方法：
```c
void Timer::update(float dt)
{
    if (_elapsed == -1)//第一次update时调用
    {
        _elapsed = 0; //初始化
        _timesExecuted = 0;
    }
    else
    {
        if (_runForever && !_useDelay) //如果永久循环还没有延迟
        {//standard timer usage
            _elapsed += dt; //累计逝去时间
            if (_elapsed >= _interval) //大于一个周期
            {
                trigger();//这是一个虚函数，需要子类实现，调用回调函数

                _elapsed = 0;//重新计算逝去时间
            }
        }
        else
        {//advanced usage
            _elapsed += dt;
            if (_useDelay) //有延迟时间，仅调用一次
            {
                if( _elapsed >= _delay ) //当逝去时间大于延迟时间
                {
                    trigger(); //执行一次

                    _elapsed = _elapsed - _delay;
                    _timesExecuted += 1; //记录执行次数
                    _useDelay = false; //不会再执行到这里
                }
            }
            else
            {
                if (_elapsed >= _interval)//大于一个周期
                {
                    trigger(); //回调

                    _elapsed = 0;
                    _timesExecuted += 1; //再次记录执行次数

                }
            }

            if (!_runForever && _timesExecuted > _repeat) //若不是永久循环，而且执行次数超过规定次数
            {    //unschedule timer
                cancel(); //调用unschedule方法，Timer结束了
            }
        }
    }
}
```
Timer非常好理解。而他的两个子类不同点在于实现的虚函数：
```c
void TimerTargetSelector::trigger()
{
    if (_target && _selector)
    {
        (_target->*_selector)(_elapsed);
    }
}

void TimerTargetSelector::cancel()
{
    _scheduler->unschedule(_selector, _target);
}

void TimerTargetCallback::trigger()
{
    if (_callback)
    {
        _callback(_elapsed);
    }
}
void TimerTargetCallback::cancel()
{
    _scheduler->unschedule(_key, _target);
}
```
很容易明白，不需要多讲了。

多线程中需要主线程执行某个函数
```c
void Scheduler::performFunctionInCocosThread(const std::function<void ()> &function)
{
    _performMutex.lock();

    _functionsToPerform.push_back(function);

    _performMutex.unlock();
}
```
这个函数和delta时间没有关系，不需要用到计时器。相当于Scheduler的一个辅助功能吧。主要是利用了scheduler的update方法，能够不断的循环检查_functionsToPerform是否有函数需要执行。
