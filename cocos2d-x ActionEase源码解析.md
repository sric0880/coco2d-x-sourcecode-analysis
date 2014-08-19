# cocos2d-x ActionEase 源码解析

**version cocos2d-x 3.2**
<!-- create time: 2014-08-12 17:01  -->

继承关系
Ease*--> ActionEase--> ActionInterval

EaseAction代理了一个ActionInterval。
**EaseAction和ActionInterval是继承又是代理关系**
```c++
bool initWithAction(ActionInterval *action);
...
/** The inner action */
ActionInterval *_inner;
...
```
这里将时间[0-1秒]通过tweenfunc函数映射到非线性时间。
```c++
void EaseIn::update(float time)
{
    _inner->update(tweenfunc::easeIn(time, _rate));
}
void EaseOut::update(float time)
{
    _inner->update(tweenfunc::easeOut(time, _rate));
}
void EaseExponentialOut::update(float time)
{
    _inner->update(tweenfunc::expoEaseOut(time));
}
```



>关于update原理参见`cocos2d-x Action动画 源码解析`

关键在于tweenfunc namespace,
在文件CCTweenFunction.h中：
```c++
namespace tweenfunc {
    enum TweenType
    {
        CUSTOM_EASING = -1,

        Linear,

        Sine_EaseIn,
        Sine_EaseOut,
        Sine_EaseInOut,


        Quad_EaseIn,
        Quad_EaseOut,
        Quad_EaseInOut,

        Cubic_EaseIn,
        Cubic_EaseOut,
        Cubic_EaseInOut,

        Quart_EaseIn,
        Quart_EaseOut,
        Quart_EaseInOut,

        Quint_EaseIn,
        Quint_EaseOut,
        Quint_EaseInOut,

        Expo_EaseIn,
        Expo_EaseOut,
        Expo_EaseInOut,

        Circ_EaseIn,
        Circ_EaseOut,
        Circ_EaseInOut,

        Elastic_EaseIn,
        Elastic_EaseOut,
        Elastic_EaseInOut,

        Back_EaseIn,
        Back_EaseOut,
        Back_EaseInOut,

        Bounce_EaseIn,
        Bounce_EaseOut,
        Bounce_EaseInOut,

        TWEEN_EASING_MAX = 10000
    };


    //tween functions for CCActionEase
    float easeIn(float time, float rate);
    float easeOut(float time, float rate);
    float easeInOut(float time, float rate);

    float bezieratFunction( float a, float b, float c, float d, float t );

    float quadraticIn(float time);
    float quadraticOut(float time);
    float quadraticInOut(float time);


    float tweenTo(float time, TweenType type, float *easingParam);

    float linear(float time);


    float sineEaseIn(float time);
    float sineEaseOut(float time);
    float sineEaseInOut(float time);

    float quadEaseIn(float time);
    float quadEaseOut(float time);
    float quadEaseInOut(float time);

    float cubicEaseIn(float time);
    float cubicEaseOut(float time);
    float cubicEaseInOut(float time);

    float quartEaseIn(float time);
    float quartEaseOut(float time);
    float quartEaseInOut(float time);

    float quintEaseIn(float time);
    float quintEaseOut(float time);
    float quintEaseInOut(float time);

    float expoEaseIn(float time);
    float expoEaseOut(float time);
    float expoEaseInOut(float time);

    float circEaseIn(float time);
    float circEaseOut(float time);
    float circEaseInOut(float time);

    float elasticEaseIn(float time, float period);
    float elasticEaseOut(float time, float period);
    float elasticEaseInOut(float time, float period);

    float backEaseIn(float time);
    float backEaseOut(float time);
    float backEaseInOut(float time);

    float bounceEaseIn(float time);
    float bounceEaseOut(float time);
    float bounceEaseInOut(float time);

    float customEase(float time, float *easingParam);
}
```

官方结构图：
![image](http://cn.cocos2d-x.org/doc/cocos2d-x-3.0/db/dcf/classcocos2d_1_1_action_ease.png)

关于映射函数这里不再详细说明，下图一清二楚：
![image](http://m3.img.papaapp.com/farm5/d/2013/1213/15/A1E5646B242F91BE6D5E4C24E950878A_LARGE_731_644.jpeg)
