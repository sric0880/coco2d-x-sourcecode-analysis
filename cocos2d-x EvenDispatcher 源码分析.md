# cocos2d-x EventDispatcher 源码分析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-20 12:03  -->

本文介绍新版cocos2d-x中的事件分发机制，最核心的类是`EventDispatcher`
，在`Director::init`初始化时就创建了该类的对象，所以全局只需要维护一个`EventDispatcher`即可，所有事件的监听和事件分发都是靠它完成的。

关于事件机制的类有EventXXX，EventListnerXXX，EventDispatcher这三类，分别表示事件、事件监听类、事件分发类。

##事件
首先介绍一下EventXXX事件，所有事件类都是继承于`Event`，类型包括：
1. touch 触摸
2. keyboard 键盘
3. acceleration 加速器
4. mouse 鼠标
5. focus 焦点
6. game controller 游戏手柄（部分平台才有）
7. custom 自定义事件

下面一一介绍
###EventTouch
成员变量：
```c
enum class EventCode
{
    BEGAN,
    MOVED,
    ENDED,
    CANCELLED
};
...
EventCode _eventCode; //该触摸的状态
//touch的数组
//每一个touch都保存了当前的position，pre-position，start-position
std::vector<Touch*> _touches;  
```
###EventKeyboard
成员变量：
```c
KeyCode _keyCode; //按的是哪个键
bool _isPressed; //是否按下
```
###EventAcceleration
成员变量：
```c
//包装了一个Acceleration类，它包括三个方向的加速度值
Acceleration _acc;
```
###EventMouse
成员变量：
```c
enum class MouseEventType
{
    MOUSE_NONE,
    MOUSE_DOWN,
    MOUSE_UP,
    MOUSE_MOVE,
    MOUSE_SCROLL,
};
MouseEventType _mouseEventType;
int _mouseButton; //左按钮还是右按钮
float _x; //光标的x
float _y; //光标的y
float _scrollX; //滑轮滚动
float _scrollY; //滑轮滚动
```
###EventFocus
成员变量：
```c
ui::Widget *_widgetGetFocus; //获得焦点的widget
ui::Widget *_widgetLoseFocus; //失去焦点的widget
```
###EventCustom
成员变量：
```c
void* _userData;       ///< User data
std::string _eventName; //名称
```

##事件监听类
###EventListener
成员变量：
```c
std::function<void(Event*)> _onEvent;   /// Event callback function

Type _type; /// Event listener type - 包括上述7种事件类型
ListenerID _listenerID; /// Event listener ID不同事件类型不同 - std::string类型
bool _isRegistered; /// Whether the listener has been added to dispatcher.

int   _fixedPriority;   // The higher the number, the higher the priority, 0 is for scene graph base priority.
Node* _node;            // 用于scene graph based priority
bool _paused;           // Whether the listener is paused
bool _isEnabled;        // Whether the listener is enabled
```
该类方法都是关于上述变量的set/get函数。

每种事件类型都有自己的lister监听类，来举几个例子：
###EventListenerCustom
```c
bool EventListenerCustom::init(const ListenerID& listenerId, const std::function<void(EventCustom*)>& callback)
{
    bool ret = false;

    _onCustomEvent = callback; //设置回调
    //将回调包装成父类的回调类型
    auto listener = [this](Event* event){
        if (_onCustomEvent != nullptr)
        {
            _onCustomEvent(static_cast<EventCustom*>(event));
        }
    };
    //初始化父类
    if (EventListener::init(EventListener::Type::CUSTOM, listenerId, listener))
    {
        ret = true;
    }
    return ret;
}
```
其他监听类都需要根据自己的回调的参数，在init函数中对回调函数进行包装，传入父类，实现多态。

