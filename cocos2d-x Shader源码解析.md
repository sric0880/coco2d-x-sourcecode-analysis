# cocos2d-x Shader源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-18 20:46  -->

##GLProgram
GLProgram是对openGL的program进行了一次封装，提供了通过字符数组或文件创建program的方法，
包括compile、link等过程。
创建Shader的方法中，将shader脚本开头统一加上了cocos2d-x定义的uniform变量，如下所示：
```c
bool GLProgram::compileShader(GLuint * shader, GLenum type, const GLchar* source)
{
    GLint status;

    if (!source)
    {
        return false;
    }

    const GLchar *sources[] = {
#if (CC_TARGET_PLATFORM != CC_PLATFORM_WIN32 && CC_TARGET_PLATFORM != CC_PLATFORM_LINUX && CC_TARGET_PLATFORM != CC_PLATFORM_MAC)
        (type == GL_VERTEX_SHADER ? "precision highp float;\n" : "precision mediump float;\n"),
#endif
        "uniform mat4 CC_PMatrix;\n"
        "uniform mat4 CC_MVMatrix;\n"
        "uniform mat4 CC_MVPMatrix;\n"
        "uniform vec4 CC_Time;\n"
        "uniform vec4 CC_SinTime;\n"
        "uniform vec4 CC_CosTime;\n"
        "uniform vec4 CC_Random01;\n"
        "uniform sampler2D CC_Texture0;\n"
        "uniform sampler2D CC_Texture1;\n"
        "uniform sampler2D CC_Texture2;\n"
        "uniform sampler2D CC_Texture3;\n"
        "//CC INCLUDES END\n\n",
        source,
    };

    *shader = glCreateShader(type);
    glShaderSource(*shader, sizeof(sources)/sizeof(*sources), sources, nullptr);
    glCompileShader(*shader);

    //检查编译状态是否成功
    ...
    return (status == GL_TRUE);
}
```
初始化program的时候，会创建两个Shader-vertex Shader&fragment shader,
在init中将两个shader attach到program之后，需要主动调用link函数
```c
bool GLProgram::link()
{
    CCASSERT(_program != 0, "Cannot link invalid program");

#if (CC_TARGET_PLATFORM == CC_PLATFORM_WINRT) || (CC_TARGET_PLATFORM == CC_PLATFORM_WP8)
    if(!_hasShaderCompiler)
    {
        // precompiled shader program is already linked

        //bindPredefinedVertexAttribs();
        parseVertexAttribs();
        parseUniforms();
        return true;
    }
#endif

    GLint status = GL_TRUE;
    //主动去绑定attribute，分别将这些index绑定如下属性名到:
    //VERTEX_ATTRIB_POSITION - "a_color"
    //VERTEX_ATTRIB_COLOR - "a_position"
    //VERTEX_ATTRIB_TEX_COORD - "a_texCoord"
    //VERTEX_ATTRIB_NORMAL - "a_normal"
    bindPredefinedVertexAttribs();

    glLinkProgram(_program);//绑定之后才能linkProgram
    //将所有attributes从OpenGL中查询出来，存入
    //std::unordered_map<std::string, VertexAttrib> _vertexAttribs;
    //以属性名称作为key, VertexAttrib包含index,size,type,name
    parseVertexAttribs();
    //查询所有uniform变量，过滤掉CC_开头的内置变量
    //通过glGetUniformLocation查询到uniform变量的index,存入Uniform结构体
    //统一保存到std::unordered_map<std::string, Uniform> _userUniforms;
    parseUniforms();

    if (_vertShader)
    {
        glDeleteShader(_vertShader);//不会真正删除，只是标记为删除，当调用glDetachShader时才真在删除
    }

    if (_fragShader)
    {
        glDeleteShader(_fragShader);//不会真正删除，只是标记为删除，当调用glDetachShader时才真在删除
    }

    _vertShader = _fragShader = 0;

#if DEBUG || (CC_TARGET_PLATFORM == CC_PLATFORM_WINRT) || (CC_TARGET_PLATFORM == CC_PLATFORM_WP8)
    glGetProgramiv(_program, GL_LINK_STATUS, &status);

    if (status == GL_FALSE)
    {
        CCLOG("cocos2d: ERROR: Failed to link program: %i", _program);
        GL::deleteProgram(_program);
        _program = 0;
    }
#endif

#if (CC_TARGET_PLATFORM == CC_PLATFORM_WINRT) || (CC_TARGET_PLATFORM == CC_PLATFORM_WP8)
    if (status == GL_TRUE)
    {
        CCPrecompiledShaders::getInstance()->addProgram(_program, _shaderId);
    }
#endif

    return (status == GL_TRUE);
}
```
在link结束后，GLProgram接着执行了函数`updateUniforms`，用于获取CC_开头的uniform变量的location
```c
void GLProgram::updateUniforms()
{ //通过函数glGetUniformLocation获取uniform的location
    _builtInUniforms[UNIFORM_P_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_P_MATRIX);
    _builtInUniforms[UNIFORM_MV_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_MV_MATRIX);
    _builtInUniforms[UNIFORM_MVP_MATRIX] = glGetUniformLocation(_program, UNIFORM_NAME_MVP_MATRIX);

    _builtInUniforms[UNIFORM_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_TIME);
    _builtInUniforms[UNIFORM_SIN_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_SIN_TIME);
    _builtInUniforms[UNIFORM_COS_TIME] = glGetUniformLocation(_program, UNIFORM_NAME_COS_TIME);

    _builtInUniforms[UNIFORM_RANDOM01] = glGetUniformLocation(_program, UNIFORM_NAME_RANDOM01);

    _builtInUniforms[UNIFORM_SAMPLER0] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER0);
    _builtInUniforms[UNIFORM_SAMPLER1] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER1);
    _builtInUniforms[UNIFORM_SAMPLER2] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER2);
    _builtInUniforms[UNIFORM_SAMPLER3] = glGetUniformLocation(_program, UNIFORM_NAME_SAMPLER3);

    _flags.usesP = _builtInUniforms[UNIFORM_P_MATRIX] != -1;
    _flags.usesMV = _builtInUniforms[UNIFORM_MV_MATRIX] != -1;
    _flags.usesMVP = _builtInUniforms[UNIFORM_MVP_MATRIX] != -1;
    _flags.usesTime = (
                       _builtInUniforms[UNIFORM_TIME] != -1 ||
                       _builtInUniforms[UNIFORM_SIN_TIME] != -1 ||
                       _builtInUniforms[UNIFORM_COS_TIME] != -1
                       );
    _flags.usesRandom = _builtInUniforms[UNIFORM_RANDOM01] != -1;
    //一切都就绪了 调用use就行了
    this->use();

    // Since sample most probably won't change, set it to 0,1,2,3 now.
    //设置纹理采样器的值，一般都是从0开始的，所以这里就把值给设定死了
    if(_builtInUniforms[UNIFORM_SAMPLER0] != -1)
       setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER0], 0);
    if(_builtInUniforms[UNIFORM_SAMPLER1] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER1], 1);
    if(_builtInUniforms[UNIFORM_SAMPLER2] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER2], 2);
    if(_builtInUniforms[UNIFORM_SAMPLER3] != -1)
        setUniformLocationWith1i(_builtInUniforms[UNIFORM_SAMPLER3], 3);
}
void GLProgram::use()
{
    GL::useProgram(_program);
}
```
在每次渲染时，需要向vertex shader和fragment shader传值，关于attributes的值，使用
glVertexAttribPointer进行传递，详情参见`cocos2d-x 渲染详解`;关于uniform的值，使用
`setUniformLocationWith*`函数进行传递。

