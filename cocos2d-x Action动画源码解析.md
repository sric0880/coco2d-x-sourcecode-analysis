# cocos2d-x Action动画 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-12 12:05  -->

对于动作的使用说明见[coco2d-x 官方wiki](http://cn.cocos2d-x.org/doc/cocos-docs-master/manual/framework/native/v2/graphic/action/res/inherent.png?t=111)  
官方结构图：  
![image](http://cn.cocos2d-x.org/doc/cocos-docs-master/manual/framework/native/v2/graphic/action/res/inherent.png?t=111)

***********

###原理
核心代码：
Director
```c++
//Director初始化时就开始循环调用_actionManager的update方法
bool Director::init(void)
{
  ...
  _actionManager = new ActionManager();
  _scheduler->scheduleUpdate(_actionManager, Scheduler::PRIORITY_SYSTEM, false);
  ...
}
```
Scheduler
```c++
template <class T>
void scheduleUpdate(T *target, int priority, bool paused)
{
	//每一帧调用target的update方法，具体如何每一帧调用参见另一篇<cocos2d-x Shader源码解析>
	this->schedulePerFrame([target](float dt){
	    target->update(dt);
	}, target, priority, paused);
}
    
Node
```c++
Node::Node()
{
  ...
  _actionManager = director->getActionManager();
  ...
}

Action * Node::runAction(Action* action)
{
    CCASSERT( action != nullptr, "Argument must be non-nil");
    _actionManager->addAction(action, this, !_running);
    return action;
}

void Node::stopAllActions()
{
    _actionManager->removeAllActionsFromTarget(this);
}

void Node::stopAction(Action* action)
{
    _actionManager->removeAction(action);
}

void Node::stopActionByTag(int tag)
{
    CCASSERT( tag != Action::INVALID_TAG, "Invalid tag");
    _actionManager->removeActionByTag(tag, this);
}

Action * Node::getActionByTag(int tag)
{
    CCASSERT( tag != Action::INVALID_TAG, "Invalid tag");
    return _actionManager->getActionByTag(tag, this);
}
```
ActionManager:
```c++
void ActionManager::addAction(Action *action, Node *target, bool paused)
{
    CCASSERT(action != nullptr, "");
    CCASSERT(target != nullptr, "");
    //将target封装成element，并保存到hash表中去
    tHashElement *element = nullptr;
    // we should convert it to Ref*, because we save it as Ref*
    Ref *tmp = target;
    HASH_FIND_PTR(_targets, &tmp, element);
    if (! element)
    {
        element = (tHashElement*)calloc(sizeof(*element), 1);
        element->paused = paused;
        target->retain();
        element->target = target;
        HASH_ADD_PTR(_targets, target, element);
    }

     actionAllocWithHashElement(element);

     CCASSERT(! ccArrayContainsObject(element->actions, action), "");
     ccArrayAppendObject(element->actions, action);

     action->startWithTarget(target);//哈哈调用Action的start方法了
}

//最关键的代码
// main loop
//循环遍历hash表，如果target没有暂停、target中的action不为nullptr
//就执行该action的step方法
//如果action->isDone()==true，就stop action，并且将
//action从ActionManager中移除
void ActionManager::update(float dt)
{
    for (tHashElement *elt = _targets; elt != nullptr; )
    {
        _currentTarget = elt;
        _currentTargetSalvaged = false;

        if (! _currentTarget->paused)
        {
            // The 'actions' MutableArray may change while inside this loop.
            for (_currentTarget->actionIndex = 0; _currentTarget->actionIndex < _currentTarget->actions->num;
                _currentTarget->actionIndex++)
            {
                _currentTarget->currentAction = (Action*)_currentTarget->actions->arr[_currentTarget->actionIndex];
                if (_currentTarget->currentAction == nullptr)
                {
                    continue;
                }

                _currentTarget->currentActionSalvaged = false;

                _currentTarget->currentAction->step(dt);

                if (_currentTarget->currentActionSalvaged)
                {
                    // The currentAction told the node to remove it. To prevent the action from
                    // accidentally deallocating itself before finishing its step, we retained
                    // it. Now that step is done, it's safe to release it.
                    _currentTarget->currentAction->release();
                } else
                if (_currentTarget->currentAction->isDone())
                {
                    _currentTarget->currentAction->stop();

                    Action *action = _currentTarget->currentAction;
                    // Make currentAction nil to prevent removeAction from salvaging it.
                    _currentTarget->currentAction = nullptr;
                    removeAction(action);
                }

                _currentTarget->currentAction = nullptr;
            }
        }

        // elt, at this moment, is still valid
        // so it is safe to ask this here (issue #490)
        elt = (tHashElement*)(elt->hh.next);

        // only delete currentTarget if no actions were scheduled during the cycle (issue #481)
        if (_currentTargetSalvaged && _currentTarget->actions->num == 0)
        {
            deleteHashElement(_currentTarget);
        }
    }

    // issue #635
    _currentTarget = nullptr;
}
```
####Action Class
`Action`保存了一个`Node* _target`的指针  
```c++
/**
@brief Base class for Action objects.
 */
class CC_DLL Action : public Ref, public Clonable
{
public:
    /// Default tag used for all the actions
    static const int INVALID_TAG = -1;
    /**
     * @js NA
     * @lua NA
     */
    virtual std::string description() const;

	/** returns a clone of action */
	virtual Action* clone() const = 0;

    /** returns a new action that performs the exactly the reverse action */
	virtual Action* reverse() const = 0;

    //! return true if the action has finished
    virtual bool isDone() const;

    //! called before the action start. It will also set the target.
    virtual void startWithTarget(Node *target);

    /**
    called after the action has finished. It will set the 'target' to nil.
    IMPORTANT: You should never call "[action stop]" manually. Instead, use: "target->stopAction(action);"
    */
    virtual void stop();

    //! called every frame with it's delta time. DON'T override unless you know what you are doing.
    virtual void step(float dt);

    /**
    called once per frame. time a value between 0 and 1

    For example:
    - 0 means that the action just started
    - 0.5 means that the action is in the middle
    - 1 means that the action is over
    */
    virtual void update(float time);

    inline Node* getTarget() const { return _target; }
    /** The action will modify the target properties. */
    inline void setTarget(Node *target) { _target = target; }

    inline Node* getOriginalTarget() const { return _originalTarget; }
    /** Set the original target, since target can be nil.
    Is the target that were used to run the action. Unless you are doing something complex, like ActionManager, you should NOT call this method.
    The target is 'assigned', it is not 'retained'.
    @since v0.8.2
    */
    inline void setOriginalTarget(Node *originalTarget) { _originalTarget = originalTarget; }

    inline int getTag() const { return _tag; }
    inline void setTag(int tag) { _tag = tag; }

protected:
    Action();
    virtual ~Action();

    Node    *_originalTarget;
    /** The "target".
    The target will be set with the 'startWithTarget' method.
    When the 'stop' method is called, target will be set to nil.
    The target is 'assigned', it is not 'retained'.
    */
    Node    *_target;
    /** The action tag. An identifier of the action */
    int     _tag;

private:
    CC_DISALLOW_COPY_AND_ASSIGN(Action);
};
```
`FiniteTimeAction`继承`Action`，多了一个`_duration`属性
####ActionInterval
`ActionInterval`继承`FiniteTimeAction`，它的`_duration`必须大于0。  
它的`step`实现：
```c++
void ActionInterval::step(float dt)
{
    if (_firstTick)
    {
        _firstTick = false;
        _elapsed = 0;
    }
    else
    {
        _elapsed += dt;
    }
    //关键代码，保证update(float dt)中的dt在[0,1]之间
    this->update(MAX (0,                                  // needed for rewind. elapsed could be negative
                      MIN(1, _elapsed /
                          MAX(_duration, FLT_EPSILON)   // division by 0
                          )
                      )
                 );
}
//在时间小于_duration时，step是不会停止的
bool ActionInterval::isDone() const
{
    return _elapsed >= _duration;
}
```
所有继承于`Action`的动画都会实现如下方法：
```c++
virtual RotateTo* clone() const override;
virtual RotateTo* reverse() const override;
virtual void startWithTarget(Node *target) override; //初始化target，以及动画相关的参数
virtual void update(float time) override; //根据时间不停地变化参数，并通过target->set***函数传递给target
```
例子：
- **TargetedAction**: 将`FiniteTimeAction* _action`绑定到它的一个成员`_forcedTarget`上，动画执行时，
  将会调用`_action->startWithTarget(_forcedTarget);`，一个node可以通过TargetedAction使另外一个node调用Action。
- **Animate**：包含`Animation* _animation`里面含有许多AnimationFrame，通过`update`函数不停的调用`target->setSpriteFrame`
  来切换图片，达到播放帧动画的效果
- **ReverseTime**：倒着执行动画
```c++
void ReverseTime::update(float time)
{
    if (_other)
    {
        _other->update(1 - time); //time 取值在[0-1]之间
    }
}
```
包装一个`FiniteTimeAction* _other`，逆向时间调用update函数。实现反向动画。
- **Spawn**: 同时执行多个动画，持续时间为所有动画中最大的那个
```c++
//多个动画递归调用构造
for (int i = 1; i < arrayOfActions.size(); ++i)
{
    prev = createWithTwoActions(prev, arrayOfActions.at(i));
}
//Spawn类只含有两个action
void Spawn::update(float time)
{
    if (_one)
    {
        _one->update(time);
    }
    if (_two)
    {
        _two->update(time);
    }
}
```
- **SkewTo/SkewBy**: 保证面积相等，倾斜
- **RotateTo**: 保证周长相等，倾斜  
实现方法：RotationalSkewTo,RotationalSkewBy
- **TintTo**：颜色渐变
- **ActionCamera/OrbitCamera**: 移动摄像机。`OrbitCamera`继承自`ActionCamera`，不能直接调用，因为`ActionCamera`没有重写update方法，只能使用它的子类。
`OrbitCamera`update方法：
```c
void OrbitCamera::update(float dt)
{
    float r = (_radius + _deltaRadius * dt) * FLT_EPSILON;
    float za = _radZ + _radDeltaZ * dt;
    float xa = _radX + _radDeltaX * dt;

    float i = sinf(za) * cosf(xa) * r + _center.x;
    float j = sinf(za) * sinf(xa) * r + _center.y;
    float k = cosf(za) * r + _center.z;

    setEye(i,j,k); //调用父类的方法，调整摄像机的位置
}
```
- **CardinalSplineTo/CardinalSplineBy**：见[Cardinal spline wikipedia](http://en.wikipedia.org/wiki/Cubic_Hermite_spline#Cardinal_spline)定义
，以Point Array为参数，原理根据曲线插值计算坐标，通过update更新_target的坐标实现。
插值函数见`Vec2 ccCardinalSplineAt(Vec2 &p0, Vec2 &p1, Vec2 &p2, Vec2 &p3, float tension, float t)`
- **CatmullRomTo/CatmullRomBy**: 见[Catmull–Rom spline wikipedia](http://en.wikipedia.org/wiki/Cubic_Hermite_spline#Catmull.E2.80.93Rom_spline)定义
，继承`CardinalSplineTo/CardinalSplineBy`，是Cardinal Spline tension为0.5时的特例
- **ActionTween**: 可以让target的属性width2秒内从200变成300，target必须继承`ActionTweenDelegate`并实现`updateTweenAction`方法
```c
auto modifyWidth = ActionTween::create(2, "width", 200, 300);
target->runAction(modifyWidth);
```
- **其他参见**
  >`cocos2d-x ActionEase源码解析`  
  `cocos2d-x  进度条源码解析`  
  `cocos2d-x 网格动画源码解析`

###ActionInstant 瞬时动作
`ActionInstant`继承`FiniteTimeAction`，没有添加新的属性
```c++
class CC_DLL ActionInstant : public FiniteTimeAction //<NSCopying>
{
public:
    //
    // Overrides
    //
	virtual ActionInstant* clone() const override = 0;
    virtual ActionInstant * reverse() const override = 0;
    virtual bool isDone() const override;
    virtual void step(float dt) override;
    virtual void update(float time) override;
};
//始终返回true,说明第一次执行后就已经结束了
bool ActionInstant::isDone() const
{
    return true;
}
//因为只调用一次step,所以update传最大值1
void ActionInstant::step(float dt) {
    CC_UNUSED_PARAM(dt);
    update(1);
}
```
例子：
- **Hide/Show**: 有reverse方法，可以将隐藏的对象变成不隐藏(或相反)
```c
void Hide::update(float time) {
    CC_UNUSED_PARAM(time);
    _target->setVisible(false);
}
```
- **ToggleVisibility**：如果是可见就变成不可见，如果是不可见就变成可见
```c
void ToggleVisibility::update(float time)
{
    CC_UNUSED_PARAM(time);
    _target->setVisible(!_target->isVisible());
}
```
- **CCPlace**：放置某一个position
- **CallFunc/CallFuncN**：执行一个回调，可以是一个成员函数指针，可以是一个function对象
- **RemoveSelf**:

###Follow
不是`FiniteTimeAction`，直接继承`Action`，可以使一个layer跟着一个node走，这时node绝对位置不变，layer作为背景以相反方向移动

核心代码：
```c
void Follow::step(float dt)
{
    CC_UNUSED_PARAM(dt);

    if(_boundarySet) //接近世界的边缘
    {
        // whole map fits inside a single screen, no need to modify the position - unless map boundaries are increased
        if(_boundaryFullyCovered)
            return;

        Vec2 tempPos = _halfScreenSize - _followedNode->getPosition();

        _target->setPosition(Vec2(clampf(tempPos.x, _leftBoundary, _rightBoundary),
                                   clampf(tempPos.y, _bottomBoundary, _topBoundary)));
    }
    else
    {
        //人物如果移动，那么反方向移动世界地图，人物相对屏幕中点不动
        _target->setPosition(_halfScreenSize - _followedNode->getPosition());
    }
}

bool Follow::isDone() const
{
    return ( !_followedNode->isRunning() );//只要_followedNode没有onExit，那么该函数一直return false.
}
```

###Speed
不是`FiniteTimeAction`，直接继承`Action`，包装一个`IntervalAction`，使其动画播放的速度改变（>1变快，<1变慢）

核心代码：
```c
void Speed::step(float dt)
{
    _innerAction->step(dt * _speed);
}
```