###EventListenerTouchOneByOne
```c
class EventListenerTouchOneByOne : public EventListener
{
...
public: //回调函数因为是public的，所以不用写get/set方法
    std::function<bool(Touch*, Event*)> onTouchBegan;
    std::function<void(Touch*, Event*)> onTouchMoved;
    std::function<void(Touch*, Event*)> onTouchEnded;
    std::function<void(Touch*, Event*)> onTouchCancelled;
...
private:
    ...
    std::vector<Touch*> _claimedTouches; //TODO:
    bool _needSwallow; //是否吞没 不再向下传递
    ...
}

bool EventListenerTouchOneByOne::init()
{ //没有给父类传入回调函数
    if (EventListener::init(Type::TOUCH_ONE_BY_ONE, LISTENER_ID, nullptr))
    {
        return true;
    }

    return false;
}
```
###EventListenerTouchAllAtOnce
```c
class EventListenerTouchAllAtOnce : public EventListener
{
public: //回调函数因为是public的，所以不用写get/set方法
    std::function<void(const std::vector<Touch*>&, Event*)> onTouchesBegan;
    std::function<void(const std::vector<Touch*>&, Event*)> onTouchesMoved;
    std::function<void(const std::vector<Touch*>&, Event*)> onTouchesEnded;
    std::function<void(const std::vector<Touch*>&, Event*)> onTouchesCancelled;
...
};
bool EventListenerTouchAllAtOnce::init()
{ //没有给父类传入回调函数
    if (EventListener::init(Type::TOUCH_ALL_AT_ONCE, LISTENER_ID, nullptr))
    {
        return true;
    }
    return false;
}
```
其他类型的监听事件都大同小异

下面介绍主角EventDispatcher

##EventDispatcher
该类主要是用来添加或移除Eventlistener，并在事件产生时，根据listener 中的回调函数
dispatch相应类型的Event。

同样先看成员变量：
```c
/** 所有事件监听对象保存的地方，首先按类型id分类，然后每一个类别下保存一个vector */
std::unordered_map<EventListener::ListenerID, EventListenerVector*> _listenerMap;

/** 标记哪个vector为脏数据 */
std::unordered_map<EventListener::ListenerID, DirtyFlag> _priorityDirtyFlagMap;

/** Scene Graph型的监听类还会根据node进行区分，另外保存到这个vector里面 */
std::unordered_map<Node*, std::vector<EventListener*>*> _nodeListenersMap;

/** 在Scene Graph的vector中的所有nodes的排名
    用在函数sortEventListenersOfSceneGraphPriority
 */
std::unordered_map<Node*, int> _nodePriorityMap;

/** 为了方便_nodePriorityMap的生成，产生的一个临时变量
  在函数visitTarget中使用 */
std::unordered_map<float, std::vector<Node*>> _globalZOrderNodeMap;

/** 如果这时候正在分发事件，新添加的事件暂时保存在这个数组中，有待添加 */
std::vector<EventListener*> _toAddedListeners;

/** 标记为脏数据的node - 只和Scene Graph有关 */
std::set<Node*> _dirtyNodes;

/** 是否正在分发事件 */
int _inDispatch;

/** Whether to enable dispatching event */
bool _isEnabled;

int _nodePriorityIndex;
//内部的自定义事件的listenerID，可以不用管
//主要为了防止清除所有监听事件的时候，将内部自定义的事件清除
std::set<std::string> _internalCustomListenerIDs;
```
再来看看它提供的add函数：
```c
//它默认的priority为0
void EventDispatcher::addEventListenerWithSceneGraphPriority(EventListener* listener, Node* node)
{ //不能重复添加
    CCASSERT(listener && node, "Invalid parameters.");
    CCASSERT(!listener->isRegistered(), "The listener has been registered.");

    if (!listener->checkAvailable()) //是否合法
        return;

    listener->setAssociatedNode(node); //将node传入listener
    listener->setFixedPriority(0);
    listener->setRegistered(true);

    addEventListener(listener);
}
void EventDispatcher::addEventListenerWithFixedPriority(EventListener* listener, int fixedPriority)
{
    CCASSERT(listener, "Invalid parameters.");
    CCASSERT(!listener->isRegistered(), "The listener has been registered.");
    CCASSERT(fixedPriority != 0, "0 priority is forbidden for fixed priority since it's used for scene graph based priority.");

    if (!listener->checkAvailable())
        return;

    listener->setAssociatedNode(nullptr); //和node无关
    listener->setFixedPriority(fixedPriority); //不能为0 0已经被占用了
    listener->setRegistered(true);
    listener->setPaused(false);

    addEventListener(listener);
}
```
上述方法统一调用了`addEventListener`方法
```c
void EventDispatcher::addEventListener(EventListener* listener)
{
    if (_inDispatch == 0)
    {
        forceAddEventListener(listener);
    }
    else
    {
        _toAddedListeners.push_back(listener);
    }

    listener->retain(); //为什么要增加一次引用计数呢
}
```
该方法很简单，如果正在处理分发，就暂时将listener保存到_toAddedListeners，后面会讲怎么处理_toAddedListeners中的listener的。
如果不在分发，就直接将listener加入到监听列队中去
```c
void EventDispatcher::forceAddEventListener(EventListener* listener)
{
    EventListenerVector* listeners = nullptr;
    EventListener::ListenerID listenerID = listener->getListenerID();
    auto itr = _listenerMap.find(listenerID);
    if (itr == _listenerMap.end())
    {

        listeners = new EventListenerVector();
        _listenerMap.insert(std::make_pair(listenerID, listeners));
    }
    else
    {
        listeners = itr->second;
    }

    listeners->push_back(listener);

    if (listener->getFixedPriority() == 0)
    { //只要vector有改动, 就需要将对应的vector标脏
        setDirty(listenerID, DirtyFlag::SCENE_GRAPH_PRIORITY);

        auto node = listener->getAssociatedNode();
        CCASSERT(node != nullptr, "Invalid scene graph priority!");
        //加入到一个map<node, vector>数据结构中去
        //vector保存的是所有该node上的事件监听类
        associateNodeAndEventListener(node, listener);

        if (node->isRunning())
        { //所有和node关联的listener调用setPaused(false),从暂停中恢复过来
            resumeEventListenersForTarget(node);
        }
    }
    else
    { //只要vector有改动, 就需要将对应的vector标脏
        setDirty(listenerID, DirtyFlag::FIXED_PRIORITY);
    }
}
```
首先根据listener的ID从map中找到属于自己类型的vector，
我们看到了`EventListenerVector`这个结构体，这个结构体维护了两个std::vector，一个是保存fixedListeners，它
的优先级不能等于0，一个是保存sceneGraphListeners，它的优先级等于0.