cocos2d-x竟然还做了这样的优化，传给uniform的值都会以location做key,建立hash表，缓存所有传入的数据，
如果对于同一个location，输入了相同的data数据，那么就不需要调用glUniform*函数传值了。
否则需要更新uniform的值。

还有一个比较重要的函数：
```c
void GLProgram::setUniformsForBuiltins()
{
    Director* director = Director::getInstance();
    CCASSERT(nullptr != director, "Director is null when seting matrix stack");

    Mat4 matrixMV;
    matrixMV = director->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    setUniformsForBuiltins(matrixMV);
}
//设置几个比较重要的变换矩阵到vertex shader中去
void GLProgram::setUniformsForBuiltins(const Mat4 &matrixMV)
{
    Mat4 matrixP = Director::getInstance()->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_PROJECTION);

    if(_flags.usesP)
        setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_P_MATRIX], matrixP.m, 1);

    if(_flags.usesMV)
        setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_MV_MATRIX], matrixMV.m, 1);

    if(_flags.usesMVP) {
        Mat4 matrixMVP = matrixP * matrixMV;
        setUniformLocationWithMatrix4fv(_builtInUniforms[UNIFORM_MVP_MATRIX], matrixMVP.m, 1);
    }

    if(_flags.usesTime) {
        Director *director = Director::getInstance();
        // This doesn't give the most accurate global time value.
        // Cocos2D doesn't store a high precision time value, so this will have to do.
        // Getting Mach time per frame per shader using time could be extremely expensive.
        float time = director->getTotalFrames() * director->getAnimationInterval();
        //从程序启动到目前所用时间的大概估计
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_TIME], time/10.0, time, time*2, time*4);
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_SIN_TIME], time/8.0, time/4.0, time/2.0, sinf(time));
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_COS_TIME], time/8.0, time/4.0, time/2.0, cosf(time));
    }

    if(_flags.usesRandom)
        setUniformLocationWith4f(_builtInUniforms[GLProgram::UNIFORM_RANDOM01], CCRANDOM_0_1(), CCRANDOM_0_1(), CCRANDOM_0_1(), CCRANDOM_0_1());
}
```
经过上面几个函数，所有关于program的初始化以及设置都完成了，内置变量uniform值也初始化好了
其他attributes和uniform的值的location也保存在
```c
std::unordered_map<std::string, Uniform> _userUniforms;
std::unordered_map<std::string, VertexAttrib> _vertexAttribs;
```
cocos2d-x定义的attributes的index(location)定义在
```c
enum
    {
        VERTEX_ATTRIB_POSITION,
        VERTEX_ATTRIB_COLOR,
        VERTEX_ATTRIB_TEX_COORD,
        VERTEX_ATTRIB_NORMAL,
        VERTEX_ATTRIB_BLEND_WEIGHT,
        VERTEX_ATTRIB_BLEND_INDEX,

        VERTEX_ATTRIB_MAX,

        // backward compatibility
        VERTEX_ATTRIB_TEX_COORDS = VERTEX_ATTRIB_TEX_COORD,
    };
```
cocos2d-x定义的uniform的location定义在
```c
GLint             _builtInUniforms[UNIFORM_MAX];
//索引使用
enum
    {
        UNIFORM_P_MATRIX,
        UNIFORM_MV_MATRIX,
        UNIFORM_MVP_MATRIX,
        UNIFORM_TIME,
        UNIFORM_SIN_TIME,
        UNIFORM_COS_TIME,
        UNIFORM_RANDOM01,
        UNIFORM_SAMPLER0,
        UNIFORM_SAMPLER1,
        UNIFORM_SAMPLER2,
        UNIFORM_SAMPLER3,

        UNIFORM_MAX,
    };
```
是不是非常简单的一个类，好了，接下来我们分析`GLProgramCache`

