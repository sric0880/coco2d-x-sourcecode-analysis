# cocos2d-x Director类详解

**version cocos2d-x 3.2**
<!-- create time: 2014-08-12 20:15  -->
##1. Main Loop
Director有4个纯虚方法，说明Director本身不能被实例化，DisplayLinkDirector继承自Director，
下面是这4个纯虚方法：
```c
virtual void mainLoop() override;
virtual void setAnimationInterval(double value) override;
virtual void startAnimation() override;
virtual void stopAnimation() override;
```
Director::getInstance获得的是DisplayLinkDirector实例：
```c
Director* Director::getInstance()
{
    if (!s_SharedDirector)
    {
        s_SharedDirector = new (std::nothrow) DisplayLinkDirector();
        CCASSERT(s_SharedDirector, "FATAL: Not enough memory");
        s_SharedDirector->init();
    }

    return s_SharedDirector;
}
```
我们来看看DisplayLinkDirector
```c
void DisplayLinkDirector::startAnimation()
{
    if (gettimeofday(_lastUpdate, nullptr) != 0)
    {
        CCLOG("cocos2d: DisplayLinkDirector: Error on gettimeofday");
    }

    _invalid = false;
    //重新设置帧率
    Application::getInstance()->setAnimationInterval(_animationInterval);

    // fix issue #3509, skip one fps to avoid incorrect time calculation.
    setNextDeltaTimeZero(true);
}

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

void DisplayLinkDirector::stopAnimation()
{
    _invalid = true;
}

void DisplayLinkDirector::setAnimationInterval(double interval)
{
    _animationInterval = interval;
    if (! _invalid)
    {
        stopAnimation();
        startAnimation();
    }
}
```
主循环中有最关键的两个函数drawScene和内存池清理，既然我们知道了所有事件的源头来自
mainLoop()函数，那么什么地方调用了它呢？
这里不同的平台调用的方法不一样，以iOS为例，在AppController.mm中有一行`cocos2d::Application::getInstance()->run();`
，进入到这个函数
```objc
int Application::run()
{
    if (applicationDidFinishLaunching())
    {
        [[CCDirectorCaller sharedDirectorCaller] startMainLoop];
    }
    return 0;
}
```
进入`[[CCDirectorCaller sharedDirectorCaller] startMainLoop];`

```objc
-(void) startMainLoop
{
        // Director::setAnimationInterval() is called, we should invalidate it first
    [self stopMainLoop];

    displayLink = [NSClassFromString(@"CADisplayLink") displayLinkWithTarget:self selector:@selector(doCaller:)];
    [displayLink setFrameInterval: self.interval];
    [displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
}
-(void) doCaller: (id) sender
{
    cocos2d::Director* director = cocos2d::Director::getInstance();
    [EAGLContext setCurrentContext: [(CCEAGLView*)director->getOpenGLView()->getEAGLView() context]];
    director->mainLoop(); //就是这里，调用了之前的mainLoop函数
}
```
displayLink设置好回调函数（doCaller）和帧率，通过addToRunLoop加入到RunLoop中去。
每一帧都会回调doCaller方法，而doCaller方法调用了mainLoop方法。

关于内存管理与回收机制，参见
> `cocos2d-x 内存池源码分析`

现在我们来分析最重要的函数之一drawScene
```c
void Director::drawScene()
{
    // calculate "global" dt
    //更新_deltaTime（上次调用该函数到这次调用的间隔时间）
    //更新_lastUpdate（记录上次调用的时间点）
    calculateDeltaTime();

    // skip one flame when _deltaTime equal to zero.
    if(_deltaTime < FLT_EPSILON)
    {
        return;
    }

    if (_openGLView)
    {
        _openGLView->pollInputEvents(); //TODO: 还不知道有啥用
    }

    //tick before glClear: issue #533
    if (! _paused)
    {
        _scheduler->update(_deltaTime); //调度系统就在这里驱动的
        _eventDispatcher->dispatchEvent(_eventAfterUpdate); //TODO:update结束了我要发送一个消息
    }
    //\(^o^)/ 开始绘制了
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); //开启颜色和深度缓冲区

    /* to avoid flickr, nextScene MUST be here: after tick and before draw.
     XXX: Which bug is this one. It seems that it can't be reproduced with v0.9 */
    if (_nextScene)
    {
        setNextScene(); //和Scene Manager相关，见下文
    }
    //将栈顶的矩阵复制并压栈
    pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    // draw the scene
    if (_runningScene)
    { //visit方法会递归调用所有children的visit方法，
        //visit方法内会调用draw方法，(3.0之前是直接进行绘制)，将draw函数中的
        //绘制command发送给renderer，由renderer统一绘制
        _runningScene->visit(_renderer, Mat4::IDENTITY, false);
        _eventDispatcher->dispatchEvent(_eventAfterVisit);//TODO: visit结束了我要发一个消息
    }

    // 绘制悬浮节点，如果叫global Node比较好理解，不随场景的切换而消失
    if (_notificationNode)
    {
        _notificationNode->visit(_renderer, Mat4::IDENTITY, false);
    }

    if (_displayStats)
    {
        showStats(); //屏幕上显示帧率
    }
    //开始渲染
    _renderer->render();
    _eventDispatcher->dispatchEvent(_eventAfterDraw);//TODO: 渲染结束了我要发一个消息
    //出栈
    popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    _totalFrames++; //计算从程序开始到现在共刷过多少帧

    //绘制：swap buffers
    if (_openGLView)
    {
        _openGLView->swapBuffers();
    }

    if (_displayStats)
    {
        calculateMPF(); //计算_secondsPerFrame
    }
}
```
关于schedule调度，参见
> `cocos2d-x scheduler调度源码分析`

