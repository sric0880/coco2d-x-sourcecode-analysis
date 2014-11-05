# cocos2d-x RenderTexture 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-11-05 16:02  -->

RenderTexture继承于Node，它是对openGL FBO的封装，我们能够直接在RenderTexture上绘制，然后将其作为纹理添加到场景中，或者导出为PNG和JPG格式。

它的成员属性包括：
```c
//flags: whether generate new modelView and projection matrix or not
bool         _keepMatrix;
Rect         _rtTextureRect; //渲染矩形，和画布大小无关，和资源大小有关
Rect         _fullRect; //不知道和_rtTextureRect的关系
Rect         _fullviewPort; //和画布大小有关的

GLuint       _FBO;   //FBO句柄
GLuint       _depthRenderBufffer;  //FBO开启的深度缓冲区（RenderBuffer）
GLint        _oldFBO;  //渲染之前的FBO句柄
Texture2D* _texture;  //渲染得到的纹理
Texture2D* _textureCopy;    // a copy of _texture
Image*     _UITextureImage; //用于图片输出
Texture2D::PixelFormat _pixelFormat;  //纹理像素格式 两种：RGBA8888 RGB888

// code for "auto" update
GLbitfield   _clearFlags;
Color4F    _clearColor;
GLclampf     _clearDepth;
GLint        _clearStencil;
bool         _autoDraw;

/** The Sprite being used.
 The sprite, by default, will use the following blending function: GL_ONE, GL_ONE_MINUS_SRC_ALPHA.
 The blending function can be changed in runtime by calling:
 - renderTexture->getSprite()->setBlendFunc((BlendFunc){GL_ONE, GL_ONE_MINUS_SRC_ALPHA});
 */
Sprite* _sprite;    //根据渲染的纹理生成的Sprite

GroupCommand _groupCommand; //所有绘制命令都会放在group里面
CustomCommand _beginWithClearCommand; //调用onBegin()和onClear()
CustomCommand _clearDepthCommand; //调用onClearDepth()
CustomCommand _clearCommand; //调用onClear()
CustomCommand _beginCommand; //只调用onBegin()
CustomCommand _endCommand; //调用onEnd()
CustomCommand _saveToFileCommand; //调用onSaveToFile()
```

初始化函数：
```c
bool RenderTexture::initWithWidthAndHeight(int w, int h, Texture2D::PixelFormat format, GLuint depthStencilFormat)
{
    CCASSERT(format != Texture2D::PixelFormat::A8, "only RGB and RGBA formats are valid for a render texture");

    bool ret = false;
    void *data = nullptr;
    do
    {
        初始化绘制矩形大小
        _fullRect = _rtTextureRect = Rect(0,0,w,h);
        //Size size = Director::getInstance()->getWinSizeInPixels();
        //_fullviewPort = Rect(0,0,size.width,size.height);
        w = (int)(w * CC_CONTENT_SCALE_FACTOR());
        h = (int)(h * CC_CONTENT_SCALE_FACTOR());
        _fullviewPort = Rect(0,0,w,h);

        //获得当前的FBO，用以恢复
        glGetIntegerv(GL_FRAMEBUFFER_BINDING, &_oldFBO);

        // textures must be power of two squared
        int powW = 0;
        int powH = 0;

        if (Configuration::getInstance()->supportsNPOT())
        {
            powW = w;
            powH = h;
        }
        else
        {
            powW = ccNextPOT(w); //计算最近的2的n次方
            powH = ccNextPOT(h);
        }

        auto dataLen = powW * powH * 4;
        data = malloc(dataLen);
        CC_BREAK_IF(! data);

        memset(data, 0, dataLen);
        _pixelFormat = format;

        _texture = new Texture2D(); //初始化空白纹理
        if (_texture)
        {
            _texture->initWithData(data, dataLen, (Texture2D::PixelFormat)_pixelFormat, powW, powH, Size((float)w, (float)h));
        }
        else
        {
            break;
        }
        //获得当前的Render Buffer Object，用以恢复
        GLint oldRBO;
        glGetIntegerv(GL_RENDERBUFFER_BINDING, &oldRBO);

        if (Configuration::getInstance()->checkForGLExtension("GL_QCOM")) //为了解决特殊平台上clear的bug
        {
            _textureCopy = new Texture2D();
            if (_textureCopy)
            {
                _textureCopy->initWithData(data, dataLen, (Texture2D::PixelFormat)_pixelFormat, powW, powH, Size((float)w, (float)h));
            }
            else
            {
                break;
            }
        }

        // generate FBO(frame buffer object)
        glGenFramebuffers(1, &_FBO);
        glBindFramebuffer(GL_FRAMEBUFFER, _FBO);

        // associate texture with FBO
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, _texture->getName(), 0);

        if (depthStencilFormat != 0)
        {
            //create and attach depth buffer
            glGenRenderbuffers(1, &_depthRenderBufffer);
            glBindRenderbuffer(GL_RENDERBUFFER, _depthRenderBufffer);
            glRenderbufferStorage(GL_RENDERBUFFER, depthStencilFormat, (GLsizei)powW, (GLsizei)powH);
            glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, _depthRenderBufffer);

            // if depth format is the one with stencil part, bind same render buffer as stencil attachment
            if (depthStencilFormat == GL_DEPTH24_STENCIL8)
            {
                glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_STENCIL_ATTACHMENT, GL_RENDERBUFFER, _depthRenderBufffer);
            }
        }

        // check if it worked (probably worth doing :) )
        CCASSERT(glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE, "Could not attach texture to framebuffer");

        //设置纹理的缩放规则
        _texture->setAliasTexParameters();

        // 将纹理转换成Sprite
        setSprite(Sprite::createWithTexture(_texture));

        _texture->release(); //释放纹理
        _sprite->setFlippedY(true);

        _sprite->setBlendFunc( BlendFunc::ALPHA_PREMULTIPLIED );

        //还原之前的FBO和RBO
        glBindRenderbuffer(GL_RENDERBUFFER, oldRBO);
        glBindFramebuffer(GL_FRAMEBUFFER, _oldFBO);

        // Diabled by default.
        _autoDraw = false;

        // add sprite for backward compatibility
        addChild(_sprite);

        ret = true;
    } while (0);

    CC_SAFE_FREE(data);

    return ret;
}
```

