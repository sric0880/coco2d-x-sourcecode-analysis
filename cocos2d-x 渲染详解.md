# cocos2d-x 渲染详解

**version cocos2d-x 3.2**
<!-- create time: 2014-08-18 13:03  -->

渲染大致分为两步，先调用visit函数，visit函数中会调用draw函数，当然3.0版本后，draw函数不在直接绘制，而是将绘制命令发送到一个列队，再调用render函数执行列队中的命令。

###Visit
```c
void Node::visit()
{
    //全局就一个render
    auto renderer = Director::getInstance()->getRenderer();
    //栈顶上的modelview矩阵
    Mat4 parentTransform = Director::getInstance()->getMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    visit(renderer, parentTransform, true);//true --强制刷新modelview矩阵
}
//parentFlags：
//  true - 表示无论转换矩阵是否dirty,每次visit的时候强制重新计算modelview矩阵
//  false - 只有当转换矩阵标识为dirty的时候，才重新计算
void Node::visit(Renderer* renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    // quick return if not visible. children won't be drawn.
    if (!_visible)
    {
        return;
    }
    //在parent的基础上，乘上自身node的变换矩阵得到新的_modelViewTransform
    uint32_t flags = processParentFlags(parentTransform, parentFlags);

    // IMPORTANT:
    // To ease the migration to v3.0, we still support the Mat4 stack,
    // but it is deprecated and your code should not rely on it
    Director* director = Director::getInstance();
    //首先复制parent的变换矩阵并压栈
    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    //然后用最新的_modelViewTransform更新栈顶
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);

    int i = 0;
    //深度遍历儿子节点
    if(!_children.empty())
    {
        sortAllChildren();//按local Z排序，如果local Z相等，按照加入的顺序排序
        // draw children zOrder < 0
        for( ; i < _children.size(); i++ )
        {
            auto node = _children.at(i);
            //先绘制local Z小于0的节点，用于遮挡
            if ( node && node->_localZOrder < 0 )
                node->visit(renderer, _modelViewTransform, flags);
            else
                break;
        }
        // 绘制自己
        this->draw(renderer, _modelViewTransform, flags);
        // 绘制接下来local z > 0的节点
        for(auto it=_children.cbegin()+i; it != _children.cend(); ++it)
            (*it)->visit(renderer, _modelViewTransform, flags);
    }
    else
    { //没有儿子节点就直接绘制自己
        this->draw(renderer, _modelViewTransform, flags);
    }
    //弹出modelview矩阵
    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
}

void Node::draw()
{
    auto renderer = Director::getInstance()->getRenderer();
    draw(renderer, _modelViewTransform, true);
}
//什么都没做，因为它没有需要绘制的内容，需要子类继承，并重写，比如像sprite
void Node::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
}
```
接下来我们来看看Sprite::draw
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
```
这里使用了QuadCommand和CustomCommand两种绘制命令

要想理解上面的代码，首先需要了解Render类

###Render
关于Render初始化设置，创建投影矩阵、创建VAO和VBO等在Director类详解中已经分析过了。

Render类主要的成员：
```c
//存储command id,第一个是0，其他用于groupCommand,
//void Renderer::pushGroup(int renderQueueID)会压入一个group id
std::stack<int> _commandGroupStack;

    std::vector<RenderQueue> _renderGroups;//维持多个渲染列队

    uint32_t _lastMaterialID;
    //记录最后一次调用的MeshCommand，用于优化MeshCommand的渲染
    MeshCommand*              _lastBatchedMeshCommand;
    //用于将QuadCommand收集起来，等到4边形个数_numQuads大于VBO_SIZE时再统一绘制
    std::vector<QuadCommand*> _batchedQuadCommands;

    V3F_C4B_T2F_Quad _quads[VBO_SIZE];//这就是所有缓存的四边形
    GLushort _indices[6 * VBO_SIZE]; //索引数组，发送到显存后其实可以删除了
    GLuint _quadVAO;
    GLuint _buffersVBO[2]; //0: vertex  1: indices

    int _numQuads; //一次渲染中，有多少四边形

    bool _glViewAssigned;

    // stats
    ssize_t _drawnBatches;//记录调用glDrawElements的次数
    ssize_t _drawnVertices; //记录参与绘制的顶点个数
    //the flag for checking whether renderer is rendering
    bool _isRendering;

    GroupCommandManager* _groupCommandManager; //管理group command 的id的生成
