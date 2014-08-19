# cocos2d-x ParticleSystem源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-19 19:02  -->

关于粒子系统里面各种参数比如初始速度、生命周期这里不再详述。主要来看一下它的原理。

`ParticleSystem`类继承于`Node`，它维护一组`tParticle`，`tParticle`指众多粒子中的一个。它
包含颜色、坐标、旋转角度等成员变量。

驱动`ParticleSystem`开始运动的地方在函数`onEnter`
```c
void ParticleSystem::onEnter()
{
  ......
  //开始每帧调用update
    this->scheduleUpdateWithPriority(1);
}
void ParticleSystem::onExit()
{
    this->unscheduleUpdate(); //停止调用update
    Node::onExit();
}
```
`ParticleSystem::update`函数会不断的调用addParticle()来往系统中添加新的粒子，并且循环遍历所有的粒子，
在生命周期内的粒子，就改变其状态。已经不在生命周期的粒子，就把它替换成数组末尾的粒子，并且粒子个数
减一。

`ParticleSystem`不能直接用在程序中，因为它并没有实现`draw`函数，所以在程序中看不出任何效果。

##ParticleSystemQuad
`ParticleSystemQuad`继承于`ParticleSystem`
它具有如下属性：
```c
//XXX: 我实在不明白，使用了QuadCommand为什么还需要自己定义VAO和VBO
//这些事情不是已经交给Render类去做了吗？
//3.0之前直接在draw函数里面调用glDraw*函数，确实需要用到VAO和VBO
//现在已经将渲染的事情交给了Render，就没必要自己创建VAO和VBO了
//估计他们程序员太累了，反正不影响效果，就只改了draw函数
V3F_C4B_T2F_Quad    *_quads;        // quads to be rendered
GLushort            *_indices;      // indices
GLuint              _VAOname;
GLuint              _buffersVBO[2]; //0: vertex  1: indices

QuadCommand _quadCommand;           // quad command
```
它的draw函数，将_quadCommand发送给Render
```c
void ParticleSystemQuad::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    CCASSERT( _particleIdx == 0 || _particleIdx == _particleCount, "Abnormal error in particle quad");
    //quad command
    if(_particleIdx > 0)
    {
        _quadCommand.init(_globalZOrder, _texture->getName(), getGLProgramState(), _blendFunc, _quads, _particleIdx, transform);
        renderer->addCommand(&_quadCommand);
    }
}
```
`ParticleSystemQuad`重写了`ParticleSystem`的函数`updateQuadWithParticle`，这是一个非常关键的函数。
该函数负责将update更新后的每个粒子的属性再转移到Quad上，更新后的quad自然就通过quadCommand发送给渲染列队了。

`ParticleSystemQuad`还能能够加入到Batch里去，它叫`ParticleBatchNode`

##ParticleBatchNode
可以将多个`ParticleSystemQuad`放入其中，能够一次性绘制，但是必须保证它们都使用相同的纹理

它和`SpriteBatchNode`一样都有这两个成员变量：
```c
/** the texture atlas used for drawing the quads */
TextureAtlas* _textureAtlas;
// batch command
BatchCommand _batchCommand;
```
关于`SpriteBatchNode`请参见`cocos2d-x Texture,TextureAtlas源码解析`

`ParticleBatchNode`重写了`addChild`函数，因为它只支持`ParticleSystem`作为它的子节点，
(并将`ParticleSystem`中的Quad取出加入到_textureAtlas中去)。

*括号中的这句话其实是错误的，它和`SpriteBatchNode`不一样，因为每一个`ParticleSystem`都可能有成百上千个
粒子，所以如果将每一个`ParticleSystem`中的所有Quad通过memcpy到_textureAtlas中去除了占用不必要的内存外，
效率也不高。而且最大的问题是，每一次`ParticleSystem`update之后，Quad都发生了改变，需要将这些改变发送给
_textureAtlas可能又需要进行memcpy，效率大大下降。*

***解决方案：<u>如果`ParticleSystem`添加到了batch中，那么它本身不维持自己的Quad数组，
它的Quad数据全部在batchNode中的_textureAtlas里面，并记录一个AtlasIndex标示它的quad数据在_textureAtlas中的起始位置
。同时，update函数如果发现自己已经被batch了，那么就直接更新_textureAtlas里的quad数组了</u>***

我们来看看`ParticleBatchNode`是如何防止绘制它的儿子节点的：
```c
//visit函数并没有递归调用它儿子节点的visit函数
void ParticleBatchNode::visit(Renderer *renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    if (!_visible)
    {
        return;
    }

    uint32_t flags = processParentFlags(parentTransform, parentFlags);

    // IMPORTANT:
    // To ease the migration to v3.0, we still support the Mat4 stack,
    // but it is deprecated and your code should not rely on it
    Director* director = Director::getInstance();
    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);

    draw(renderer, _modelViewTransform, flags);

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
}
```
它的`draw`函数，是不是很眼熟，和`SpriteBatchNode`几乎一模一样
```c
void ParticleBatchNode::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    CC_PROFILER_START("CCParticleBatchNode - draw");

    if( _textureAtlas->getTotalQuads() == 0 )
    {
        return;
    }

    _batchCommand.init(
                       _globalZOrder,
                       getGLProgram(),
                       _blendFunc,
                       _textureAtlas,
                       _modelViewTransform);
    renderer->addCommand(&_batchCommand);
    CC_PROFILER_STOP("CCParticleBatchNode - draw");
}
```
其他主要方法就是对`ParticleSystem`节点的增、删了，这里不再详述。