基本使用方法：
```c
auto rend = RenderTexture::create(32, 64, Texture2D::PixelFormat::RGBA8888);
rend->begin();
...
sprite->visit();
...
rend->end();
```

##begin() & beginWithClear() & onBegin()
```c
void RenderTexture::begin()
{
    Director* director = Director::getInstance();
    CCASSERT(nullptr != director, "Director is null when seting matrix stack");

    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION); //复制top投影矩阵加入stack
    _projectionMatrix = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION); //取得该投影矩阵

    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);//复制top modelview矩阵加入stack
    _transformMatrix = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW); //取得该modelview矩阵

    if(!_keepMatrix) //如果不保持矩阵，那么重新定义平行投影矩阵
    {
        director->setProjection(director->getProjection());

        const Size& texSize = _texture->getContentSizeInPixels();

        // Calculate the adjustment ratios based on the old and new projections
        Size size = director->getWinSizeInPixels();

        float widthRatio = size.width / texSize.width;
        float heightRatio = size.height / texSize.height;

        Mat4 orthoMatrix;
        Mat4::createOrthographicOffCenter((float)-1.0 / widthRatio, (float)1.0 / widthRatio, (float)-1.0 / heightRatio, (float)1.0 / heightRatio, -1, 1, &orthoMatrix);
        director->multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, orthoMatrix);
    }

    _groupCommand.init(_globalZOrder);  //初始化GroupCommand

    Renderer *renderer =  Director::getInstance()->getRenderer();
    renderer->addCommand(&_groupCommand); //加入到渲染列队
    renderer->pushGroup(_groupCommand.getRenderQueueID());//从此之后，所有Command都加入到group中

    _beginCommand.init(_globalZOrder); //构造beginCommand
    _beginCommand.func = CC_CALLBACK_0(RenderTexture::onBegin, this);

    Director::getInstance()->getRenderer()->addCommand(&_beginCommand); //将beginCommand加入到groupCommand中去
}
```

```c
///先设置clear的颜色、深度值、模板值、flags，再调用上述begin()函数
///再将clear的命令发送到渲染列队中
void RenderTexture::beginWithClear(float r, float g, float b, float a, float depthValue, int stencilValue, GLbitfield flags)
{
    setClearColor(Color4F(r, g, b, a));

    setClearDepth(depthValue);

    setClearStencil(stencilValue);

    setClearFlags(flags);

    this->begin();

    //clear screen
    _beginWithClearCommand.init(_globalZOrder);
    _beginWithClearCommand.func = CC_CALLBACK_0(RenderTexture::onClear, this);
    Director::getInstance()->getRenderer()->addCommand(&_beginWithClearCommand);
}
```

