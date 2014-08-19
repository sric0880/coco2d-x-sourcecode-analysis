
# cocos2d-x TransitionScene 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-11 19:06:55  -->

### 所有效果
- TransitionJumpZoom,先缩小跳出去，然后跳进来放大
- TransitionProgressRadialW,逆时针旋转
- TransitionProgressRadialCW,顺时针旋转
- TransitionProgressHorizontal,水平
- TransitionProgressVertical,进度条竖直
- TransitionProgressInOut,从中间逐渐放大
- TransitionProgressOutIn,逐渐缩小
- TransitionCrossFade,叠加渐变
- TransitionPageForward,向前翻页
- TransitionPageBackward,向后翻页
- TransitionFadeTR,向右上网格
- TransitionFadeBL,向左下网格
- TransitionFadeUp,向上条形
- TransitionFadeDown,向下条形
- TransitionTurnOffTiles,随机网格
- TransitionSplitRows,切成3个横条，左右移出
- TransitionSplitCols,切成3个竖条，上下移出
- TransitionFade,由黑渐变，颜色参数
- FlipXLeftOver,由左向右翻转
- FlipXRightOver,由右向左翻转
- FlipYUpOver,由上而下翻转
- FlipYDownOver,由下而上翻转
- FlipAngularLeftOver,由左向右翻跟斗
- FlipAngularRightOver,由右向左翻跟斗
- ZoomFlipXLeftOver,同上，带透视效果
- ZoomFlipXRightOver,
- ZoomFlipYUpOver,
- ZoomFlipYDownOver,
- ZoomFlipAngularLeftOver,
- ZoomFlipAngularRightOver,
- TransitionShrinkGrow,从一旁变小，从另一旁变大
- TransitionRotoZoom,旋转退出，旋转进入
- TransitionMoveInL,从左进入
- TransitionMoveInR,从右进入
- TransitionMoveInT,从上进入
- TransitionMoveInB,从下进入
- TransitionSlideInL,从左往右推
- TransitionSlideInR,从右往左推
- TransitionSlideInT,从上往下推
- TransitionSlideInB,从下往上推

###源码分析

####TransitionScene Class

``` c++
class _DLL TransitionScene : public Scene
{
public:
    /** Orientation Type used by some transitions
     */
    enum class Orientation
    {
        /// An horizontal orientation where the Left is nearer
        LEFT_OVER = 0,
        /// An horizontal orientation where the Right is nearer
        RIGHT_OVER = 1,
        /// A vertical orientation where the Up is nearer
        UP_OVER = 0,
        /// A vertical orientation where the Bottom is nearer
        DOWN_OVER = 1,
    };

    /** creates a base transition with duration and incoming scene */
    static TransitionScene * create(float t, Scene *scene);

    /** called after the transition finishes */
    void finish(void);

    /** used by some transitions to hide the outer scene */
    void hideOutShowIn(void);

    //
    // Overrides
    //
    virtual void draw(Renderer *renderer, const Mat4 &transform, uint32_t flags) override;
    virtual void onEnter() override;
    virtual void onExit() override;
    virtual void cleanup() override;

_CONSTRUCTOR_AESS:
    TransitionScene();
    virtual ~TransitionScene();

    /** initializes a transition with duration and incoming scene */
    bool initWithDuration(float t,Scene* scene);

protected:
    virtual void sceneOrder();
    void setNewScene(float dt);

    Scene *_inScene; //包含两个场景，做切换
    Scene *_outScene;
    float _duration;
    bool _isInSceneOnTop;
    bool _isSendCleanupToScene;

private:
    _DISALLOW_COPY_AND_ASSIGN(TransitionScene);
};
```

####TransitionScene Initialization
```c++
bool TransitionScene::initWithDuration(float t, Scene *scene)
{
    ASSERT( scene != nullptr, "Argument scene must be non-nil");

    if (Scene::init())
    {
        _duration = t;

        // retain
        _inScene = scene;
        _inScene->retain();
        _outScene = Director::getInstance()->getRunningScene();
        if (_outScene == nullptr)
        {
            _outScene = Scene::create();
        }
        _outScene->retain();

        ASSERT( _inScene != _outScene, "Incoming scene must be different from the outgoing scene" );

        sceneOrder(); //_isInSceneOnTop = true;

        return true;
    }
    else
    {
        return false;
    }
}
```

####draw 调用_inScene和_outScene分别做渲染
```c++
void TransitionScene::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    Scene::draw(renderer, transform, flags);

    if( _isInSceneOnTop ) {
        _outScene->visit(renderer, transform, flags);
        _inScene->visit(renderer, transform, flags);
    } else {
        _inScene->visit(renderer, transform, flags);
        _outScene->visit(renderer, transform, flags);
    }
}

```
####跳转后必须调用finish
- 注意：不能直接直接调用TransitionScene::create方法，只能调用它的子类方法，因为TransitionScene没有调用finish。
- TODO: 引擎应该去掉TransitionScene::create方法

```c++
void TransitionScene::finish()
{
    // clean up
    _inScene->setVisible(true);
    _inScene->setPosition(Vec2(0,0));
    _inScene->setScale(1.0f);
    _inScene->setRotation(0.0f);
    _inScene->setAdditionalTransform(nullptr);

    _outScene->setVisible(false);
    _outScene->setPosition(Vec2(0,0));
    _outScene->setScale(1.0f);
    _outScene->setRotation(0.0f);
    _outScene->setAdditionalTransform(nullptr);

    //[self schedule:@selector(setNewScene:) interval:0];
    this->schedule(schedule_selector(TransitionScene::setNewScene), 0); //为什么要下一帧调用呢？？
}

void TransitionScene::setNewScene(float dt)
{
    _UNUSED_PARAM(dt);

    this->unschedule(schedule_selector(TransitionScene::setNewScene));

    // Before replacing, save the "send cleanup to scene"
    Director *director = Director::getInstance();
    _isSendCleanupToScene = director->isSendCleanupToScene();

    director->replaceScene(_inScene); //这里将过场场景交给最终的场景

    // issue #267
    _outScene->setVisible(true);
}
```
####基本流程为：init->onEnter->draw...->finish->onExit->cleanup

