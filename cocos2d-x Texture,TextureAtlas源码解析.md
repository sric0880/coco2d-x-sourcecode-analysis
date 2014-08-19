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
###1. Sprite & SpriteFrameCache
在init函数中会调用TextureCache::addImage来创建Texture，如果已经创建好了直接从cache中取出。
```c
bool Sprite::initWithFile(const std::string& filename)
{
    CCASSERT(filename.size()>0, "Invalid filename for sprite");

    Texture2D *texture = Director::getInstance()->getTextureCache()->addImage(filename);
    if (texture)
    {
        Rect rect = Rect::ZERO;
        rect.size = texture->getContentSize();
        return initWithTexture(texture, rect);
    }
    return false;
}
bool Sprite::initWithFile(const std::string &filename, const Rect& rect)
{
    CCASSERT(filename.size()>0, "Invalid filename");

    Texture2D *texture = Director::getInstance()->getTextureCache()->addImage(filename);
    if (texture)
    {
        return initWithTexture(texture, rect);
    }
    return false;
}
```
Sprite拥有比较特殊的是`SpriteFrame`，可以把它看做纹理的一小部分。`SpriteFrame`受到`SpriteFrameCache`的管理，就像`Texture2D`受到`TextureCache`的管理一样。
获得或者创建Spriteframe都是通过`SpriteFrameCache`。

`SpriteFrameCache`以Frame的name作为key，所有不同的图片千万不能重名，如果重名，可能会将已有的frame替换掉，程序会出现意想不到的效果。
这里说的图片不是纹理，这里的图片通过打包程序生成一张大图，这张大图可以称为纹理，就是说，一张纹理上可以有很多小图片，每一个小图片其实就对应
一个`SpriteFrame`。

`SpriteFrameCache`有通过plist创建`SpriteFrame`的函数，纹理图片的名称可以作为参数传入，也可以通过plist传入。
`SpriteFrameCache`中也是通过函数`TextureCache::addImage`创建纹理并传入`SpriteFrame`中的。

如果纹理和SpriteFrame已经通过`SpriteFrameCache`加载到内存了，这个时候创建Sprite就简单多了，只需要调用函数
```c
bool Sprite::initWithSpriteFrameName(const std::string& spriteFrameName)
```
这也是用的最多的函数，因为游戏开发都会考虑将小图片打包成大图片，以减少纹理加载的时间，以及渲染的效率，打包程序一般会生成一个plist文件，用于记录spriteframe长宽坐标等信息。

###2. SpriteBatchNode
将多个node的绘制放到一个glDraw*命令里面去，提高了绘制效率，前提是这些node都使用同一个`Texture2D`。

果然`SpriteBatchNode`用到了`TextureAtlas`，还使用了BatchCommand发送给Render进行绘制。下面是`SpriteBatchNode`的成员变量
```c
TextureAtlas *_textureAtlas;
BlendFunc _blendFunc;
BatchCommand _batchCommand;     // render command

// all descendants: children, grand children, etc...
// There is not need to retain/release these objects, since they are already retained by _children
// So, using std::vector<Sprite*> is slightly faster than using cocos2d::Array for this particular case
std::vector<Sprite*> _descendants;
```
`SpriteBatchNode`的init函数参数和`TextureAtlas`一样
```c
bool SpriteBatchNode::initWithTexture(Texture2D *tex, ssize_t capacity)
{
    CCASSERT(capacity>=0, "Capacity must be >= 0");

    _blendFunc = BlendFunc::ALPHA_PREMULTIPLIED;
    if(tex->hasPremultipliedAlpha())//如果纹理本身已经事先乘上了alpha值
    {
        _blendFunc = BlendFunc::ALPHA_NON_PREMULTIPLIED;//src color不需要在乘alpha值了
    }
    _textureAtlas = new TextureAtlas();

    if (capacity == 0)
    {
        capacity = DEFAULT_CAPACITY; //为什么是29个
    }

    _textureAtlas->initWithTexture(tex, capacity);//初始化

    updateBlendFunc();

    _children.reserve(capacity);//给capacity个node预备空间

    _descendants.reserve(capacity);
    //设置所使用的Program
    setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR));
    return true;
}
```
`SpriteBatchNode`重写了addChild函数，因为它只支持Sprites作为它的子节点，它还需要自己管理子节点，比如将节点加入到_descendants中去，
并将Sprite中的Quad取出加入到_textureAtlas中去。它还递归的将子节点的儿子节点加入到了SpriteBatchNode中。

*将Sprite中的Quad复制一份加入到_textureAtlas，会导致如果Sprite中的Quad变了（颜色或位置改变了），将无法同步到_textureAtlas中去，那么绘制
_textureAtlas将看不到这些变化。*

***解决方案：<u>Sprite维护了一个索引_atlasIndex,它指向_textureAtlas中的Quads的位置，在Sprite
的update函数中，都有调用`_textureAtlas->updateQuad(&_quad, _atlasIndex);`来保证_textureAtlas中是最新的Quad。</u>***

`SpriteBatchNode`重写了`visit`函数，因为它只visit自己，没有调用儿子节点的visit函数。

同样，它重写了`draw`函数：
```c
void SpriteBatchNode::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    // Optimization: Fast Dispatch
    if( _textureAtlas->getTotalQuads() == 0 )
    {
        return;
    }

    for(const auto &child: _children)
        child->updateTransform();
        //发送BatchCommand给Render
        //BatchCommand在执行绘制时，会调用_textureAtlas的draw*函数。
        //关于BatchCommand可以见渲染机制详解
    _batchCommand.init(
                       _globalZOrder,
                       getGLProgram(), //从glProgramState中取出GLProgram
                       _blendFunc,
                       _textureAtlas, //
                       transform);
    renderer->addCommand(&_batchCommand);
}
```
`SpriteBatchNode`其他函数主要用来增删查改Quad并动态分配capacity。这里不再详解。

###3. ParticleSystem
粒子系统详解请参见`cocos2d-x ParticleSystem源码解析`
