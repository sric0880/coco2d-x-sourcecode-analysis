# cocos2d-x 网格动画 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-12 18:58  -->

要弄清楚GridAction，先得弄清楚NodeGrid以及Grid  
##NodeGrid
继承关系：NodeGrid --> Node
```c
class NodeGrid : public Node
{
public:
    static NodeGrid* create();

    GridBase* getGrid() { return _nodeGrid; }
    /**
    * @js NA
    */
    const GridBase* getGrid() const { return _nodeGrid; }

    /**
     * Changes a grid object that is used when applying effects
     *  只能用该函数设置GridBase
     * @param grid  A Grid object that is used when applying effects
     */
    void setGrid(GridBase *grid);

    void setTarget(Node *target);

    // overrides
    virtual void visit(Renderer *renderer, const Mat4 &parentTransform, uint32_t parentFlags) override;

protected:
    NodeGrid();
    virtual ~NodeGrid();

    void onGridBeginDraw();
    void onGridEndDraw();

    Node* _gridTarget; //需要绘制的节点，在visit中调用了该节点的visit函数
    GridBase* _nodeGrid; //网格数据结构
    GroupCommand _groupCommand; //抱团绘图命令
    CustomCommand _gridBeginCommand; //开始绘图命令
    CustomCommand _gridEndCommand; //结束绘图命令

private:
    CC_DISALLOW_COPY_AND_ASSIGN(NodeGrid);
}

//实现
void NodeGrid::onGridBeginDraw()
{
    if (_nodeGrid && _nodeGrid->isActive())
    {
        _nodeGrid->beforeDraw(); //调用GridBase的beforeDraw方法
    }
}
void NodeGrid::onGridEndDraw()
{
    if(_nodeGrid && _nodeGrid->isActive())
    {
        _nodeGrid->afterDraw(this); //调用GridBase的afterDraw方法
    }
}
//重写了Node::visit方法
void NodeGrid::visit(Renderer *renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    // quick return if not visible. children won't be drawn.
    if (!_visible)
    {
        return;
    }
    //关于GroupCommand在渲染机制详解中有说明
    _groupCommand.init(_globalZOrder);
    renderer->addCommand(&_groupCommand);
    //使用group Command 必须先pushGroup
    renderer->pushGroup(_groupCommand.getRenderQueueID());

    bool dirty = (parentFlags & FLAGS_TRANSFORM_DIRTY) || _transformUpdated;
    if(dirty)
        _modelViewTransform = this->transform(parentTransform);
    _transformUpdated = false;

    // IMPORTANT:
    // To ease the migration to v3.0, we still support the Mat4 stack,
    // but it is deprecated and your code should not rely on it
    Director* director = Director::getInstance();
    CCASSERT(nullptr != director, "Director is null when seting matrix stack");

    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);

    Director::Projection beforeProjectionType = Director::Projection::DEFAULT;
    if(_nodeGrid && _nodeGrid->isActive())
    {
        beforeProjectionType = Director::getInstance()->getProjection();
        _nodeGrid->set2DProjection();//改变了Director的投影为2D投影
    }

    _gridBeginCommand.init(_globalZOrder);
    _gridBeginCommand.func = CC_CALLBACK_0(NodeGrid::onGridBeginDraw, this);
    renderer->addCommand(&_gridBeginCommand); //切成2D投影，开启FBO

  //以下所有绘制将被绘制到_nodeGrid的_texture中去，而不是屏幕上
  //因为_nodeGrid->beforeDraw();启用了一个FBO，并绑定到了它的一个Texture上去了
    //绘制目标节点
    if(_gridTarget)
    {
        _gridTarget->visit(renderer, _modelViewTransform, dirty);
    }

    int i = 0;
    //继续绘制它的子节点
    //XXX:我就不明白了为什么不将_gridTarget作为它的子节点
    if(!_children.empty())
    {
        sortAllChildren();
        // draw children zOrder < 0
        for( ; i < _children.size(); i++ )
        {
            auto node = _children.at(i);

            if ( node && node->getLocalZOrder() < 0 )
                node->visit(renderer, _modelViewTransform, dirty);
            else
                break;
        }
        // self draw,currently we have nothing to draw on NodeGrid, so there is no need to add render command
        this->draw(renderer, _modelViewTransform, dirty);

        for(auto it=_children.cbegin()+i; it != _children.cend(); ++it) {
            (*it)->visit(renderer, _modelViewTransform, dirty);
        }
    }
    else
    {
        this->draw(renderer, _modelViewTransform, dirty);
    }

    // reset for next frame
    _orderOfArrival = 0;

    if(_nodeGrid && _nodeGrid->isActive())
    {
        // 把投影矩阵改回来
        director->setProjection(beforeProjectionType);
    }

    _gridEndCommand.init(_globalZOrder);
    _gridEndCommand.func = CC_CALLBACK_0(NodeGrid::onGridEndDraw, this);
    renderer->addCommand(&_gridEndCommand); //这里执行了网格的绘制

    renderer->popGroup(); //使用group Command必须popGroup

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
}
```
##Grid数据结构
相关类以及继承关系：  
GridBase --> Ref  
Grid3D --> GridBase 网格（点阵）  
TiledGrid3D --> GridBase 瓦片（块阵）

