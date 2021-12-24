# 前言

本编程指南仅尽可能的列出编写一个shader的参考

# 

# shader编写结构

## 1.定义Properties（方便在创建材质的时候可以传入贴图等数据）

## 2.编写SubShader主题

## 2.1【Tags】

## 2.2【RenderSetUp】

### 2.2.1【LOD】渲染深度（详情参考Unity LOD 远景渲染部分）

## 2.3【GrabPass】【UsePass】【Pass】【Pass】(第二个或者其他)

----------------------案例Pass--------------------
【RenderSetUp】
CGPROGRAM
//编译顶点着色器
#pragma vertex vert
//编译片元着色器
#pragma fragment frag

### 2.3.1 其他引用以及其它编译指令

### 2.3.2 声明参数

### 2.3.3 设置应用数据->顶点着色器结构体a2v，appdata（a2v为app to vert简写）

### 2.3.4 设置顶点->片源着色器结构体v2f（v2f为vert to fragment简写）

### 2.3.5 编写顶点着色器渲染方法

v2f vert (a2v v){
    v2f o;
    //着色器内容
    return o;
}

### 2.3.5 编写片源着色器渲染方法

fixed4 frag (v2f i) : SV_Target{
    fixed4 tex = ...;
    //着色器内容
    return tex;
}
ENDCG

---------------------------------------------------

## 3.设置Fallback，如果显卡无法运行上面的SubShader，则执行FallBack预设进行渲染

# 1.Properties（参考书籍P30）

--------------------------------------------------------------------------

### 传入数据常用类型

## Int

_Int("Int",Int)=2

## Float

_Float("Float",Float)=1.0

## Range

_Range("Range",Range(0.0,5.0))=1.0

## Color

_Color("Color",Color)=(1,1,1,1)

## Vector

_Vector("Vector",Vector)=(1,1,1,1)

## 2D

_MainTex ("Texture", 2D) = "white" {}
ps:内置纹理名称包含white、black、gray、bump等等

## Cube

_Cubemap ("Environment Cubemap", Cube) = "_Skybox" {}

## 3D

_3D ("3D", 3D) = "black" {}
--------------------------------------------------------------------------

# 2.Tags（参考网站：https://gameinstitute.qq.com/community/detail/121997）

--------------------------------------------------------------------------

### 渲染类型声明（包含渲染顺序）

Tags的书写方法为键值对，可多可少，不需要全部声明，未声明时按默认值
Tags { "RenderType"="Opaque" "Queue"="Transparent" }

# 2.1（SubShader）Tags

## key:Queue

渲染顺序,括号内为优先度，值越小越先被渲染

## value

Background(1000)
Geometry(2000)【默认】
AlphaTest(2450)
Transparent(3000)
Overlay(4000)
可以自定义值例如"Queue"="3100"

## key:RenderType

渲染类型

## value

Opaque-----------------不透明（法线、自发光、反射、地形Shader）
Transparent------------半透明（透明、粒子、字体、地形添加通道Shader）
TransparentCutout------遮罩透明（透明裁切、双通道植物Shader）
Background-------------天空盒Shader
Overlay----------------GUI纹理、光晕、闪光Shader
TreeOpaque-------------地形引擎——树皮
TreeTransparentCutout--地形引擎——树叶
TreeBillboard----------地形引擎——公告牌（始终面向摄像机）式树木
Grass------------------地形引擎——草
GrassBillboard---------地形引擎——公告牌（始终面向摄像机）式草

## key:DisableBatching

是否禁用Batch（打包、合并）

## value

True-------------------禁用
False------------------不禁用（默认）
LODFading--------------当LOD fade开启的时候禁用，一般用在树木上面

## key:ForceNoShadowCasting

是否强制不投射阴影，当这个值为True的时候，使用这个Shader的对象便不会投射阴影。

## key:IgnoreProjector

无视投影器，当这个值为True的时候，对象便不受投射器影响。

## key:CanUseSpriteAtlas

可使用精灵集，当这个值为False的时候，不能使用精灵集。

## key:PreviewType

材质的预览形式，默认显示为球体，可以使用Plane（2D平面）或Skybox（天空盒）

# 2.2（Pass）Tags

## key:LightMode

光照模式

## value