```

```c
//往栈顶的id添加command
void Renderer::addCommand(RenderCommand* command)
{
    int renderQueue =_commandGroupStack.top();
    addCommand(command, renderQueue);
}
void Renderer::addCommand(RenderCommand* command, int renderQueue)
{
    CCASSERT(!_isRendering, "Cannot add command while rendering");
    CCASSERT(renderQueue >=0, "Invalid render queue");
    CCASSERT(command->getType() != RenderCommand::Type::UNKNOWN_COMMAND, "Invalid Command Type");
    _renderGroups[renderQueue].push_back(command);
}
```

function render：
```c
void Renderer::render()
{
    _isRendering = true;

    if (_glViewAssigned)
    {
        // cleanup
        _drawnBatches = _drawnVertices = 0;

        //Process render commands
        //1. Sort render commands based on ID
        for (auto &renderqueue : _renderGroups)
        {
            renderqueue.sort();
        }
        //只需要渲染索引为0的渲染列队
        //调用渲染列队中的command
        visitRenderQueue(_renderGroups[0]);
        flush();
    }
    clean();
    _isRendering = false;
}

void Renderer::visitRenderQueue(const RenderQueue& queue)
{
    ssize_t size = queue.size();

    for (ssize_t index = 0; index < size; ++index)
    {
        auto command = queue[index];
        auto commandType = command->getType();
        if(RenderCommand::Type::QUAD_COMMAND == commandType)
        {//绘制4边形
            flush3D();
            auto cmd = static_cast<QuadCommand*>(command);
            //Batch quads
            if(_numQuads + cmd->getQuadCount() > VBO_SIZE)
            {
                CCASSERT(cmd->getQuadCount()>= 0 && cmd->getQuadCount() < VBO_SIZE, "VBO is not big enough for quad data, please break the quad data down or use customized render command");

                //Draw batched quads if VBO is full
                drawBatchedQuads();
            }

            _batchedQuadCommands.push_back(cmd);
            //卧槽，内存中维持了两份相同的顶点数据，我勒个去?? 然后显存中也还有，太刁了
            memcpy(_quads + _numQuads, cmd->getQuads(), sizeof(V3F_C4B_T2F_Quad) * cmd->getQuadCount());
            //在这里对顶点进行矩阵变换，而不是在vertex shader中???
            convertToWorldCoordinates(_quads + _numQuads, cmd->getQuadCount(), cmd->getModelView());

            _numQuads += cmd->getQuadCount();

        }
        else if(RenderCommand::Type::GROUP_COMMAND == commandType)
        {//绘制group command，获得group的id作为索引，递归调用visitRenderQueue
          //后面会详细分析group command
            flush();
            int renderQueueID = ((GroupCommand*) command)->getRenderQueueID();
            visitRenderQueue(_renderGroups[renderQueueID]);
        }
        else if(RenderCommand::Type::CUSTOM_COMMAND == commandType)
        {//自定义command，直接调用回调函数
            flush();
            auto cmd = static_cast<CustomCommand*>(command);
            cmd->execute();
        }
        else if(RenderCommand::Type::BATCH_COMMAND == commandType)
        {//批处理，像batchNode这种,个人觉得可以和QuadCommand合并
            flush();
            auto cmd = static_cast<BatchCommand*>(command);
            cmd->execute();
        }
        else if (RenderCommand::Type::MESH_COMMAND == commandType)
        {//绘制3D Sprite使用的网格绘制
            flush2D();
            auto cmd = static_cast<MeshCommand*>(command);
            if (_lastBatchedMeshCommand == nullptr || _lastBatchedMeshCommand->getMaterialID() != cmd->getMaterialID())
            {
                flush3D();
                //第一次，或者MaterialID和上次不一样，就会调用
                //其他情况下不需要调用, 可以共用一个VAO
                cmd->preBatchDraw();
                cmd->batchDraw();
                _lastBatchedMeshCommand = cmd;
            }
            else
            {
                cmd->batchDraw();
            }
        }
        else
        {
            CCLOGERROR("Unknown commands in renderQueue");
        }
    }
}
```

下面分别介绍上面几种command：

各种command都继承于RenderCommand，RenderCommand包括两个属性
```c
Type _type;
// commands are sort by depth
float _globalOrder;
```

1. ####QUAD_COMMAND
`QuadCommand`主要有如下属性：
```c
uint32_t _materialID; //通过glProgram, (int)_textureID, (int)_blendType.src, (int)_blendType.dst哈希得到
GLuint _textureID;
GLProgramState* _glProgramState;
BlendFunc _blendType;
V3F_C4B_T2F_Quad* _quads;
ssize_t _quadsCount;
Mat4 _mv;
```
绘制四边形关键函数在:
```c
void Renderer::drawBatchedQuads()
{
    int quadsToDraw = 0;
    int startQuad = 0;

    if(_numQuads <= 0 || _batchedQuadCommands.empty())
    {
        return;
    }

    if (Configuration::getInstance()->supportsShareableVAO())
    {
        //Set VBO data
        glBindBuffer(GL_ARRAY_BUFFER, _buffersVBO[0]);

        // orphaning + glMapBuffer
        // 将顶点数据输入到显存中去
        glBufferData(GL_ARRAY_BUFFER, sizeof(_quads[0]) * (_numQuads), nullptr, GL_DYNAMIC_DRAW);
        void *buf = glMapBuffer(GL_ARRAY_BUFFER, GL_WRITE_ONLY);
        memcpy(buf, _quads, sizeof(_quads[0])* (_numQuads));
        glUnmapBuffer(GL_ARRAY_BUFFER);

        glBindBuffer(GL_ARRAY_BUFFER, 0);

        //Bind VAO
        GL::bindVAO(_quadVAO);
    }
    else
    {
#define kQuadSize sizeof(_quads[0].bl)
        glBindBuffer(GL_ARRAY_BUFFER, _buffersVBO[0]);

        glBufferData(GL_ARRAY_BUFFER, sizeof(_quads[0]) * _numQuads , _quads, GL_DYNAMIC_DRAW);

        GL::enableVertexAttribs(GL::VERTEX_ATTRIB_FLAG_POS_COLOR_TEX);

        // vertices
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_POSITION, 3, GL_FLOAT, GL_FALSE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, vertices));

        // colors
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_COLOR, 4, GL_UNSIGNED_BYTE, GL_TRUE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, colors));

        // tex coords
        glVertexAttribPointer(GLProgram::VERTEX_ATTRIB_TEX_COORD, 2, GL_FLOAT, GL_FALSE, kQuadSize, (GLvoid*) offsetof(V3F_C4B_T2F, texCoords));

        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, _buffersVBO[1]);
    }

    //Start drawing verties in batch
    for(const auto& cmd : _batchedQuadCommands)
    {
        auto newMaterialID = cmd->getMaterialID();
        //直到遇到一个MaterialID不同的command,就开始渲染
        //这里将MaterialID相同的command合并，因为所用的纹理、shader、blendfunc都一样
        //就是顶点不同，将这些顶点统一加入到quadsToDraw中，然后一次性绘制
        if(_lastMaterialID != newMaterialID || newMaterialID == QuadCommand::MATERIAL_ID_DO_NOT_BATCH)
        {
            //Draw quads
            if(quadsToDraw > 0)
            {   //开始绘制四边形，startQuad记录绘制四边形的起始位置
                glDrawElements(GL_TRIANGLES, (GLsizei) quadsToDraw*6, GL_UNSIGNED_SHORT, (GLvoid*) (startQuad*6*sizeof(_indices[0])) );
                _drawnBatches++;
                _drawnVertices += quadsToDraw*6;

                startQuad += quadsToDraw;
                quadsToDraw = 0;
            }

            //Use new material
            cmd->useMaterial();
            _lastMaterialID = newMaterialID;
        }

        quadsToDraw += cmd->getQuadCount();
    }

    //还剩下最后一次绘制
    if(quadsToDraw > 0)
    {
        glDrawElements(GL_TRIANGLES, (GLsizei) quadsToDraw*6, GL_UNSIGNED_SHORT, (GLvoid*) (startQuad*6*sizeof(_indices[0])) );
        _drawnBatches++;
        _drawnVertices += quadsToDraw*6;
    }

    if (Configuration::getInstance()->supportsShareableVAO())
    {
        //Unbind VAO
        GL::bindVAO(0);
    }
    else
    {
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);
    }

    _batchedQuadCommands.clear();
    _numQuads = 0;
}
```
该函数实现的一大特点就是auto-batch：可能将多次绘制命令合并成一个绘制命令，通过将连续的具有相同的MaterialID的command进行合并，减少了绘制的调用次数，提高了绘制的效率。

2. ####CUSTOM_COMMAND
`CustomCommand`就多了一个回调函数
```c
std::function<void()> func;
```

3. ####BATCH_COMMAND
`BatchCommand`在cocos2d-x中的应用有两个地方：ParticleBatchNode和SpriteBatchNode。  
`BatchCommand`包含如下属性：
```c
int32_t _materialID;
    GLuint _textureID; //_textureAtlas中的id
    GLProgram* _shader; //和QuadCommand不一样??
    BlendFunc _blendType;

    TextureAtlas *_textureAtlas;

    // ModelView transform
    Mat4 _mv;
