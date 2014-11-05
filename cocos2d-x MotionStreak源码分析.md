# cocos2d-x MotionStreak 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-11-05 10:25  -->

MotionStreak继承于node，是用于实现将一个纹理拉成条带，并实现运动残影。移动的时候才会绘制（通过_startingPositionInitialized来控制）。

要理解MotionStreak，关键代码在于`void MotionStreak::update(float delta)`和`void MotionStreak::onDraw(const Mat4 &transform, uint32_t flags)`

首先看看它的成员属性：
```c
//用于函数`ccVertexLineToPolygon`，当fastMode为true时，只对增量的点进行扩展成为多边形，当为false时，每一次对所有顶点进行扩展计算
bool _fastMode;
bool _startingPositionInitialized;

/** texture used for the motion streak */
Texture2D* _texture; //纹理
BlendFunc _blendFunc; //混合
Vec2 _positionR; //当前node的位置（最前端的位置）

float _stroke;    //   条带的宽度
float _fadeDelta; //   1.0f/fade --  用户update函数，保证fade seconds后，顶点消失
float _minSeg;    //当移动的距离大于minSeg时，就产生一个新的顶点

//(int)(fade*60.0f)+2;  最多允许有这么多点，因为一秒最大60帧，在渐变的时间内，最多能update出60*fade个新顶点
unsigned int _maxPoints;
//实际update出来的顶点，初始化为0
unsigned int _nuPoints;
//用来判断是否有新添加的顶点，用在更新纹理坐标
unsigned int _previousNuPoints;

/** Pointers */
Vec2* _pointVertexes;  //用户保存条带的中心点 共_maxPoints个点
float* _pointState; //记录对应顶点的透明度，0-1之间，初始化为1，消失时为0

// Opengl
Vec2* _vertices;  //和_pointVertexes相关，用户openGL绘制
GLubyte* _colorPointer; //顶点颜色
Tex2F* _texCoords; //顶点纹理坐标

CustomCommand _customCommand;

```

update方法：添加新的点
```c
void MotionStreak::update(float delta)
{
    if (!_startingPositionInitialized) //如果没有移动就直接返回
    {
        return;
    }

    delta *= _fadeDelta;

    unsigned int newIdx, newIdx2, i, i2;
    unsigned int mov = 0; //记录需要消失的点的个数

    // Update current points
    for(i = 0; i<_nuPoints; i++)
    {
        _pointState[i]-=delta; //保证在fade seconds时间后消失

        if(_pointState[i] <= 0)
            mov++;
        else
        {
            newIdx = i-mov;

            if(mov>0)
            {
                // Move data
                _pointState[newIdx] = _pointState[i];

                // Move point
                _pointVertexes[newIdx] = _pointVertexes[i];

                // Move vertices
                i2 = i*2;
                newIdx2 = newIdx*2;
                _vertices[newIdx2] = _vertices[i2];
                _vertices[newIdx2+1] = _vertices[i2+1];

                // Move color（两个点的rgb,共6个数）
                // 不移动alpha值，因为此时alpha值相对递减
                i2 *= 4;
                newIdx2 *= 4;
                _colorPointer[newIdx2+0] = _colorPointer[i2+0];
                _colorPointer[newIdx2+1] = _colorPointer[i2+1];
                _colorPointer[newIdx2+2] = _colorPointer[i2+2];
                _colorPointer[newIdx2+4] = _colorPointer[i2+4];
                _colorPointer[newIdx2+5] = _colorPointer[i2+5];
                _colorPointer[newIdx2+6] = _colorPointer[i2+6];
            }else
              //递减alpha值
                newIdx2 = newIdx*8;

            const GLubyte op = (GLubyte)(_pointState[newIdx] * 255.0f);
            _colorPointer[newIdx2+3] = op;
            _colorPointer[newIdx2+7] = op;
        }
    }
    _nuPoints-=mov; //将消失的点从当前顶点数中删除

    // Append new point
    bool appendNewPoint = true;
    if(_nuPoints >= _maxPoints) //不能超过最大值，否则数组越界
    {
        appendNewPoint = false;
    }

    else if(_nuPoints>0)
    {
        bool a1 = _pointVertexes[_nuPoints-1].getDistanceSq(_positionR) < _minSeg; //要超过最大间隔距离minSeg，才增加新的顶点
        bool a2 = (_nuPoints == 1) ? false : (_pointVertexes[_nuPoints-2].getDistanceSq(_positionR)< (_minSeg * 2.0f)); //不懂
        if(a1 || a2)
        {
            appendNewPoint = false;
        }
    }

    if(appendNewPoint)
    {
        //新的顶点
        _pointVertexes[_nuPoints] = _positionR;
        _pointState[_nuPoints] = 1.0f;

        // 初始化新的颜色
        const unsigned int offset = _nuPoints*8;
        *((Color3B*)(_colorPointer + offset)) = _displayedColor;
        *((Color3B*)(_colorPointer + offset+4)) = _displayedColor;

        // 初始化颜色的alpha值
        _colorPointer[offset+3] = 255;
        _colorPointer[offset+7] = 255;

        // 根据_pointVertexes生成四边形，实际上是三角形带
        if(_nuPoints > 0 && _fastMode )
        {
            if(_nuPoints > 1)
            {
                /*该函数将线段扩展成多边形，扩展的大小为0.5*_stroke*/
                ccVertexLineToPolygon(_pointVertexes, _stroke, _vertices, _nuPoints, 1); //这就是fastMode模式，最后两个参数传法不一样
            }
            else
            {
                ccVertexLineToPolygon(_pointVertexes, _stroke, _vertices, 0, 2);
            }
        }

        _nuPoints ++;
    }

    if( ! _fastMode )
    {
        //所有顶点全部重新计算扩展多边形
        ccVertexLineToPolygon(_pointVertexes, _stroke, _vertices, 0, _nuPoints);
    }

    // 如果有新添加的顶点，那么更新所有的顶点的纹理坐标
    if( _nuPoints  && _previousNuPoints != _nuPoints ) {
      //顶点坐标线性划分
        float texDelta = 1.0f / _nuPoints;
        for( i=0; i < _nuPoints; i++ ) {
            _texCoords[i*2] = Tex2F(0, texDelta*i);
            _texCoords[i*2+1] = Tex2F(1, texDelta*i);
        }

        _previousNuPoints = _nuPoints;
    }
}
```

onDraw函数不再分析了，就是将上述获得的三角形条带（包括顶点坐标、颜色、纹理坐标）发送到renderer
进行渲染。