然后往相应的vector中插入listener。

调用`setDirty`标脏。比如说，我添加了一个scenegraph类型的touch事件监听，那么保存所有touch监听的`EventListenerVector`就变了，多了一个元素，那么我就要将这个类型的ID标脏，而且还需明确是sceneGraphListeners的数组脏了。
为什么需要标脏，这是为了将数组排序，如果数组脏了我就重新排序，否则就不用了，提高了效率。

这里的标记数据全部存在另一个map中。<u>为什么不将标记放在`EventListenerVector`里面呢？</u>

如果priority等于0，那么还需要调用`associateNodeAndEventListener`和`resumeEventListenersForTarget`。

还有一个添加自定义事件监听的方法：
```c
EventListenerCustom* EventDispatcher::addCustomEventListener(const std::string &eventName, const std::function<void(EventCustom*)>& callback)
{
    EventListenerCustom *listener = EventListenerCustom::create(eventName, callback);
    addEventListenerWithFixedPriority(listener, 1);
    return listener;
}
```
创建一个`EventListenerCustom`通过addEventListenerWithFixedPriority加入到监听列队中

除了add函数外，还提供了多种remove函数，遍历查找，并将listener移除监听列队，将改动的vector标脏。这里不再详述。
```c
/** Remove a listener
 *  @param listener The specified event listener which needs to be removed.
 */
void removeEventListener(EventListener* listener);

/** Removes all listeners with the same event listener type */
void removeEventListenersForType(EventListener::Type listenerType);

/** Removes all listeners which are associated with the specified target. */
void removeEventListenersForTarget(Node* target, bool recursive = false);

/** Removes all custom listeners with the same event name */
void removeCustomEventListeners(const std::string& customEventName);

/** Removes all listeners */
void removeAllEventListeners();
```