Always------------------总是渲染，不使用光照
ForwardBase-------------用于前向渲染，使用环境光、主平行光、顶点/SH（球谐函数）光照以及光照贴图
ForwardAdd--------------用于前向渲染，额外使用每像素光，每个光照一个通道
Deferred----------------用于延迟着色，渲染G-Buffer
ShadowCaster------------渲染对象的深度到阴影贴图或者深度纹理
PrepassBase-------------用于（旧版）延迟光照，渲染法线和高光指数
PrepassFinal------------用于（旧版）延迟光照，合并贴图、光照和自发光来渲染最终色彩
Vertex------------------当对象不受光照贴图影响的时候，用来渲染（旧版）顶点发光。使用所有的顶点光照
VertexLMRGBM------------当对象接受光照贴图影响的时候，用来渲染（旧版）顶点发光。适用于使用RGBM编码光照贴图的平台（PC&主机）
VertexLM----------------当对象接受光照贴图影响的时候，用来渲染（旧版）顶点发光。适用于使用double-LDR编码光照贴图的平台（移动平台）

## key:PassFlags

标志渲染管线如何传递数据给通道

## value

OnlyDirectional---------只有主平行光、环境光和光照探测器的数据会传递给通道。仅用于LightMode为ForwardBase

## key:RequireOptions

标志通道至于在某些外部条件满足时才会被渲染
SoftVegetation----------当Quality Setting中的Soft Vegetation选项被开启时，才会渲染通道

# 3.RenderSetUp(以下写在CGPROGRAM之前)（参考书籍P31）

--------------------------------------------------------------------------

## 3.1 Cull

Cull Back | Front | Off 
设置剔除模式：剔除背面/正面/关闭剔除

## 3.2 ZTest

ZTest Less Greater | LEqual | GEqual | Equal | NotEqual | Always
设置深度测试时使用的函数

## 3.3 ZWrite

ZWrite On | Off
开启/关闭深度写入

## 3.4 Blend

BlendOp (Op)
设置混合操作
Blend (SrcFactor) (DstFactor)
Blend (SrcFactor) (DstFactor),(SrcFactorA) (DstFactorA)
开启并设置混合模式（透明材质用）
例如：Blend SrcAlpha OneMinusSrcAlpha

其中SrcFactor为源颜色乘以的混合因子
DstFactor为目标颜色乘以的混合因子
两者相加存入颜色缓冲区

Blend混合操作表（参考书籍P174）
-----------------------------------------

Add         |     源颜色和目标颜色相加
Sub         |     源颜色减去目标颜色
RevSub      |     目标颜色减去源颜色
Min         |     使用目标颜色跟源颜色中较小的值
Max         |     使用源颜色跟目标颜色中较大的值

Blend混合因子表（参考书籍P174）
-----------------------------------------

One              |     1
Zero             |     0
SrcColor         |     源颜色值
SrcAlpha         |     源颜色的透明度值
DstColor         |     目标颜色值
DstAlpha         |     目标颜色的透明度值
OneMinusSrcColor |     1-源颜色值
OneMinusSrcAlpha |     1-源颜色的透明值
OneMinusDstColor |     1-目标颜色值
OneMinusDstAlpha |     1-目标颜色的透明度值

常用混合类型
-----------------------------------------

正常--------| Blend SrcAlpha OneMinusSrcAlpha
柔向相加----| Blend OneMinusDstColor One
正片叠底----| Blend DstColor Zero
两倍相乘----| Blend DstColor SrcColor
变暗--------| BlendOp Min
------------| Blend One One
变亮--------| BlendOp Max
------------| Blend One One
滤色--------| Blend OneMinusDstColor One
线性减淡----| Blend One One

# 4.Pass

--------------------------------------------------------------------------

渲染所使用的通道，可以使用抓取屏幕图像的Pass--GrabPass
以及引用其他shader的Pass--UsePass
以及自己编写的Pass

# 5.vert & frag

--------------------------------------------------------------------------

顶点着色器，编写案例如下
需要前置结构体a2v
输出结构体v2f
（应用->顶点坐标->顶点着色器->输出顶点）
v2f vert (a2v v){
    v2f o;
    //着色器内容
    return o;
}

需要前置结构体v2f
输出着色器颜色值fixed4

（顶点->片元着色器->输出颜色）

