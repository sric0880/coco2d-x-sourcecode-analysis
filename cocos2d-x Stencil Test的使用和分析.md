# cocos2d-x Stencil Test的使用和分析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-21 14:43  -->

本文分析cocos2d-x中使用到的模板测试的类`ClippingNode`，顾名思义，就是用于裁切的节点，怎么裁切呢，这就用到了Stencil Test。关于效果，可以参见cpp-test例子中的
`ClippingNodeTest`

先看看Stencil Test的基本工作流程：
1. 启动模板测试
```c
glStencilMask(0xFF);//初始化一个模板掩码为0xFF
glEnable(GL_STENCIL_TEST);//启动模板测试
```
2. 选择比较函数
```c
//func - 比较函数
//ref - 参考值
//mask﻿ - 掩码
glStencilFunc(GLenum func, GLint ref, GLuint mask);
```
openGL提供这些比较函数：
 - GL_NEVER - Always fails. 此时参考值无意义
 - GL_LESS - Passes if ( ref & mask ) < ( stencil & mask ).
 - GL_LEQUAL - Passes if ( ref & mask ) <= ( stencil & mask ).
 - GL_GREATER - Passes if ( ref & mask ) > ( stencil & mask ).
 - GL_GEQUAL - Passes if ( ref & mask ) >= ( stencil & mask ).
 - GL_EQUAL - Passes if ( ref & mask ) = ( stencil & mask ).
 - GL_NOTEQUAL - Passes if ( ref & mask ) != ( stencil & mask ).
 - GL_ALWAYS - Always passes. 此时参考值无意义

  stencil表示模板缓冲区中的值。通过测试或未通过测试都的将进行下一步操作。

3. 指定下一步操作
```c
//该函数指定了三种情况下“模板值”该如何变化
//fail - 表示模板测试未通过时该如何变化
//zfail - 表示模板测试通过，但深度测试未通过时该如何变化
//zpass - 表示模板测试和深度测试或者未执行深度测试均通过时该如何变化
glStencilOp(GLenum fail, GLenum zfail, GLenum zpass);
```
可供选择的参数：
 - GL_KEEP（不改变，这也是默认值）
 - GL_ZERO（回零）
 - GL_REPLACE（使用测试条件中的设定值来代替当前模板值）
 - GL_INCR（增加1，但如果已经是最大值，则保持不变）
 - GL_INCR_WRAP（增加1，但如果已经是最大值，则从零重新开始）
 - GL_DECR（减少1，但如果已经是零，则保持不变）
 - GL_DECR_WRAP（减少1，但如果已经是零，则重新设置为最大值）
 - GL_INVERT（按位取反）

4. 绘制，模板缓冲区没有通过的不会绘制到颜色缓冲区的
  1. draw something…//模板图形
  2. 重新执行上面2，3步骤
  3. draw something…//正式绘图

接下来我们介绍coco2d-x为我们封装好的类型`ClippingNode`，
`ClippingNode`给我提供了一个用其他图形镂空另外一个图形的方法。

先看看它又哪些成员变量：
```c
Node* _stencil; //充当模板的node
//只有当_stencil的alpha值大于这个值的时候
//才往模板缓冲区中写入
GLfloat _alphaThreshold;
//true - 模板缓冲区没有写入的地方，就绘制
bool    _inverted;

//renderData and callback
void onBeforeVisit();
void onAfterDrawStencil();
void onAfterVisit();

/** 模板测试的参数*/
GLboolean _currentStencilEnabled;
GLuint _currentStencilWriteMask;
GLenum _currentStencilFunc;
GLint _currentStencilRef;
GLuint _currentStencilValueMask;
GLenum _currentStencilFail;
GLenum _currentStencilPassDepthFail;
GLenum _currentStencilPassDepthPass;
GLboolean _currentDepthWriteMask;

/** alpha test的参数*/
GLboolean _currentAlphaTestEnabled;
GLenum _currentAlphaTestFunc;
GLclampf _currentAlphaTestRef;

GLint _mask_layer_le; //掩码

/**渲染命令*/
GroupCommand _groupCommand;
CustomCommand _beforeVisitCmd;
CustomCommand _afterDrawStencilCmd;
CustomCommand _afterVisitCmd;
```
直接看visit方法，这里不贴代码了，关键流程是这样：
1. 执行函数`ClippingNode::onBeforeVisit`
2. 如果_alphaThreshold<0，那么重新设置_stencil的program
3. 绘制_stencil
4. 执行函数`ClippingNode::onAfterDrawStencil`
5. 绘制儿子节点以及自身
6. 执行函数`ClippingNode::onAfterVisit`