所有场景切换的类都是继承TransitionScene，重载onEnter函数实现动画切换的效果。
其本质是利用Action实现动画效果。
例如：
```c++
void TransitionRotoZoom:: onEnter()
{
    TransitionScene::onEnter();

    _inScene->setScale(0.001f);
    _outScene->setScale(1.0f);

    _inScene->setAnchorPoint(Vec2(0.5f, 0.5f));
    _outScene->setAnchorPoint(Vec2(0.5f, 0.5f));

    ActionInterval *rotozoom = (ActionInterval*)(Sequence::create
    (
        Spawn::create
        (
            ScaleBy::create(_duration/2, 0.001f),
            RotateBy::create(_duration/2, 360 * 2),
            nullptr
        ),
        DelayTime::create(_duration/2),
        nullptr
    ));

    _outScene->runAction(rotozoom);
    _inScene->runAction
    (
        Sequence::create
        (
            rotozoom->reverse(),
            //动画结束后调用finish函数
            CallFunc::create(_CALLBACK_0(TransitionScene::finish,this)),
            nullptr
        )
    );
}

void TransitionJumpZoom::onEnter()
{
    TransitionScene::onEnter();
    Size s = Director::getInstance()->getWinSize();

    _inScene->setScale(0.5f);
    _inScene->setPosition(Vec2(s.width, 0));
    _inScene->setAnchorPoint(Vec2(0.5f, 0.5f));
    _outScene->setAnchorPoint(Vec2(0.5f, 0.5f));

    ActionInterval *jump = JumpBy::create(_duration/4, Vec2(-s.width,0), s.width/4, 2);
    ActionInterval *scaleIn = ScaleTo::create(_duration/4, 1.0f);
    ActionInterval *scaleOut = ScaleTo::create(_duration/4, 0.5f);

    ActionInterval *jumpZoomOut = (ActionInterval*)(Sequence::create(scaleOut, jump, nullptr));
    ActionInterval *jumpZoomIn = (ActionInterval*)(Sequence::create(jump, scaleIn, nullptr));

    ActionInterval *delay = DelayTime::create(_duration/2);

    _outScene->runAction(jumpZoomOut);
    _inScene->runAction
    (
        Sequence::create
        (
            delay,
            jumpZoomIn,
            //在这里调用finish方法
            CallFunc::create(_CALLBACK_0(TransitionScene::finish,this)),
            nullptr
        )
    );
}
```
大家可能不明白TransitionScene的hideOutShowIn方法有何用，在TransitionFade中使用：
```c++
void TransitionFade :: onEnter()
{
    TransitionScene::onEnter();
    //添加一个颜色蒙版
    LayerColor* l = LayerColor::create(_color);
    _inScene->setVisible(false);

    addChild(l, 2, kSceneFade);
    Node* f = getChildByTag(kSceneFade);

    ActionInterval* a = (ActionInterval *)Sequence::create
        (
            FadeIn::create(_duration/2),
            //交换场景
            CallFunc::create(CC_CALLBACK_0(TransitionScene::hideOutShowIn,this)),
            FadeOut::create(_duration/2),
            CallFunc::create(CC_CALLBACK_0(TransitionScene::finish,this)),

         nullptr
        );
    f->runAction(a);
}
```

####其他效果
- TransitionCrossFade用到了`RenderTexture`和`BlendFunc`
- TransitionTurnOffTiles使用了网格动画`TurnOffTiles`
- TransitionSplitCols使用了网格动画`SplitCols`
- TransitionSplitRows使用了网格动画`SplitRows`
- TransitionFadeTR、TransitionFadeBL、TransitionFadeUp、TransitionFadeDown使用了网格动画`FadeOutTRTiles`
- TransitionProgress系列单独存在于文件TransitionProgress.h中，它也是继承于TransitionScene，
原理：
  1. 将_outScene绘制到RenderTexture
  2. 用RenderTexture中的Sprite生成ProgressTimer
  3. 利用ProgressFromTo动画来实现ProgressTimer的效果

```c++
class CC_DLL TransitionProgress : public TransitionScene
{
public:
    static TransitionProgress* create(float t, Scene* scene);

    //
    // Overrides
    //
    virtual void onEnter() override;
    virtual void onExit() override;

CC_CONSTRUCTOR_ACCESS:
    TransitionProgress();
    virtual ~TransitionProgress(){}

protected:
    virtual void sceneOrder() override;

protected:
    virtual ProgressTimer* progressTimerNodeWithRenderTexture(RenderTexture* texture);
    virtual void setupTransition();

protected:
    float _to;
    float _from;
    Scene* _sceneToBeModified;
};
```
- TransitionPageTurn实现翻页效果，单独在文件TransitionPageTurn.h中实现，
原理是声明了两个NodeGrid，重写draw函数，利用PageTurn3D网格动画播放翻页动画

####其他
> 关于NodeGrid、网格动画见`cocos2d-x 网格动画源码解析`  
关于Action动画见`cocos2d-x Action动画源码解析`  
关于ProgressTimer的设置和使用见`cocos2d-x 进度条源码解析`