##GLProgramCache
是一个单例用来管理GLProgram对象的类，维持一个`std::unordered_map<std::string, GLProgram*> _programs;`, key为program的名称。
初始化init函数中调用了`loadDefaultGLPrograms`函数，该函数加载了很多预先定义好的program。下面我们来看看加载了哪些program。
```c
//在CCGLProgram.h中定义了很多shader的名称：
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR = "ShaderPositionTextureColor";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP = "ShaderPositionTextureColor_noMVP";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST = "ShaderPositionTextureColorAlphaTest";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST_NO_MV = "ShaderPositionTextureColorAlphaTest_NoMV";
const char* GLProgram::SHADER_NAME_POSITION_COLOR = "ShaderPositionColor";
const char* GLProgram::SHADER_NAME_POSITION_COLOR_NO_MVP = "ShaderPositionColor_noMVP";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE = "ShaderPositionTexture";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_U_COLOR = "ShaderPositionTexture_uColor";
const char* GLProgram::SHADER_NAME_POSITION_TEXTURE_A8_COLOR = "ShaderPositionTextureA8Color";
const char* GLProgram::SHADER_NAME_POSITION_U_COLOR = "ShaderPosition_uColor";
const char* GLProgram::SHADER_NAME_POSITION_LENGTH_TEXTURE_COLOR = "ShaderPositionLengthTextureColor";

const char* GLProgram::SHADER_NAME_LABEL_DISTANCEFIELD_NORMAL = "ShaderLabelDFNormal";
const char* GLProgram::SHADER_NAME_LABEL_DISTANCEFIELD_GLOW = "ShaderLabelDFGlow";
const char* GLProgram::SHADER_NAME_LABEL_NORMAL = "ShaderLabelNormal";
const char* GLProgram::SHADER_NAME_LABEL_OUTLINE = "ShaderLabelOutline";

const char* GLProgram::SHADER_3D_POSITION = "Shader3DPosition";
const char* GLProgram::SHADER_3D_POSITION_TEXTURE = "Shader3DPositionTexture";
const char* GLProgram::SHADER_3D_SKINPOSITION_TEXTURE = "Shader3DSkinPositionTexture";
```
具体创建在函数`void GLProgramCache::loadDefaultGLProgram(GLProgram *p, int type)`中。
使用`initWithByteArrays`通过传入字符数组创建。
举个例子：
```c
    case kShaderType_PositionTextureColor:
            p->initWithByteArrays(ccPositionTextureColor_vert, ccPositionTextureColor_frag);
            break;
...
p->link();
p->updateUniforms();
```
ccPositionTextureColor_vert和ccPositionTextureColor_frag分别就是vertex shader和fragment shader脚本了。

