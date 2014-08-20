# cocos2d-x 内存管理 源码分析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-20 22:29  -->

整个内存管理的动力在`PoolManager::getInstance()->getCurrentPool()->clear()`
```c
void DisplayLinkDirector::mainLoop()
{
    if (_purgeDirectorInNextLoop) //如果调用了end()方法
    {
        _purgeDirectorInNextLoop = false;
        purgeDirector();
    }
    else if (! _invalid)//如果没有stopAnimation
    {
        drawScene(); //OpenGL 主循环 绘制场景

        // release the objects 每一次循环清理一次内存池
        PoolManager::getInstance()->getCurrentPool()->clear();
    }
}
```
##Ref
受内存管理的类都继承于`Ref`，该类维持一个引用计数，有4个函数：
```c
//增加引用计数
void retain();
//减少引用计数, 如果变成0，就delete this
void release();
//将自身加入到内存池中，执行一次clear后，调用release()
Ref* autorelease();
//获得引用计数的值
unsigned int getReferenceCount() const;
```
`Ref`初始化默认的引用计数是1

##PoolManager
主要的两个方法：
```c
AutoreleasePool *getCurrentPool() const;
//判断AutoreleasePool是否有obj
bool isObjectInPools(Ref* obj) const;
```
以及它的成员变量：
```c
//程序一般只使用一个AutoreleasePool，你也可以自己创建AutoreleasePool
//就可以保存在这里
//要是程序只使用默认的AutoreleasePool，这个类感觉多余啊
std::vector<AutoreleasePool*> _releasePoolStack;
```
##AutoreleasePool
```c
class CC_DLL AutoreleasePool
{
public:
    AutoreleasePool();

    //name说是便于调试，程序写到要调试AutoreleasePool的地步，佩服！
    AutoreleasePool(const std::string &name);

    ~AutoreleasePool();

    //添加一个Ref，多个相同的对象可能被加入，每一个相同的对象会调用一次Release，这是错误的操作
    void addObject(Ref *object);

    //每次主循环都会调用的函数，对于释放池中的对象，都会调用Release
    void clear();

#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    /**
     * Whether the pool is doing `clear` operation.
     */
    bool isClearing() const { return _isClearing; };
#endif

    /**
     * Checks whether the pool contains the specified object.
     */
    bool contains(Ref* object) const;

    /**
     * 用于测试，输出内存池中的数据
     * The result will look like:
     * Object pointer address     object id     reference count
     *
     */
    void dump();

private:
    //这就是那个保存Ref的池子
    std::vector<Ref*> _managedObjectArray;
    std::string _name;

#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    /**
     *  The flag for checking whether the pool is doing `clear` operation.
     */
    bool _isClearing;
#endif
};
```
我都觉得已经没啥好介绍的了