###GridBase Class解析
```c
/** Base class for other
*/
class CC_DLL GridBase : public Ref
{
public:
    static GridBase* create(const Size& gridSize, Texture2D *texture, bool flipped);
    static GridBase* create(const Size& gridSize);
    virtual ~GridBase(void);
    //初始化所有成员变量，最后调用calculateVertexPoints
    bool initWithSize(const Size& gridSize, Texture2D *texture, bool flipped);
    //根据winsize创建一张白色的纹理，再调用上述方法
    bool initWithSize(const Size& gridSize);

    /** whether or not the grid is active */
    inline bool isActive(void) const { return _active; }
    void setActive(bool active);

    /** number of times that the grid will be reused */
    inline int getReuseGrid(void) const { return _reuseGrid; }
    inline void setReuseGrid(int reuseGrid) { _reuseGrid = reuseGrid; }

    /** size of the grid */
    inline const Size& getGridSize(void) const { return _gridSize; }
    inline void setGridSize(const Size& gridSize) { _gridSize = gridSize; }

    /** pixels between the grids */
    inline const Vec2& getStep(void) const { return _step; }
    inline void setStep(const Vec2& step) { _step = step; }

    /** is texture flipped */
    inline bool isTextureFlipped(void) const { return _isTextureFlipped; }
    void setTextureFlipped(bool flipped);

  //开启2D投影，开启FBO
    void beforeDraw(void);
    //关闭FBO，还原成3D投影，绑定_texture纹理，调用blit
    void afterDraw(Node *target);

    /**子类需要实现的方法*/
    virtual void blit(void);
    virtual void reuse(void);
    virtual void calculateVertexPoints(void);

    void set2DProjection(void);

protected:
    bool _active;
    int  _reuseGrid; //重用次数
    Size _gridSize; //mxn的格子
    Texture2D *_texture; //FBO使用的attachment
    Vec2 _step; //每个格子的像素长和宽

    //创建了一个FBO，将_texture绑定到FBO的color attachment
    //beforeDraw，开启FBO，渲染将会在_texture上
    //afterDraw，关闭FBO
    Grabber *_grabber;
    bool _isTextureFlipped;
    GLProgram* _shaderProgram;
    Director::Projection _directorProjection;
};
```
GridBase并不能直接使用，因为它有三个虚函数，并没有实现，需要它的子类实现。

###Grid3D
Grid3D继承于GridBase，实现了三个虚函数，它是一个点阵

它需要如下成员变量：
```c
//假设是一个2*2网格，那么就需要(2+1)*(2+1)=9个顶点
GLvoid *_texCoordinates; //顶点的纹理坐标
GLvoid *_vertices; //顶点坐标
GLvoid *_originalVertices; //原始顶点坐标
GLushort *_indices; //索引
```

```c
//根据纹理大小，网格大小，初始化成员变量
//包括所有顶点的纹理坐标和位置、索引（绘制三角形）
void Grid3D::calculateVertexPoints(void)
```
```c
//就是将_originalVertices重新赋给_vertices
//_reuseGrid减1
void Grid3D::reuse(void)
```
```c
//执行到这个函数，说明前面的FBO已经绘制好了
//纹理也有了，顶点也有了
//然后通过glDrawElements进行绘制
void Grid3D::blit(void)
```

###TiledGrid3D
TiledGrid3D继承于GridBase，实现了三个虚函数，它是一个瓦片阵

它和Grid3D成员变量相同，这里只讲他们不一样的地方：
* 假设它是3*4的网格，那么就有12个Quad，一个Quad有4个顶点，那么它的顶点数就是12x4=48个。
* 它有自己的setTile, getTile, getOriginalTile函数，Tile其实就是4个顶点构成的struct。

由于顶点的意义不一样了，所以方法`calculateVertexPoints`完全不同，其他2个虚函数基本一致。