接下来分析事件分发
```c
void EventDispatcher::dispatchEvent(Event* event)
{
    if (!_isEnabled)
        return;
    //将_dirtyNodes的脏数据转移到_priorityDirtyFlagMap里去
    updateDirtyFlagForSceneGraph();

    DispatchGuard guard(_inDispatch); //进来时+1 出去时-1

    if (event->getType() == Event::Type::TOUCH)
    { //触摸事件处理逻辑和其他不一样，单独处理
        dispatchTouchEvent(static_cast<EventTouch*>(event));
        return;
    }
    //获取事件的ID
    auto listenerID = __getListenerID(event);
    //根据是否标脏，对相应ID下的数组进行排序
    sortEventListeners(listenerID);

    auto iter = _listenerMap.find(listenerID);
    if (iter != _listenerMap.end())
    {
        auto listeners = iter->second;

        auto onEvent = [&event](EventListener* listener) -> bool{
          //这里将listener中的node节点发送到Event事件中去了
          //所以在回调用，可以访问node节点了
            event->setCurrentTarget(listener->getAssociatedNode());
            //虽然调用的是EventListener的_onEvent回调，但实际是其子类的包装过的回调
            listener->_onEvent(event);
            //true - 一旦找到一个监听类，就调用该监听类的回调，然后不再继续遍历了
            //false - 可能不止调用一个监听类的回调函数
            return event->isStopped();
        };
        //先遍历fixedPriorityListeners中优先级<0的监听
        //再遍历sceneGraphPriorityListeners
        //再遍历fixedPriorityListeners中优先级>0的监听
        dispatchEventToListeners(listeners, onEvent);
    }
    //移除所有标记为removed的监听
    //添加所有标记为add的监听
    updateListeners(event);
}
```
关于排序函数，是理解事件优先级的关键。
```c
//对优先级不是0的监听事件进行排序，直接按照FixedPriority作为比较标准
void EventDispatcher::sortEventListenersOfFixedPriority(const EventListener::ListenerID& listenerID)
{
    auto listeners = getListeners(listenerID);

    if (listeners == nullptr)
        return;
    // 函数返回的是指针，所以不用担心改变fixedListeners会无效
    auto fixedListeners = listeners->getFixedPriorityListeners();
    if (fixedListeners == nullptr)
        return;
    //调用algorithm的sort排序
    // After sort: priority < 0, > 0
    std::sort(fixedListeners->begin(), fixedListeners->end(), [](const EventListener* l1, const EventListener* l2) {
        return l1->getFixedPriority() < l2->getFixedPriority();
    });

    //统计大于0的个数，这样才知道优先级=0的监听事件的起始位置
    int index = 0;
    for (auto& listener : *fixedListeners)
    {
        if (listener->getFixedPriority() >= 0)
            break;
        ++index;
    }

    listeners->setGt0Index(index);
//打开这个宏可以看到排序结果哦
#if DUMP_LISTENER_ITEM_PRIORITY_INFO
    log("-----------------------------------");
    for (auto& l : *fixedListeners)
    {
        log("listener priority: node (%p), fixed (%d)", l->_node, l->_fixedPriority);
    }
#endif
}
//rootNode就是当前运行的场景
void EventDispatcher::sortEventListenersOfSceneGraphPriority(const EventListener::ListenerID& listenerID, Node* rootNode)
{
    auto listeners = getListeners(listenerID);

    if (listeners == nullptr)
        return;
    auto sceneGraphListeners = listeners->getSceneGraphPriorityListeners();

    if (sceneGraphListeners == nullptr)
        return;

    // Reset priority index
    _nodePriorityIndex = 0;
    // 用来记录节点的优先级，先清空
    _nodePriorityMap.clear();
    // 从当前场景节点开始递归遍历节点
    visitTarget(rootNode, true);

    // After sort: priority < 0, > 0
    std::sort(sceneGraphListeners->begin(), sceneGraphListeners->end(), [this](const EventListener* l1, const EventListener* l2) {
        return _nodePriorityMap[l1->getAssociatedNode()] > _nodePriorityMap[l2->getAssociatedNode()];
    });

#if DUMP_LISTENER_ITEM_PRIORITY_INFO
    log("-----------------------------------");
    for (auto& l : *sceneGraphListeners)
    {
        log("listener priority: node ([%s]%p), priority (%d)", typeid(*l->_node).name(), l->_node, _nodePriorityMap[l->_node]);
    }
#endif
}
```
这里向西讲解一下visitTarget函数，这里的排序规则，决定了最后sceneGraphListeners里面监听类的顺序，越在前面的越会优先调用。

