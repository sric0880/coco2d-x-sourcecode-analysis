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
     *
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

    Node* _gridTarget;
    GridBase* _nodeGrid; //网格数据结构
    GroupCommand _groupCommand; //批量绘图命令
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

    _groupCommand.init(_globalZOrder);
    renderer->addCommand(&_groupCommand);
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
        _nodeGrid->set2DProjection();
    }

    _gridBeginCommand.init(_globalZOrder);
    _gridBeginCommand.func = CC_CALLBACK_0(NodeGrid::onGridBeginDraw, this);
    renderer->addCommand(&_gridBeginCommand);


    if(_gridTarget)
    {
        _gridTarget->visit(renderer, _modelViewTransform, dirty);
    }

    int i = 0;

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
        // restore projection
        director->setProjection(beforeProjectionType);
    }

    _gridEndCommand.init(_globalZOrder);
    _gridEndCommand.func = CC_CALLBACK_0(NodeGrid::onGridEndDraw, this);
    renderer->addCommand(&_gridEndCommand);

    renderer->popGroup();

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
}

void NodeGrid::setGrid(GridBase *grid)
{
    CC_SAFE_RELEASE(_nodeGrid);
    CC_SAFE_RETAIN(grid);
    _nodeGrid = grid;
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
    /** create one Grid */
    static GridBase* create(const Size& gridSize, Texture2D *texture, bool flipped);
    /** create one Grid */
    static GridBase* create(const Size& gridSize);
    /**
     * @js NA
     * @lua NA
     */
    virtual ~GridBase(void);

    bool initWithSize(const Size& gridSize, Texture2D *texture, bool flipped);
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

    void beforeDraw(void);
    void afterDraw(Node *target);
    virtual void blit(void);
    virtual void reuse(void);
    virtual void calculateVertexPoints(void);

    void set2DProjection(void);

protected:
    bool _active;
    int  _reuseGrid; //重用次数
    Size _gridSize; //mxn的格子
    Texture2D *_texture; //纹理
    Vec2 _step; //每个格子的像素长和宽
    Grabber *_grabber;
    bool _isTextureFlipped;
    GLProgram* _shaderProgram;
    Director::Projection _directorProjection;
};
```
GridAction --> ActionInterval

```c
/** @brief Base class for Grid actions */
class CC_DLL GridAction : public ActionInterval
{
public:

    /** returns the grid */
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

    Size _gridSize;

    NodeGrid* _gridNodeTarget;

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


```