fixed4 frag (v2f i) : SV_Target{
 fixed4 tex = ...;
 //着色器内容
 return tex;
}

## 5.0 结构体语义（参考书籍P110）

### a2v------从应用阶段传递模型数据给顶点着色器

| 语义          | 描述                                         | 类型            |
| ----------- | ------------------------------------------ | ------------- |
| POSITION    | 模型空间的顶点位置                                  | float4        |
| NORMAL      | 顶点法线                                       | float3        |
| TANGENT     | 顶点切线                                       | float3        |
| TEXCOORD0-7 | 该顶点的纹理坐标，可以定义TEXCOORD0，TEXCOORD1表示第一组或者第二组 | float2&float4 |
| COLOR       | 顶点颜色                                       | fixed4&float4 |

### v2f------从顶点着色器传递数据给片元着色器

| 语义          | 描述                | 类型     |
| ----------- | ----------------- | ------ |
| SV_POSITION | 裁剪空间中的顶点坐标        | float4 |
| COLOR0      | 通常用于输出第一组顶点颜色，非必需 | fixed4 |
| COLOR1      | 通常用于输出第二组顶点颜色，非必需 | fixed4 |
| TEXCOORD0-7 | 用于输出纹理坐标0-7       | 任意     |

### SV_Target------片元着色器输出时用的语义

输出值将会存储到渲染目标(Render Target)中。

### VPOS/WPOS------片元着色器传递屏幕坐标的语义

类型为float4，xy为屏幕空间中的坐标，z为近裁剪平面处于远裁剪平面处的分量，范围0-1。对于w，如果摄像机投影类型为正交投影，w恒为1，如果使用透视投影，w的取值范围为[1/Near,1/Far],Near为近裁剪平面处于摄像机的距离，Far为远裁剪平面处于摄像机的距离。

## 5.1 变换矩阵（参考书籍P87）

-----------------------------------------

### UNITY_MATRIX_MVP（取模现在被UnityObjectToClipPos(v.vertex)所替代）

当前模型的观察投影矩阵，顶点/方向矢量 模型空间->裁剪空间

### UNITY_MATRIX_MV

当前的模型观察矩阵，顶点/方向矢量 模型空间->观察空间

### UNITY_MATRIX_V

当前的观察矩阵，顶点/方向矢量 世界空间->观察空间

### UNITY_MATRIX_P

当前的投影矩阵，顶点/方向矢量 观察空间->裁剪空间

### UNITY_MATRIX_VP

当前的观察投影矩阵，顶点/方向矢量 世界空间->裁剪空间

### UNITY_MATRIX_T_MV

UNITY_MATRIX_MV的转置矩阵

### UNITY_MATRIX_IT_MV

UNITY_MATRIX_MV的逆转置矩阵，用于将发现从模型空间变换到观察空间，也可用于得到
UNITY_MATRIX_MV的逆矩阵

### unity_ObjectToWorld（原为_Object2World）

当前的模型矩阵，用于将顶点/方向矢量从模型空间变换到世界空间

### unity_WorldToObject（原为_World2Object）

unity_ObjectToWorld的逆矩阵，用于将顶点/方向矢量从世界空间变换到模型空间

## 5.2 Unity内置的摄像机和屏幕参数（参考书籍P88）

-------------------------------------------------

### 【float3】 _WorldSpaceCameraPos

该摄像机在世界空间中的位置

### 【float4】 _ProjectionParams

x = 1.0 & -1.0 | y = Near | z = Far | w = 1.0 + 1.0 / Far
Near = 近裁剪平面到摄像机的距离
Far = 远裁剪平面到摄像机的距离

### 【float4】 _ScreenParams

x = width | y = height | z = 1.0 + 1.0 / width | w = 1.0 + 1.0 / height
width = 该摄像机的渲染目标的像素宽度
height = 该摄像机的渲染目标的像素高度

### 【float4】 _ZBufferParams

x = 1 - Far / Near | y = Far / Near | z = x / Far | w = y / Far
该变量用于线性化Z缓存中的深度值

### 【float4】unity_OrthoParams

x = width | y = height | z = 无定义 | w = 1.0(该摄像机为正交摄像机) & 0.0(该摄像机为透视摄像机)
width = 正交投影摄像机的宽度
height = 正交投影摄像机的高度