这些脚本统一在ccShaders.h中定义。

关键我们是要知道到底这些program都有什么用
1. SHADER_NAME_POSITION_TEXTURE_COLOR  
vertex：将position经过MVP转化，color,texcoord不做任何处理  
fragment：将纹理颜色和顶点插值颜色相乘

2. SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP  
vertex：将position经过Projection转化，color,texcoord不做任何处理
fragment：同1

3. SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST  
vertex：同1  
fragment：如果纹理颜色的alpha值小于设定的CC_alpha_value，那么就discard，否则同1

4. SHADER_NAME_POSITION_TEXTURE_ALPHA_TEST_NO_MV  
vertex：同2  
fragment：同3

5. SHADER_NAME_POSITION_COLOR  
vertex：没有纹理坐标，position通过MVP转换，color不做处理  
fragment：直接显示color的插值

6. SHADER_NAME_POSITION_COLOR_NO_MVP  
vertex：同2  
fragment：同5

7. SHADER_NAME_POSITION_TEXTURE  
vertex：没有颜色属性，position通过MVP转换，纹理坐标不做处理  
fragment：直接显示纹理采样的颜色

8. SHADER_NAME_POSITION_TEXTURE_U_COLOR  
vertex：同7  
fragment：定义了一个uniform - u_color(uniform color) ，和纹理颜色相乘

9. SHADER_NAME_POSITION_TEXTURE_A8_COLOR  
vertex：同1  
fragment：rgb取顶点颜色的插值的rgb，alpha取插值的a和纹理的a相乘的结果（很难想象这个效果）

10. SHADER_NAME_POSITION_U_COLOR  
vertex：position通过MVP转换，没有纹理坐标，固定uniform颜色值，固定gl_PointSize大小  
fragment：顶点颜色的插值

11. SHADER_NAME_POSITION_LENGTH_TEXTURE_COLOR  
vertex：position通过MVP转换，顶点颜色的rgb和alpha值相乘，纹理坐标不做处理  
fragment：如果纹理坐标的length>1, 就变成黑色，否则为颜色插值

12. SHADER_NAME_LABEL_NORMAL  
vertex：同1  
fragment：rgb同字体颜色，alpha为字体与纹理的alpha的乘积

13. SHADER_NAME_LABEL_OUTLINE
vertex：同1  
fragment：将描边颜色和字体颜色之间进行插值，作为像素颜色

14. SHADER_NAME_LABEL_DISTANCEFIELD_NORMAL  
vertex：同1  
fragment：颜色的rgb为uniform变量的值，alpha为uniform变量在[0-1]之间的插值

15. SHADER_NAME_LABEL_DISTANCEFIELD_GLOW  
vertex：同1  
fragment：有点复杂！！

16. SHADER_3D_POSITION  
vertex：position通过MVP转换，纹理坐标：texcoord.y = 1.0 - texcoord.y  
fragment：为uniform变量的颜色值