上面三个函数主要是我前面说过的OpenGL的函数的调用，具体来看看是如何用的：
```c
//初始化一些参数
void ClippingNode::onBeforeVisit()
{
    // 这是一个全局变量，
    s_layer++;
    //s_layer从0开始,这样保证mask_layer从0x1开始
    //如果该节点还有一个儿子节点也是ClippingNode，那么s_layer就变成了2
    //这样mask_layer就变成了0b00000010
    // mask of the current layer (ie: for layer 3: 00000100)
    GLint mask_layer = 0x1 << s_layer;
    // mask of all layers less than the current (ie: for layer 3: 00000011)
    GLint mask_layer_l = mask_layer - 1;
    // mask of all layers less than or equal to the current (ie: for layer 3: 00000111)
    _mask_layer_le = mask_layer | mask_layer_l;

    //保存stencil的状态
    _currentStencilEnabled = glIsEnabled(GL_STENCIL_TEST);
    glGetIntegerv(GL_STENCIL_WRITEMASK, (GLint *)&_currentStencilWriteMask);
    glGetIntegerv(GL_STENCIL_FUNC, (GLint *)&_currentStencilFunc);
    glGetIntegerv(GL_STENCIL_REF, &_currentStencilRef);
    glGetIntegerv(GL_STENCIL_VALUE_MASK, (GLint *)&_currentStencilValueMask);
    glGetIntegerv(GL_STENCIL_FAIL, (GLint *)&_currentStencilFail);
    glGetIntegerv(GL_STENCIL_PASS_DEPTH_FAIL, (GLint *)&_currentStencilPassDepthFail);
    glGetIntegerv(GL_STENCIL_PASS_DEPTH_PASS, (GLint *)&_currentStencilPassDepthPass);

    // 开启
    glEnable(GL_STENCIL_TEST);
    CHECK_GL_ERROR_DEBUG();

    // 除了掩码所在bit,模板缓冲区中的其他bits是只读的
    glStencilMask(mask_layer);

    // manually save the depth test state
    glGetBooleanv(GL_DEPTH_WRITEMASK, &_currentDepthWriteMask);

    //就是说往模板缓冲区中绘制，又不是绘制到屏幕上，没有必要做深度测试
    glDepthMask(GL_FALSE);

    //不通过测试（永远不会写入颜色缓冲区），没有反转，就写入0，否则写入1
    glStencilFunc(GL_NEVER, mask_layer, mask_layer);
    glStencilOp(!_inverted ? GL_ZERO : GL_REPLACE, GL_KEEP, GL_KEEP);

    //通过绘制一个矩形来初始化模板缓冲区
    //这个时候，模板缓冲区要么全是0 要么全是1
    drawFullScreenQuadClearStencil();

    //不通过测试（永远不会写入颜色缓冲区），没有反转，就写入1，否则写入0
    glStencilFunc(GL_NEVER, mask_layer, mask_layer);
    glStencilOp(!_inverted ? GL_REPLACE : GL_ZERO, GL_KEEP, GL_KEEP);

    if (_alphaThreshold < 1) {
#if (CC_TARGET_PLATFORM == CC_PLATFORM_MAC || CC_TARGET_PLATFORM == CC_PLATFORM_WINDOWS || CC_TARGET_PLATFORM == CC_PLATFORM_LINUX)
        // manually save the alpha test state
        _currentAlphaTestEnabled = glIsEnabled(GL_ALPHA_TEST);
        glGetIntegerv(GL_ALPHA_TEST_FUNC, (GLint *)&_currentAlphaTestFunc);
        glGetFloatv(GL_ALPHA_TEST_REF, &_currentAlphaTestRef);
        // enable alpha testing
        glEnable(GL_ALPHA_TEST);
        // check for OpenGL error while enabling alpha test
        CHECK_GL_ERROR_DEBUG();
        // pixel will be drawn only if greater than an alpha threshold
        glAlphaFunc(GL_GREATER, _alphaThreshold);
#else

#endif
    }

    //Draw _stencil
    //画完_stencil模板缓冲区中既有0 也有1
}
```

```c
void ClippingNode::onAfterDrawStencil()
{
    // restore alpha test state
    if (_alphaThreshold < 1)
    {
#if (CC_TARGET_PLATFORM == CC_PLATFORM_MAC || CC_TARGET_PLATFORM == CC_PLATFORM_WINDOWS || CC_TARGET_PLATFORM == CC_PLATFORM_LINUX)
        // manually restore the alpha test state
        glAlphaFunc(_currentAlphaTestFunc, _currentAlphaTestRef);
        if (!_currentAlphaTestEnabled)
        {
            glDisable(GL_ALPHA_TEST);
        }
#else
// XXX: we need to find a way to restore the shaders of the stencil node and its childs
#endif
    }

    //因为要开始往颜色缓冲区中写入了，所有要开启深度缓冲区
    glDepthMask(_currentDepthWriteMask);

    // 如果_mask_layer_le == (_mask_layer_le & stencil)，就通过
    // 即stencil>=_mask_layer_le,就通过了测试
    // 也就是说，模板缓冲区有n层嵌套，就必须n位bit填满1.
    // 比如，3层嵌套，那么到第三层的时候，必须满足stencil==0b0000 0111
    // 才能够通过测试
    glStencilFunc(GL_EQUAL, _mask_layer_le, _mask_layer_le);
    glStencilOp(GL_KEEP, GL_KEEP, GL_KEEP);

    // draw (according to the stencil test func) this node and its childs
}
```

```c
//做一些清理工作
void ClippingNode::onAfterVisit()
{
    // 恢复之前模板缓冲区的状态
    glStencilFunc(_currentStencilFunc, _currentStencilRef, _currentStencilValueMask);
    glStencilOp(_currentStencilFail, _currentStencilPassDepthFail, _currentStencilPassDepthPass);
    glStencilMask(_currentStencilWriteMask);
    if (!_currentStencilEnabled)
    {
        glDisable(GL_STENCIL_TEST);
    }
    // we are done using this layer, decrement
    s_layer--;
}
```
