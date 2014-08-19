# cocos2d-x Texture纹理 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-19 12:56  -->

介绍主要的三个类：`Texture2D`, `TextureCache`, `TextureAtlas`，然后介绍Sprite、SpriteBatchNode、粒子系统是如何使用这些类的。

##Texture2D
先来看有哪些成员变量：
```c
/** pixel format of the texture */
    Texture2D::PixelFormat _pixelFormat;

    /** width in pixels */
    int _pixelsWide;

    /** height in pixels */
    int _pixelsHigh;

    /** texture name */
    GLuint _name;

    /** texture max S */
    GLfloat _maxS;

    /** texture max T */
    GLfloat _maxT;

    /** content size */
    Size _contentSize;

    /** whether or not the texture has their Alpha premultiplied */
    bool _hasPremultipliedAlpha;

    bool _hasMipmaps;

    /** shader program used by drawAtPoint and drawInRect */
    GLProgram* _shaderProgram;

    static const PixelFormatInfoMap _pixelFormatInfoTables;

    bool _antialiasEnabled;
```
建立texture在函数initWithMipmaps中，依次调用了如下方法：
```c
glPixelStorei
glGenTextures
bindTexture2D
 if (mipmapsNum == 1)
{
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, _antialiasEnabled ? GL_LINEAR : GL_NEAREST);
}else
{
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, _antialiasEnabled ? GL_LINEAR_MIPMAP_NEAREST : GL_NEAREST_MIPMAP_NEAREST);
}

glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, _antialiasEnabled ? GL_LINEAR : GL_NEAREST );
glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE );
glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE );

if (info.compressed)
{
      glCompressedTexImage2D(GL_TEXTURE_2D, i, info.internalFormat, (GLsizei)width, (GLsizei)height, 0, datalen, data);
}
else
{
      glTexImage2D(GL_TEXTURE_2D, i, info.internalFormat, (GLsizei)width, (GLsizei)height, 0, info.format, info.type, data);
}
setGLProgram(GLProgramCache::getInstance()->getGLProgram(GLProgram::SHADER_NAME_POSITION_TEXTURE));
```
关于pixelFormat以及如何转化，这里不再详细讲解

##TextureCache
在Director中维护了一个`TextureCache *_textureCache;`
一般不主动调用Texture2D的init函数，而是通过`TextureCache`去加载和释放纹理资源。
`TextureCache`给我们提供的加载函数：
```c
//通过一个文件名称，创建一个Texture2D，并加入到cache中
//Supported image extensions: .png, .bmp, .tiff, .jpeg, .pvr
Texture2D * TextureCache::addImage(const std::string &path)

//异步加载纹理到cache中
//将需要加载的文件路径和回调构成一个struct存入一个列队中
//另一个线程不停的循环读出列队中的struct，整个过程需要对列队加锁
//当列队为空时，线程并不退出，线程处于阻塞状态，等待struct加入列队
//如果需要线程退出，需要调用waitForQuit
//如果列队不为空，取出文件路径，先判断cache时候有，然后再判断列队中是否有
//如果没有，就通过initWithImageFileThreadSafe初始化一个Image
//然后将image和前面的struct再组成一个struct存入另一个queue（_imageInfoMutex）
//因为加载纹理需要在主线程完成，所以只能使用schedule每一帧都调用函数addImageAsyncCallBack
//该函数从上面的queue中取出image，然后通过initWithImage创建texture2D并加入到cache中，最后执行回调函数
void TextureCache::addImageAsync(const std::string &path, const std::function<void(Texture2D*)>& callback)

//将_imageInfoMutex中的等于filename的struct中的回调置于nullptr
virtual void unbindImageAsync(const std::string &filename);

//将_imageInfoMutex中的所有struct中的回调置于nullptr
virtual void unbindAllImageAsync();
```
还有许多remove函数，这里不再一一叙述，只是强调一点，不用的纹理，应该及时remove掉。