```c
void RenderTexture::onBegin()
{
    //
    Director *director = Director::getInstance();

    _oldProjMatrix = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, _projectionMatrix);

    _oldTransMatrix = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _transformMatrix);

    if(!_keepMatrix)同上
    {
        director->setProjection(director->getProjection());

#if CC_TARGET_PLATFORM == CC_PLATFORM_WP8
        Mat4 modifiedProjection = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);
        modifiedProjection = CCEGLView::sharedOpenGLView()->getReverseOrientationMatrix() * modifiedProjection;
        director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION,modifiedProjection);
#endif

        const Size& texSize = _texture->getContentSizeInPixels();

        // Calculate the adjustment ratios based on the old and new projections
        Size size = director->getWinSizeInPixels();
        float widthRatio = size.width / texSize.width;
        float heightRatio = size.height / texSize.height;

        Mat4 orthoMatrix;
        Mat4::createOrthographicOffCenter((float)-1.0 / widthRatio, (float)1.0 / widthRatio, (float)-1.0 / heightRatio, (float)1.0 / heightRatio, -1, 1, &orthoMatrix);
        director->multiplyMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, orthoMatrix);
    }
    else
    {
#if CC_TARGET_PLATFORM == CC_PLATFORM_WP8
        Mat4 modifiedProjection = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);
        modifiedProjection = CCEGLView::sharedOpenGLView()->getReverseOrientationMatrix() * modifiedProjection;
        director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, modifiedProjection);
#endif
    }

    //calculate viewport
    {
        Rect viewport;
        viewport.size.width = _fullviewPort.size.width;
        viewport.size.height = _fullviewPort.size.height;
        float viewPortRectWidthRatio = float(viewport.size.width)/_fullRect.size.width;
        float viewPortRectHeightRatio = float(viewport.size.height)/_fullRect.size.height;
        viewport.origin.x = (_fullRect.origin.x - _rtTextureRect.origin.x) * viewPortRectWidthRatio;
        viewport.origin.y = (_fullRect.origin.y - _rtTextureRect.origin.y) * viewPortRectHeightRatio;
        //glViewport(_fullviewPort.origin.x, _fullviewPort.origin.y, (GLsizei)_fullviewPort.size.width, (GLsizei)_fullviewPort.size.height);
        glViewport(viewport.origin.x, viewport.origin.y, (GLsizei)viewport.size.width, (GLsizei)viewport.size.height);
    }

    // 以上就是调整投影矩阵和视图大小
    /*以下是关键代码*/
    glGetIntegerv(GL_FRAMEBUFFER_BINDING, &_oldFBO); //获得当前的FBO
    glBindFramebuffer(GL_FRAMEBUFFER, _FBO); //绑定到新的FBO

    //TODO move this to configration, so we don't check it every time
    /*  Certain Qualcomm Andreno gpu's will retain data in memory after a frame buffer switch which corrupts the render to the texture. The solution is to clear the frame buffer before rendering to the texture. However, calling glClear has the unintended result of clearing the current texture. Create a temporary texture to overcome this. At the end of RenderTexture::begin(), switch the attached texture to the second one, call glClear, and then switch back to the original texture. This solution is unnecessary for other devices as they don't have the same issue with switching frame buffers.
     */
    if (Configuration::getInstance()->checkForGLExtension("GL_QCOM"))
    {
        // -- bind a temporary texture so we can clear the render buffer without losing our texture
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, _textureCopy->getName(), 0);
        CHECK_GL_ERROR_DEBUG();
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, _texture->getName(), 0);
    }
}
```

##end() & onEnd()
```c
void RenderTexture::end()
{
    _endCommand.init(_globalZOrder);
    _endCommand.func = CC_CALLBACK_0(RenderTexture::onEnd, this);

    Director* director = Director::getInstance();
    CCASSERT(nullptr != director, "Director is null when seting matrix stack");

    Renderer *renderer = director->getRenderer();
    renderer->addCommand(&_endCommand);
    renderer->popGroup(); //GroupCommand结束

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION); //弹出 对应begin中的pushMatrix
    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW); //弹出 对应begin中的pushMatrix

}
```

```c
void RenderTexture::onEnd()
{
    Director *director = Director::getInstance();
    //还原帧缓冲区
    glBindFramebuffer(GL_FRAMEBUFFER, _oldFBO);

    // restore viewport
    director->setViewport();
    //还原投影矩
     director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION, _oldProjMatrix);
//还原modelview矩阵
 director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _oldTransMatrix);
}
```

所以，在begin和end中间的代码，如果有渲染Command都是加入到了RenderTexture的一个groupCommand中，并且渲染的目标不是标准的帧缓冲区而是FBO中的Texture。

当end()结束后，渲染恢复到正常的帧缓冲区。

此时，我们可以选择：
1. 将RenderTexture作为一个node加入场景
2. 保存一个PNG或JPG图片到磁盘
3. 获得一个Sprite

要想实现将RenderTexture当做一个node，需要实现node的两个函数：
visit和draw
```c
void RenderTexture::visit(Renderer *renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    // override visit.
	// Don't call visit on its children
    if (!_visible)
    {
        return;
    }

    uint32_t flags = processParentFlags(parentTransform, parentFlags);

    Director* director = Director::getInstance();
    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);
    //Sprite估计没有儿子节点，visit在这里没用
    _sprite->visit(renderer, _modelViewTransform, flags);
    draw(renderer, _modelViewTransform, flags);

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    _orderOfArrival = 0;
}
```
```c
///保证儿子节点的绘制在begin和end之间
///所有儿子节点的绘制最终都会反映在_sprite上，通过正常绘制_sprite即可
void RenderTexture::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    if (_autoDraw)
    {
        //Begin will create a render group using new render target
        begin();

        //clear screen
        _clearCommand.init(_globalZOrder);
        _clearCommand.func = CC_CALLBACK_0(RenderTexture::onClear, this);
        renderer->addCommand(&_clearCommand);

        //! make sure all children are drawn
        sortAllChildren();

        for(const auto &child: _children)
        {
            if (child != _sprite)//这里只绘制除_sprite之外的节点
                child->visit(renderer, transform, flags);
        }

        //End will pop the current render group
        end();
    }
}
```

关于保存为图片的代码这里就不再分析了
主要原理就是从FBO中读取像素数据。