关于visit和render绘制函数的分析，参见
> `cocos2d-x 渲染详解`

接下来我们来看Director管理着哪些对象：
```c++
Scheduler *_scheduler; //调度类
ActionManager *_actionManager; //动画管理类
EventDispatcher* _eventDispatcher; //事件分发器
//自定义事件
EventCustom *_eventProjectionChanged, *_eventAfterDraw, *_eventAfterVisit, *_eventAfterUpdate;
GLView *_openGLView; //openGL的包装类
TextureCache *_textureCache; //纹理缓存
Vector<Scene*> _scenesStack; //场景管理
Node *_notificationNode; //悬浮节点，用于全局显示
Renderer *_renderer; //渲染类
Console *_console; //用于远程调试，不做分析
```
关于事件分发管理，参见
> `cocos2d-x EvenDispatcher 源码分析`

##2. ActionManager
见`cocos2d-x Action动画源码解析`

##3. Scene Management
和场景相关属性：
```c
Scene *_runningScene;
/* will be the next 'runningScene' in the next frame
  nextScene is a weak reference. */
Scene *_nextScene;
/* If true, then "old" scene will receive the cleanup message */
bool _sendCleanupToScene;
/* scheduled scenes */
Vector<Scene*> _scenesStack;
```
```c
//第一次进入场景时调用(runningScene必须为nullptr)，
//pushScene + startAnimation
void runWithScene(Scene *scene);
//runningScene不能为nullptr
void Director::pushScene(Scene *scene)
{
    CCASSERT(scene, "the scene should not null");
    //当前的Scene不需要清理
    _sendCleanupToScene = false;

    _scenesStack.pushBack(scene);
    _nextScene = scene;
}

void Director::popScene(void)
{
  //ONLY call it if there is a running scene.
    CCASSERT(_runningScene != nullptr, "running scene should not null");

    _scenesStack.popBack(); //Pops out a scene from the stack.
    ssize_t c = _scenesStack.size();
    //If there are no more scenes in the stack the execution is terminated.
    if (c == 0)
    {
        end();
    }
    else
    {
        _sendCleanupToScene = true; //The running scene will be deleted
        _nextScene = _scenesStack.at(c - 1);
    }
}
/** Pops out all scenes from the stack until the root scene in the queue.
 * This scene will replace the running one.
 * Internally it will call `popToSceneStackLevel(1)`
 */
void popToRootScene();
/** Pops out all scenes from the stack until it reaches `level`.
 If level is 0, it will end the director.
 If level is 1, it will pop all scenes until it reaches to root scene.
 If level is >= than the current stack level, it won't do anything.
 */
void popToSceneStackLevel(int level);
/** Replaces the running scene with a new one. The running scene is terminated.
 * ONLY call it if there is a running scene.
 */
void replaceScene(Scene *scene);
{
    CCASSERT(_runningScene, "Use runWithScene: instead to start the director");
    CCASSERT(scene != nullptr, "the scene should not be null");

    if (scene == _nextScene)
        return;

    if (_nextScene)
    {
        if (_nextScene->isRunning())
        {
            _nextScene->onExit();
        }
        _nextScene->cleanup();
        _nextScene = nullptr;
    }

    ssize_t index = _scenesStack.size();

    _sendCleanupToScene = true;
    _scenesStack.replace(index - 1, scene);

    _nextScene = scene;
}
/** Ends the execution, releases the running scene.
 It doesn't remove the OpenGL view from its parent. You have to do it manually.
* @lua endToLua
*/
void end()
{
    _purgeDirectorInNextLoop = true;
}
/** Draw the scene.
This method is called every frame. Don't call it manually.
*/
void Director::drawScene()
{
  ...
  if (_nextScene)
  {
      //release _runningScene and then _runningScene = _nextScene;
      setNextScene();
  }
  ...
}
```
##4. OpenGL相关
OpenGL负责初始化OpenGL上下文，设置像素格式（默认是RGB8888），设置
深度缓冲区为0-bit，设置投影为3D，设置orientation为竖屏