##TextureAtlas
TextureAtlas维持了多个Quad，它们都使用相同的纹理，使用相同的渲染参数，这样通过一次性绘制多个Quad来提高效率。
主要的成员变量：
```c
GLushort*           _indices; //索引
GLuint              _VAOname; //VAO
GLuint              _buffersVBO[2]; //0: vertex  1: indices
bool                _dirty; //indicates whether or not the array buffer of the VBO needs to be updated
/** quantity of quads that are going to be drawn */
ssize_t _totalQuads;
/** quantity of quads that can be stored with the current texture atlas size */
ssize_t _capacity;
/** Texture of the texture atlas */
Texture2D* _texture;//说明所有quad共享一个纹理
/** Quads that are going to be rendered */
V3F_C4B_T2F_Quad* _quads;
```
来看一下init函数
```c
bool TextureAtlas::initWithTexture(Texture2D *texture, ssize_t capacity)
{
    CCASSERT(capacity>=0, "Capacity must be >= 0");

//    CCASSERT(texture != nullptr, "texture should not be null");
    _capacity = capacity;
    _totalQuads = 0;

    // retained in property
    this->_texture = texture;
    CC_SAFE_RETAIN(_texture);

    // Re-initialization is not allowed
    CCASSERT(_quads == nullptr && _indices == nullptr, "");
    //按照_capacity的大小先把内存分配好
    _quads = (V3F_C4B_T2F_Quad*)malloc( _capacity * sizeof(V3F_C4B_T2F_Quad) );
    _indices = (GLushort *)malloc( _capacity * 6 * sizeof(GLushort) );

    if( ! ( _quads && _indices) && _capacity > 0)
    {
        //CCLOG("cocos2d: TextureAtlas: not enough memory");
        CC_SAFE_FREE(_quads);
        CC_SAFE_FREE(_indices);

        // release texture, should set it to null, because the destruction will
        // release it too. see cocos2d-x issue #484
        CC_SAFE_RELEASE_NULL(_texture);
        return false;
    }

    memset( _quads, 0, _capacity * sizeof(V3F_C4B_T2F_Quad) );
    memset( _indices, 0, _capacity * 6 * sizeof(GLushort) );

#if CC_ENABLE_CACHE_TEXTURE_DATA
    /** listen the event that renderer was recreated on Android/WP8 */
    _rendererRecreatedListener = EventListenerCustom::create(EVENT_RENDERER_RECREATED, CC_CALLBACK_1(TextureAtlas::listenRendererRecreated, this));
    Director::getInstance()->getEventDispatcher()->addEventListenerWithFixedPriority(_rendererRecreatedListener, -1);
#endif
  //给索引数组赋值
    this->setupIndices();
    //和Render类类似
    if (Configuration::getInstance()->supportsShareableVAO())
    {
        setupVBOandVAO();
    }
    else
    {
        setupVBO();
    }

    _dirty = true;

    return true;
}
```
`TextureAtlas`提供了对Quad的操作：
```c
void updateQuad(V3F_C4B_T2F_Quad* quad, ssize_t index);
//下面需要调用memmove移动内存
void insertQuad(V3F_C4B_T2F_Quad* quad, ssize_t index);
void insertQuads(V3F_C4B_T2F_Quad* quads, ssize_t index, ssize_t amount);
void removeQuadAtIndex(ssize_t index);
void removeQuadsAtIndex(ssize_t index, ssize_t amount);
//将_totalQuads置0
void removeAllQuads();
void increaseTotalQuadsWith(ssize_t amount);
void moveQuadsFromIndex(ssize_t oldIndex, ssize_t amount, ssize_t newIndex);
void moveQuadsFromIndex(ssize_t index, ssize_t newIndex);
void fillWithEmptyQuadsFromIndex(ssize_t index, ssize_t amount);
```
`TextureAtlas`有自己的绘制函数，在`BatchCommand`中维持一个`TextureAtlas*`，在其execute函数中
```c
void BatchCommand::execute()
{
    // Set material
    _shader->use();
    _shader->setUniformsForBuiltins(_mv);
    GL::bindTexture2D(_textureID);
    GL::blendFunc(_blendType.src, _blendType.dst);

    // Draw
    _textureAtlas->drawQuads();
}
```
我们看到了调用了`TextureAtlas::drawQuads`
```c
void TextureAtlas::drawQuads()
{
    this->drawNumberOfQuads(_totalQuads, 0);
}

void TextureAtlas::drawNumberOfQuads(ssize_t numberOfQuads)
{
    CCASSERT(numberOfQuads>=0, "numberOfQuads must be >= 0");
    this->drawNumberOfQuads(numberOfQuads, 0);
}

void TextureAtlas::drawNumberOfQuads(ssize_t numberOfQuads, ssize_t start)
{
    CCASSERT(numberOfQuads>=0 && start>=0, "numberOfQuads and start must be >= 0");

    if(!numberOfQuads)
        return;

    GL::bindTexture2D(_texture->getName());

    if (Configuration::getInstance()->supportsShareableVAO())
    {
        //
        // Using VBO and VAO
        //

        // XXX: update is done in draw... perhaps it should be done in a timer
        if (_dirty)
        {
            glBindBuffer(GL_ARRAY_BUFFER, _buffersVBO[0]);
            // option 1: subdata
//            glBufferSubData(GL_ARRAY_BUFFER, sizeof(_quads[0])*start, sizeof(_quads[0]) * n , &_quads[start] );

            // option 2: data
//            glBufferData(GL_ARRAY_BUFFER, sizeof(quads_[0]) * (n-start), &quads_[start], GL_DYNAMIC_DRAW);

            // option 3: orphaning + glMapBuffer
            glBufferData(GL_ARRAY_BUFFER, sizeof(_quads[0]) * (numberOfQuads-start), nullptr, GL_DYNAMIC_DRAW);
            void *buf = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
            memcpy(buf, _quads, sizeof(_quads[0])* (numberOfQuads-start));
            glUnmapBuffer(GL_ARRAY_BUFFER);

            glBindBuffer(GL_ARRAY_BUFFER, 0);

            _dirty = false;
        }

        GL::bindVAO(_VAOname);

#if CC_REBIND_INDICES_BUFFER
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, _buffersVBO[1]);
#endif

        glDrawElements(GL_TRIANGLES, (GLsizei) numberOfQuads*6, GL_UNSIGNED_SHORT, (GLvoid*) (start*6*sizeof(_indices[0])) );

#if CC_REBIND_INDICES_BUFFER
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
#endif

//    glBindVertexArray(0);
    }
    else
    {
        //
        // Using VBO without VAO
        //

#define kQuadSize sizeof(_quads[0].bl)
        glBindBuffer(GL_ARRAY_BUFFER, _buffersVBO[0]);

        // XXX: update is done in draw... perhaps it should be done in a timer
        if (_dirty)
        {
            glBufferSubData(GL_ARRAY_BUFFER, sizeof(_quads[0])*start, sizeof(_quads[0]) * numberOfQuads , &_quads[start] );
            _dirty = false;
        }

        GL::enableVertexAttribs(GL::VERTEX_ATTRIB_FLAG_POS_COLOR_TEX);

        // vertices
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_POSITION, 3, GL_FLOAT, GL_FALSE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, vertices));

        // colors
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_COLOR, 4, GL_UNSIGNED_BYTE, GL_TRUE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, colors));

        // tex coords
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_TEX_COORD, 2, GL_FLOAT, GL_FALSE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, texCoords));

        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, _buffersVBO[1]);

        glDrawElements(GL_TRIANGLES, (GLsizei)numberOfQuads*6, GL_UNSIGNED_SHORT, (GLvoid*) (start*6*sizeof(_indices[0])));

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
    }
    //添加Render中的计数
    CC_INCREMENT_GL_DRAWN_BATCHES_AND_VERTICES(1,numberOfQuads*6);

    CHECK_GL_ERROR_DEBUG();
}
```

##例子
###1. Sprite

###2. SpriteBatchNode
###3. 粒子Paticle