排序规则根据两个值：
- globalZOrder：默认为0，除非程序主动设置
- localZorder：默认为0，除非程序主动设置，它只能用于兄弟节点之间的比较。

节点排序时，优先考虑globalZoder，globalZoder越小越靠前，在globalZorder相同的情况下，按本身children的顺序加入即可，因为children本身就是排好序的。

Node::visit函数中会调用`sortAllChildren`，先比较localZOrder，如果localZorder相同，那么就比较加入时的顺序，先加入的在前面。绘制的时候，就是先绘制前面的，再绘制后面的，localZOrder比较小或先加入的会被
遮挡。localZOrder<0的会被父节点本身遮挡。

了解了这些，就能很好理解visitTarget中的这段代码了，因为children本身是排序好的，所以遍历的时候，前面自然都是<0，后面都是>=0的。
```c
void EventDispatcher::visitTarget(Node* node, bool isRootNode)
{
  ...
  if(childrenCount > 0)
    {
        Node* child = nullptr;
        // visit children zOrder < 0
        for( ; i < childrenCount; i++ )
        {
            child = children.at(i);

            if ( child && child->getLocalZOrder() < 0 )
                visitTarget(child, false);
            else
                break;
        }

        if (_nodeListenersMap.find(node) != _nodeListenersMap.end())
        {
            _globalZOrderNodeMap[node->getGlobalZOrder()].push_back(node);
        }

        for( ; i < childrenCount; i++ )
        {
            child = children.at(i);
            if (child)
                visitTarget(child, false);
        }
    }
    ...
}
```
上述代码其实相当于一个深度中序遍历，树状图中的最左下角优先级最低，树状图最右下角优先级最高，它也是最后被绘制的，该顺序其实和Node::visit函数中渲染的顺序是一样的。
**<u>也就是是说最后渲染的节点优先级最高。</u>**

接下来分析触摸事件的分发

关于touch事件，在分发时做了单独处理，见函数`dispatchTouchEvent`，该函数没有难点，这里就不再贴代码了。

关于自定义时间，需要主动调用函数`dispatchCustomEvent`来分发事件。
```c
void EventDispatcher::dispatchCustomEvent(const std::string &eventName, void *optionalUserData)
{
    EventCustom ev(eventName);
    ev.setUserData(optionalUserData);
    dispatchEvent(&ev);
}
```

现在有个本质问题，事件是如何发送给`_eventDispatcher`，到底在哪调用了`dispatchEvent`?

这个问题跟设备有关系，比如触摸、加速器等等，应该根据不同平台来分别设计，在`platform/ios/CCDevice.mm`中，和加速器有关
```c
- (void)accelerometer:(CMAccelerometerData *)accelerometerData
{
  ...
    cocos2d::EventAcceleration event(*_acceleration);
    auto dispatcher = cocos2d::Director::getInstance()->getEventDispatcher();
    dispatcher->dispatchEvent(&event);//哈哈 在这里调了
}
```
在`platform/ios/CCEAGLView`中，和触摸有关
```c
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    ...
    auto glview = cocos2d::Director::getInstance()->getOpenGLView();
    //调用GLViewProtocol::handleTouchesBegin
    glview->handleTouchesBegin(i, (intptr_t*)ids, xs, ys);
}

void GLViewProtocol::handleTouchesBegin(int num, intptr_t ids[], float xs[], float ys[])
{
    touchEvent._eventCode = EventTouch::EventCode::BEGAN;
    auto dispatcher = Director::getInstance()->getEventDispatcher();
    dispatcher->dispatchEvent(&touchEvent);//哈哈 又被我发现了
}
```
其他事件差不多都能找到，就不再分析了。