OpenGL开启了GL_TEXTURE_2D, GL_VERTEX_ARRAY, GL_COLOR_ARRAY,
GL_TEXTURE_COORD_ARRAY.

CCDirector.h定义了3种变换矩阵：

```c++
enum class MATRIX_STACK_TYPE
{
    MATRIX_STACK_MODELVIEW,
    MATRIX_STACK_PROJECTION,
    MATRIX_STACK_TEXTURE
};
```

每一个投影矩阵都有一个std::stack:
```c++
std::stack<Mat4> _modelViewMatrixStack;
std::stack<Mat4> _projectionMatrixStack;
std::stack<Mat4> _textureMatrixStack;
```
Director还定义了如下方法：
```c++
protected:
    //在Director::init()中调用，3个stack分别pop
    //再分别push一个单位矩阵
    void initMatrixStack();
public:
    void pushMatrix(MATRIX_STACK_TYPE type); //pop
    void popMatrix(MATRIX_STACK_TYPE type); //push(top)
    void loadIdentityMatrix(MATRIX_STACK_TYPE type); //top = identity
    void loadMatrix(MATRIX_STACK_TYPE type, const Mat4& mat); //top = matrix
    void multiplyMatrix(MATRIX_STACK_TYPE type, const Mat4& mat); //top*=matrix
    Mat4 getMatrix(MATRIX_STACK_TYPE type); //return top
    void resetMatrixStack(); //调用initMatrixStack()
```

接下来我们来看看程序是在什么地方设置这些OpenGL参数的：
在AppDelegate.cpp中的`bool AppDelegate::applicationDidFinishLaunching()`可以看到：
```c
// initialize director
    auto director = Director::getInstance();
    auto glview = director->getOpenGLView();
    if(!glview) {
        glview = GLView::create("Cpp Tests");
        director->setOpenGLView(glview); //设置OpenGL相关参数
    }
```