17. SHADER_3D_POSITION_TEXTURE  
vertex：同16  
fragment：为uniform变量的颜色值*纹理颜色

18. SHADER_3D_SKINPOSITION_TEXTURE  
vertex：顶点坐标经过了复杂的处理，纹理坐标同16  
fragment：同17

##GLProgramState(可选)
`GLProgramState`保存uniform和attributes的状态，一个`GLProgram`可以被很多Nodes使用，但是每一个node的uniform和attribute变量可以不一样，所以需要`GLProgramState`去
维护这个不同。

一个`GLProgramState`对象拥有一个`GLProgram`对象的指针，以及针对这个`GLProgram`的所有uniform和attributes的值。

`GLProgramState`提供一个根据`GLProgram`创建`GLProgramState`的方法：
```c
static GLProgramState* create(GLProgram* glprogram);

/** gets-or-creates an instance of GLProgramState for a given GLProgram */
static GLProgramState* getOrCreateWithGLProgram(GLProgram* glprogram);

/** gets-or-creates an instance of GLProgramState for a given GLProgramName */
static GLProgramState* getOrCreateWithGLProgramName(const std::string &glProgramName );
```
后面两个是去`GLProgramStateCache`中根据`glprogram`做为key去查找已有的`GLProgramState`，查到了就直接返回，否则创建。