<h4><u>总之，我们知道GridBase能够帮助NodeGrid实现将所要绘制的画面打散成许多顶点，这样我们就能够通过
控制每个顶点的位置等属性，来实现所谓的网格动画，这就是网格动画的基本原理。</u></h4>

接下来我们来具体看看怎么去控制这些顶点，达到我们需要的效果

##网格动画
和网格动画相关的类包括`GridAction`和`Grid3DAction`，
我们可以看到继承关系是：

`Grid3DAction`/`TiledGrid3DAction` -- > `GridAction` --> `ActionInterval`

还有许多cocos2d-x自带的网格动画效果在`CCActionGrid3D.h`文件里

###GridAction

```c
/** @brief Base class for Grid actions */
class CC_DLL GridAction : public ActionInterval
{
public:

    //需要子类实现，用于创建一个网格
    //Grid3DAction会创建一个Grid3D
    //TiledGrid3DAction会创建一个TiledGrid3D
    virtual GridBase* getGrid();

    // overrides
	virtual GridAction * clone() const override = 0;
    virtual GridAction* reverse() const override;
    virtual void startWithTarget(Node *target) override;

protected:
    GridAction() {}
    virtual ~GridAction() {}
    /** initializes the action with size and duration */
    bool initWithDuration(float duration, const Size& gridSize);

    Size _gridSize; //网格的大小

    NodeGrid* _gridNodeTarget; //将target node保存起来，它必须是一个NodeGrid

    void cacheTargetAsGridNode();

private:
    CC_DISALLOW_COPY_AND_ASSIGN(GridAction);
};

//_target 必须继承于 NodeGrid
void GridAction::cacheTargetAsGridNode()
{
    _gridNodeTarget = dynamic_cast<NodeGrid*> (_target);
    CCASSERT(_gridNodeTarget, "GridActions can only used on NodeGrid");
}
void GridAction::startWithTarget(Node *target)
{
    ActionInterval::startWithTarget(target);
    cacheTargetAsGridNode();
    //创建一个新的网格
    GridBase *newgrid = this->getGrid();
    //第一次调用该函数时，targetGrid == nullptr
    //再次播放动画时，targetGrid不为nullptr
    GridBase *targetGrid = _gridNodeTarget->getGrid();

  //默认_reuseGrid==0 需要调用setReuseGrid来设置重用次数
    if (targetGrid && targetGrid->getReuseGrid() > 0)
    {
        if (targetGrid->isActive() && targetGrid->getGridSize().width == _gridSize.width
            && targetGrid->getGridSize().height == _gridSize.height /*&& dynamic_cast<GridBase*>(targetGrid) != nullptr*/)
        {
            targetGrid->reuse();
        }
        else
        {
            CCASSERT(0, "");
        }
    }
    else
    { //如果重用次数到了，那么重新设置新的网格
        if (targetGrid && targetGrid->isActive())
        {
            targetGrid->setActive(false);
        }
        //target中的网格就是在这里传入的
        _gridNodeTarget->setGrid(newgrid);
        _gridNodeTarget->getGrid()->setActive(true);
    }
}
```

`Grid3DAction`和`TiledGrid3DAction`分别实现了`getGrid`方法，还增加了各自set、get顶点和瓦片的方法，并没有新的属性。

`Grid3DAction`和`TiledGrid3DAction`并不能直接使用，让网格动起来，需要重写update方法。

在文件`CCActionGrid3D.h`中，定义了许多网格动画的效果，他们要么继承于`Grid3DAction`，要么继承于`TiledGrid3DAction`。

###Waves3D
能够设置振幅和频率，以及波纹个数。
只需要看看update方法就明白了
```c
void Waves3D::update(float time)
{
    int i, j;
    for (i = 0; i < _gridSize.width + 1; ++i)
    {
        for (j = 0; j < _gridSize.height + 1; ++j)
        {
            //获得原始顶点参数
            Vec3 v = getOriginalVertex(Vec2(i ,j));
            //计算顶点的z值
            v.z += (sinf((float)M_PI * time * _waves * 2 + (v.y+v.x) * 0.01f) * _amplitude * _amplitudeRate);
            //将新的顶点值设置到Grid3D中去
            setVertex(Vec2(i, j), v);
        }
    }
}
```
这样就实现了随着时间的变化，z值上下波动，从而实现了波纹动画效果。

其他效果原理都是一样的，这里不再一一举例。

看文件`CCGridAction.h`中还有一些
`AccelDeccelAmplitude`和`DeccelAmplitude`，这些用于加速或减速波动的频率。它的原理和ActionEase类一样，包装一个
`ActionInterval`，在update里面对delta time做一些处理，然后在调用`ActionInterval`的方法设置频率。