### 【float4x4】unity_CameraProjection

该摄像机的投影矩阵

### 【float4x4】unity_CameraInvProjection

该摄像机的投影矩阵的逆矩阵

### 【float4[6]】unity_CameraWorldClipPlanes[6]

该摄像机的6个裁剪平面在世界空间下的等式
按如下顺序：左、右、上、下、近、远裁剪平面

## 5.3 UnityCG.cginc常用的帮助函数（参考书籍P137）

-------------------------------------------------

### 【float3】 WorldSpaceViewDir(float4 v)

输入：模型空间中的顶点位置
返回：世界空间红从该点到摄像机的观察方向。
ps：内部实现使用了UnityWorldSpaceViewDir函数

### 【float3】 UnityWorldSpaceViewDir(float4 v)

输入：世界空间中的顶点位置
返回：世界空间中从该点到摄像机的观察方向。

### 【float3】 ObjectSpaceViewDir(float4 v)

输入：一个模型空间中的顶点位置
返回：模型空间中该点到摄像机的观察方向。

### 【float3】 WorldSpaceLightDir(float4 v)【仅用于前向渲染】【未被归一化】

输入：一个模型空间中的顶点位置
返回：世界空间中从该点到光源的光照方向。
ps：内部实现使用了UnityWorldSpaceLightDir函数

### 【float3】 UnityWorldSpaceLightDir(float4 v)【仅用于前向渲染】【未被归一化】

输入：世界空间中的顶点位置
返回：世界空间中该点到光源的光照方向

### 【float3】 ObjSpaceLightDir(float4 v)【仅用于前向渲染】【未被归一化】

输入：一个模型空间中的顶点位置
返回：模型空间中从该点到光源的光照方向

### 【float3】 UnityObjectToWorldNormal(float3 normal)

把法线方向从模型空间转换到世界空间中

### 【float3】 UnityObjectToWorldDir(float3 dir)

把方向矢量从模型空间变换到世界空间中

### 【float3】 UnityWorldToObjectDir(float3 dir)

把方向矢量从世界空间变换到模型空间中

# 6.关于光照

## 6.1 Unity渲染路径

------------------------------------------------------------

在tag-LightMode里设置，详情见本文2.2

## 6.2 Unity内置光照变量和函数

------------------------------------------------------------

### 【float4】 _LightColor0

该Pass处理的逐像素光源的颜色

### 【float4】 _WorldSpaceLightPos0

该Pass处理的逐像素光源的位置。
如果该光源是平行光，_WorldSpaceLightPos0.w=0
如果是其他类型光，_WorldSpaceLightPos0.w=1

### 【float4x4】 _LightMatrix0

从世界空间到光源空间的变换矩阵。可以用于采样cookie和光照衰减纹理

### 【float4】 unity_4LightPosX0 unity_4LightPosY0 unity_4LightPosZ0

仅用于BasePass
前四个非重要的点光源在世界空间中的位置

### 【float4】 unity_4LightPosX0 unity_4LightPosY0 unity_4LightPosZ0

仅用于BasePass
前四个非重要的点光源在世界空间中的位置

### 【float4】 unity_4LightAtten0

仅用于BasePass
前四个非重要的点光源的衰减因子

### 【half4[4]】 unity_LightColor

仅用于BasePass
前四个非重要的点光源的颜色

## 6.3 Unity前向渲染可以使用的内置光照参数

------------------------------------------------------------

以下函数仅用于前向渲染才可使用。

### 【float3】 WorldSpaceLightDir(float4 v)【未被归一化】

输入：一个模型空间中的顶点位置
返回：世界空间中从该点到光源的光照方向。
ps：内部实现使用了UnityWorldSpaceLightDir函数

### 【float3】 UnityWorldSpaceLightDir(float4 v)【未被归一化】

输入：一个世界空间中的顶点位置
返回：世界空间中从该点到光源的光照方向。

### 【float3】 ObjSpaceLightDir(float4 v)【未被归一化】

输入：一个模型空间中的顶点位置
返回：模型空间中从该点到光源的光照方向。

### 【float3】 Shade4PointLights(...)

输入：已经打包进矢量的光照数据，参考6.2
返回：计算逐顶点光照。