该类有几个常用的函数：
```c
void GLProgramState::apply(const Mat4& modelView)
{
    applyGLProgram(modelView);

    applyAttributes();

    applyUniforms();
}
//程序已经在init的时候，初始化了_attributes, _uniforms, _uniformsByName三个map对象
//将_glprogram中的Uniform和VertexAttrib存入_attributes，_uniforms（没有具体的值）
void GLProgramState::applyGLProgram(const Mat4& modelView)
{
    CCASSERT(_glprogram, "invalid glprogram");
    if(_uniformAttributeValueDirty)
    {
        for(auto& uniformLocation : _uniformsByName)
        {
            _uniforms[uniformLocation.second]._uniform = _glprogram->getUniform(uniformLocation.first);
        }

        _vertexAttribsFlags = 0;
        for(auto& attributeValue : _attributes)
        {
            attributeValue.second._vertexAttrib = _glprogram->getVertexAttrib(attributeValue.first);;
            if(attributeValue.second._enabled) //默认是false
                _vertexAttribsFlags |= 1 << attributeValue.second._vertexAttrib->index;
        }

        _uniformAttributeValueDirty = false;

    }
    // set shader
    _glprogram->use();
    _glprogram->setUniformsForBuiltins(modelView);
}
//applyAttribFlags - 控制是否enableVertexAttribs
void GLProgramState::applyAttributes(bool applyAttribFlags)
{
    // Don't set attributes if they weren't set
    // Use Case: Auto-batching
    if(_vertexAttribsFlags) {//有attribute属性
        // enable/disable vertex attribs
        if (applyAttribFlags)
            GL::enableVertexAttribs(_vertexAttribsFlags);//开启属性
        // set attributes
        for(auto &attribute : _attributes)
        {
            attribute.second.apply();
        }
    }
}
void GLProgramState::applyUniforms()
{
    // set uniforms
    for(auto& uniform : _uniforms) {
        uniform.second.apply();
    }
}
```
每一个UniformValue和VertexAttribValue都有相应的apply方法：
```c
void VertexAttribValue::apply()
{
    if(_enabled) {
        if(_useCallback) {
            (*_value.callback)(_vertexAttrib); //回调
        }
        else
        { //直接调用glVertexAttribPointer函数
            glVertexAttribPointer(_vertexAttrib->index,
                                  _value.pointer.size,
                                  _value.pointer.type,
                                  _value.pointer.normalized,
                                  _value.pointer.stride,
                                  _value.pointer.pointer);
        }
    }
}
void UniformValue::apply()
{
    if(_useCallback) {
        (*_value.callback)(_glprogram, _uniform);//回调
    }
    else
    { //根据uniform的数据类型，调用相应的setUniform*函数
      //对于setUniformLocationWith{1，2，3，4}fv没办法了
        switch (_uniform->type) {
            case GL_SAMPLER_2D:
                _glprogram->setUniformLocationWith1i(_uniform->location, _value.tex.textureUnit);
                GL::bindTexture2DN(_value.tex.textureUnit, _value.tex.textureId);
                break;

            case GL_INT:
                _glprogram->setUniformLocationWith1i(_uniform->location, _value.intValue);
                break;

            case GL_FLOAT:
                _glprogram->setUniformLocationWith1f(_uniform->location, _value.floatValue);
                break;

            case GL_FLOAT_VEC2:
                _glprogram->setUniformLocationWith2f(_uniform->location, _value.v2Value[0], _value.v2Value[1]);
                break;

            case GL_FLOAT_VEC3:
                _glprogram->setUniformLocationWith3f(_uniform->location, _value.v3Value[0], _value.v3Value[1], _value.v3Value[2]);
                break;

            case GL_FLOAT_VEC4:
                _glprogram->setUniformLocationWith4f(_uniform->location, _value.v4Value[0], _value.v4Value[1], _value.v4Value[2], _value.v4Value[3]);
                break;

            case GL_FLOAT_MAT4:
                _glprogram->setUniformLocationWithMatrix4fv(_uniform->location, (GLfloat*)&_value.matrixValue, 1);
                break;

            default:
                CCASSERT(false, "Invalid UniformValue");
                break;
        }
    }
}
```
其他就是一些get set函数用来获得或改变uniform和attributes的值。
使用例子：`QuadCommand`中有一个`GLProgramState`对象指针，该指针是通过init函数传进来的，我们看到Sprite中有一个`QuadCommand`对象。在Sprite::draw函数中调用了QuadCommand
::init()。
```c
void Sprite::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    // Don't do calculate the culling if the transform was not updated
    _insideBounds = (flags & FLAGS_TRANSFORM_DIRTY) ? renderer->checkVisibility(transform, _contentSize) : _insideBounds;

    if(_insideBounds)
    {
        _quadCommand.init(_globalZOrder, _texture->getName(), getGLProgramState(), _blendFunc, &_quad, 1, transform);
        renderer->addCommand(&_quadCommand);
#if CC_SPRITE_DEBUG_DRAW
        _customDebugDrawCommand.init(_globalZOrder);
        _customDebugDrawCommand.func = CC_CALLBACK_0(Sprite::drawDebugData, this);
        renderer->addCommand(&_customDebugDrawCommand);
#endif //CC_SPRITE_DEBUG_DRAW
    }
}
GLProgramState* Node::getGLProgramState() const
{
    return _glProgramState;
}
```
我们看到sprite也有一个`GLProgramState*`,`_glProgramState`是通过如下函数设置的
```c
void Node::setGLProgramState(cocos2d::GLProgramState *glProgramState)
{
    if(glProgramState != _glProgramState) {
        CC_SAFE_RELEASE(_glProgramState);
        _glProgramState = glProgramState;
        CC_SAFE_RETAIN(_glProgramState);
    }
}

void Node::setGLProgram(GLProgram *glProgram)
{
    if (_glProgramState == nullptr || (_glProgramState && _glProgramState->getGLProgram() != glProgram))
    {
        CC_SAFE_RELEASE(_glProgramState);
        _glProgramState = GLProgramState::getOrCreateWithGLProgram(glProgram);
        _glProgramState->retain();
    }
}
```
sprite类在initWithTexture函数中调用了setGLProgramState
```c
// shader state
setGLProgramState(GLProgramState::getOrCreateWithGLProgramName(GLProgram::SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP));
```
原来Sprite用的Shader是`SHADER_NAME_POSITION_TEXTURE_COLOR_NO_MVP`类型。

其实，Sprite中的`GLProgramState`除了维护一个`GLProgram`之外，并没有用来存储attributes和uniform的值，因为Render类已经负责了QuadCommand的绘制了，因为Render对于QuadCommand
采用了auto-batching的技术(参见`cocos2d-x 渲染详解`)，虽然Render在渲染quad时调用了`GLProgramState::apply`，但是由于`GLProgramState::_vertexAttribsFlags==0`所以并没有
调用attribute.apply函数。

##GLProgramStateCache(可选)
一个`GLProgramState`的缓存，保存一个Map:
```c
Map<GLProgram*, GLProgramState*> _glProgramStates;
```