```
它还有一个execute函数
```c
void BatchCommand::execute()
{
    // Set material
    _shader->use(); //使用shader
    _shader->setUniformsForBuiltins(_mv);//关于shader的用法有专门介绍
    GL::bindTexture2D(_textureID);
    GL::blendFunc(_blendType.src, _blendType.dst);

    // Draw
    //这里调用绘制命令,关于textureAtlas会在纹理篇中专门讲解
    _textureAtlas->drawQuads();
}
```
在3.0之后的版本中，由于添加了auto-batch功能，ParticleBatchNode和SpriteBatchNode的节约效率的功能已经不那么明显，但是3.0之前的版本中，把精灵放在SpriteBatchNode父节点上和将粒子系统放在ParticleBatchNode，是能够把相同的精灵批处理，对于相同的贴图只调用一次绘制函数，还是对提升效率很有帮助的。

4. ####GROUP_COMMAND
`GroupCommand`只有一个属性
```c
int _renderQueueID; //作为_renderGroups的索引
```
就是创建一个group command能够在这个group中加入多个command，这些group中的command并不是加入到默认的渲染列队中，而是加入到了一个新的渲染列队中，统一管理。目测用处不大吧。
这里介绍一下使用方法吧
```c
GroupCommand _spriteWrapperCommand;
...
_spriteWrapperCommand.init(_globalZOrder); //调用初始化函数，会自动给它分配一个queue id索引
renderer->addCommand(&_spriteWrapperCommand);//加入到默认渲染列队中(queue id == 0)
renderer->pushGroup(_spriteWrapperCommand.getRenderQueueID());//将新建一个渲染列队，并使用该列队
Sprite::draw(renderer, transform, flags);//该方法中调用的所有command将被加入到新创建的渲染列队中
renderer->popGroup();//重新使用默认渲染列队
```

5. ####MESH_COMMAND
`MeshCommand`用于Sprite3D，用于绘制3d模型
它具有的一些方法：
```c
//build & release vao
void buildVAO();
void releaseVAO();
// 开启一些3D的OpenGL设置，比如：
// 背面剔除、深度测试、深度缓冲区
void applyRenderState();
//还原上述开启的设置
void restoreRenderState();
//通过这些参数hash出一个MaterialID，和QuadCommand类似
void genMaterialID(GLuint texID, void* glProgramState, void* mesh, const BlendFunc& blend);
//准备需要渲染的数据
///只要是Command的material id相同，就没必要再调用该方法，因为数据已经加载到显存中去了，
///并且没有改变，其他Command可以直接用
void MeshCommand::preBatchDraw()
{
    applyRenderState();
    // Set material
    GL::bindTexture2D(_textureID);
    GL::blendFunc(_blendType.src, _blendType.dst);

    if (Configuration::getInstance()->supportsShareableVAO() && _vao == 0)
        buildVAO(); //创建VAO和VBO
    if (_vao)
    {
        GL::bindVAO(_vao);//绑定
    }
    else
    {
        glBindBuffer(GL_ARRAY_BUFFER, _vertexBuffer);
        _glProgramState->applyAttributes();
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, _indexBuffer);
    }
}
//貌似不再使用execute了，因为它没有使用VAO，并且每次都要bindBuffer
///
void MeshCommand::batchDraw()
{
    _glProgramState->setUniformVec4("u_color", _displayColor);

    if (_matrixPaletteSize && _matrixPalette)
    {
        _glProgramState->setUniformCallback("u_matrixPalette", CC_CALLBACK_2(MeshCommand::MatrixPalleteCallBack, this));
    }

    _glProgramState->applyGLProgram(_mv);
    _glProgramState->applyUniforms();

    // Draw
    glDrawElements(_primitive, (GLsizei)_indexCount, _indexFormat, 0);

    CC_INCREMENT_GL_DRAWN_BATCHES_AND_VERTICES(1, _indexCount);
}
```

>相关知识：  
`cocos2d-x Shader源码详解`
`cocos2d-x Texture纹理源码解析`