再来看看`setOpenGLView`函数：
```c
void Director::setOpenGLView(GLView *openGLView)
{
    CCASSERT(openGLView, "opengl view should not be null");

    if (_openGLView != openGLView)
    {
        // Configuration. Gather GPU info
        Configuration *conf = Configuration::getInstance();
        conf->gatherGPUInfo();
        CCLOG("%s\n",conf->getInfo().c_str());//每次程序debug时打印的那一堆东西

        if(_openGLView)
            _openGLView->release();
        _openGLView = openGLView;
        _openGLView->retain();

        // 获得winsize 就是画布大小
        _winSizeInPoints = _openGLView->getDesignResolutionSize();

        createStatsLabel(); //初始化左下角fps字体和Label

        if (_openGLView)
        {
            //设置包括：setAlphaBlending(true);
            //  true - GL::blendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA);
            //  false - GL::blendFunc(GL_ONE, GL_ZERO);

            //setDepthTest(false); 关闭深度测试
            //setProjection(_projection); //设置投影 _projection在Director::init()调用Configuration获得
            //glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
            setGLDefaultValues();
        }

        //渲染相关的初始化 主要有两个方法
        //1. setupIndices(); 设置顶点索引，例如：
        //0--1  4--5--
        //|  |  |  |
        //2--3  6--7--
        //索引为：0，1，2，3，2，1，4，5，6，7，6，5，...
        //说明游戏中都是绘制四边形，以三角形进行绘制

        //2. setupBuffer()
        //如果支持VAO，就使用VAO，否则只使用VBO。此时并不知道需要绘制的顶点数量，
        //所以程序使用VBO_SIZE来统一顶点个数
        //创建两个VBO，1 for vertex, another for indices
        //一个顶点包括position(3 floats)、color(4 bytes)、textureCoord(2 floats)
        _renderer->initGLView();

        CHECK_GL_ERROR_DEBUG();

        if (_eventDispatcher)
        {
            _eventDispatcher->setEnabled(true);
        }
    }
}
```
看一下上述`setProjection(_projection)`的定义
```c
void Director::setProjection(Projection projection)
{
  Size size = _winSizeInPoints;
  setViewport(); //设置窗口大小为winsize
  switch (projection)
    {
        case Projection::_2D:
        {
            loadIdentityMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);
#if CC_TARGET_PLATFORM == CC_PLATFORM_WP8
            if(getOpenGLView() != nullptr)
            {
                multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, getOpenGLView()->getOrientationMatrix());
            }
#endif
            Mat4 orthoMatrix;
            //设置平行投影矩阵
            Mat4::createOrthographicOffCenter(0, size.width, 0, size.height, -1024, 1024, &orthoMatrix);
            multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, orthoMatrix);
            loadIdentityMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
            break;
        }

        case Projection::_3D:
        {
          //(_winSizeInPoints.height / 1.1566f);
          //眼睛的位置在设备屏幕上方，这个值可是有讲究的哦
            float zeye = this->getZEye();

            Mat4 matrixPerspective, matrixLookup;

            //将projection矩阵设置为单位矩阵
            loadIdentityMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);

#if CC_TARGET_PLATFORM == CC_PLATFORM_WP8
            //if needed, we need to add a rotation for Landscape orientations on Windows Phone 8 since it is always in Portrait Mode
            GLView* view = getOpenGLView();
            if(getOpenGLView() != nullptr)
            {
                multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, getOpenGLView()->getOrientationMatrix());
            }
#endif
            //创建透视投影矩阵，参数依次为：上下视角为60度，长宽比，近平面，远平面
            Mat4::createPerspective(60, (GLfloat)size.width/size.height, 10, zeye+size.height/2, &matrixPerspective);

            multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, matrixPerspective);
            //eye表示视点的位置，center表示目标点，视线方向由eye->center决定
            //up表示头顶方向，方向为正y轴
            Vec3 eye(size.width/2, size.height/2, zeye), center(size.width/2, size.height/2, 0.0f), up(0.0f, 1.0f, 0.0f);
            Mat4::createLookAt(eye, center, up, &matrixLookup);
            //妈比的，这个matrixLookup应该是视图变换矩阵啊，应该乘在MATRIX_STACK_MODELVIEW上啊！！！
            multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, matrixLookup);

            loadIdentityMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
            break;
        }

        case Projection::CUSTOM:
            // Projection Delegate is no longer needed
            // since the event "PROJECTION CHANGED" is emitted
            break;

        default:
            CCLOG("cocos2d: Director: unrecognized projection");
            break;
    }

    _projection = projection;
    GL::setProjectionMatrixDirty();

    _eventDispatcher->dispatchEvent(_eventProjectionChanged);

}
```

##5. Coordinate Management
相关概念：  
**FrameSize**: 设备分辨率  

**WinSizeInPoint**: 就是design resolution size, 可以理解为OpenGL的画布大小（z值为0时的viewport大小），但是实际上的viewPort大小需要根据实际设备分辨
率进行缩放。有几种缩放策略可以选择。  

**WinSizeInPixel**: 将WinSizeInPoint长宽分别乘上_contentScaleFactor，（_contentScaleFactor和viewPort的缩放没有关系，是用来控制design resolution size
和资源图片的缩放比例的）比如，我的图片资源是960*640，一般我会设置design resolution size也为960*640，然后通过viewPort去适配不同的设备分辨率。
这个时候的_contentScaleFactor等于1，默认也是1，所以我们不需要设置，但是假设我设置了design resolution size为480*320(不太现实)，那么我就必须
通过设置_contentScaleFactor等于0.5来缩小图片资源才能够在480320的画布上完全显示。  
我们什么时候会用到_contentScaleFactor，如果我们现在有两套图片资源，一套480320，一套960640，当然根据设备分辨率分别设置两套design resolution size
是可行的，但是我们在不同大小的设计分辨率上我们需要不同的坐标，就是说我们需要两套坐标，这是不太现实的。所以采用一套设计分辨路，假设我们采用设计分辨率为
960640，那么对于大图资源，正好；对于小图资源，我们需要设置_contentScaleFactor等于2，绘制的时候，程序会先将图放大，然后绘制到960640大小的画布上，然后
根据viewPort大小又缩放到480320的设备分辨率上，这样做，省去了480320设计分辨路下的坐标。太科学了。

**VisibleSize**: 设备上能够显示出来的画布(win size)的长宽  
**VisibleOrigin**:  设备上能够显示出来的画布(win size)的原点

将设备上的坐标点转换为OpenGL中的坐标点
```c
Vec2 convertToGL(const Vec2& point);
```

将OpenGL中的坐标点转换为设备上的坐标点
```c
Vec2 convertToUI(const Vec2& point);
```
