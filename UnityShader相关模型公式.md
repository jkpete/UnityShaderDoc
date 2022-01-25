# 引言

--------------------------------

结合UnityShader 编写指南中的部分编码，该文档仅提供编写思路，对常用的几个渲染模型进行了编写。

感谢冯乐乐的《Unity Shader入门精要》，参考书本代码的基础上，重新整理了部分代码，以及添加了些许自己的原创代码，方便各位读者查阅使用。

# 0.所用数学函数

#### 

#### mul()

- mul(M,N):计算两个矩阵相乘
- mul(M,v):计算矩阵和向量相乘
- mul(v,M):计算向量和矩阵相乘

#### normalize()

- normalize():Cg语言标准函数库中的函数  
- 函数功能：对向量进行归一化（按比例缩短到单位长度，方向不变）

#### dot(A,B) 【a·b = (ax,ay,az) · (bx,by,bz) = axbx + ayby + azbz 】

- 功能：返回A和B的点积
- 参数：A和B可以是标量，也可以是向量

#### cross（A,B）【a×b = (ax,ay,az) × (bx,by,bz) = (aybz - azby , azbx - axbz , axby - aybx) 】

- 功能：返回A和B的叉积
- 参数：A和B可以是标量，也可以是向量

#### Saturate(x)

- 功能：把x截取在【0,1】范围内，如果x是一个矢量，那么会对它的每一个分量进行这样的操作
- 参数：x为总局关于操作的标量或矢量，float  float2-4

#### step(a,x)【可用于替代部分if判断】

- 功能：输入两个数，如果x<a返回0。
- 如果x>a或者x=a返回1。
- 参数：x输入变量，a为参考量。

#### clamp(x,a,b)

- 功能：把x截取在【a,b】范围内，如果x<a，返回a；
- 如果x>b，返回b；
- 如果x>a且x<b，返回x
- 参数：x输入变量，a，b为参考量，且a < b。

#### pow(a,b)

- 功能：返回a的b次方

#### smoothstep(min,max,x)

- 功能：

#### lerp(y1,y2,weight)

- 功能：插值函数，按照比重返回一个处于y1-y2区间的值。(weight一般限制0-1区间)
- 实际返回值为：y1+(y2-y1)*weight
- 参数：y1为区间下限，y2为区间上限，weight为比重。

# 1. 基础篇

## 

## 坐标空间

---

一个模型被展示到屏幕之前，需要经历以下三个坐标空间的转换，才能正确的被渲染出来。

### 

### 1.1 模型空间

模型空间的原点，坐标轴，以及每个模型的顶点位置是由建模软件决定的。

每个模型都有自己独立的坐标空间。

当模型移动时，模型空间也会跟随移动。

当模型旋转时，模型空间也会跟随旋转。

### 

### 1.2 世界空间

世界空间被用于描述绝对位置，即Unity世界坐标系的位置。

使用左手坐标系。

每个模型可以通过Unity右侧面板中的Transform属性在世界坐标系中进行转换。

Transform的位置是相对于Transform的父节点的位置进行的变换。

一个Transform没有任何父节点的情况下，这个位置就是在世界坐标系中的位置。

### 

### 1.3 观察空间

观察空间是模型空间的一个特殊例子，服务于一类特殊的模型，叫做摄像机。

摄像机决定了我们渲染游戏所以使用的视角。

使用右手坐标系。

观察空间的x轴正方向指向右方，y轴正方向指向上方，z轴正方向指向前方。

### 

### 1.4 裁剪空间

裁剪空间的目标是能够方便的对渲染图元进行裁剪。

完全位于这块空间内部的图元会被保留。

完全位于这块空间之外的图元会被剔除。

位于边界相交的图元会被裁剪。

这块空间由Unity摄像机的视锥体决定。

由于Unity摄像机可以使用正交投影跟透视投影，所以会存在长方体的裁剪空间与椎体的裁剪空间。

### 

### 1.5 空间变换（三个步骤）

##### 模型空间 --1--> 世界空间 --2--> 观察空间 --3--> 裁剪空间

以上步骤可以按顺序进行组合，变换时，写作 mul(UNITY_MATRIX_对应矩阵 , v.vertex);

---

1 = UNITY_MATRIX_M = _Object2World = unity_ObjectToWorld

2 = UNITY_MATRIX_V

3 = UNITY_MATRIX_P

1+2 = UNITY_MATRIX_MV

2+3 = UNITY_MATRIX_VP

1+2+3 = UNITY_MATRIX_MVP = UnityObjectToClipPos(v.vertex) 

1的逆矩阵 = _World2Object = unity_WorldToObject

1+2的转置矩阵 = UNITY_MATRIX_T_MV

1+2的逆转置矩阵 = UNITY_MATRIX_IT_MV 

### 

### 1.6 法线变换（使用上面1）

##### 模型法线空间--1-->世界法线空间

1 = (float3x3)unity_ObjectToWorld = UnityObjectToWorldNormal(v.normal)

由于上述矩阵进行变换相乘的时候，没有进行归一化，需要对法线重新进行归一化才能得到正确的结果

即：normalize(mul(unity_ObjectToWorld,v.normal));

### 

### 1.7 屏幕坐标获取

###### 1.7.1 语义获取 VPOS/WPOS

```c
fixed4 frag(float4 sp : VPOS) : SV_Target{
    //用屏幕坐标处于屏幕分辨率_ScreenParams.xy，得到视口空间中的坐标
    return fixed4(sp.xy/_ScreenParams.xy,0.0,1.0);
}
```

###### 1.7.2 使用ComputeScreenPos函数

```c
struct appdata{
    float4 vertex : POSITION;
    float2 uv : TEXCOORD0;
};

struct v2f{
    float4 vertex : SV_POSITION;
    float4 scrPos : TEXCOORD0;
};

v2f vert(appdata v){
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    o.srcPos = ComputeScreenPos(o.vertex);
    return o;
}

fixed4 frag(v2f i) : SV_target{
    float2 wcoord = (i.srcPos.xy/i.srcPos.w);
    return fixed4(wcoord,0.0,1.0);
}
```

### 1.8 Unity顶点&片元着色器创建案例

右键----》Create----》shader----》UnlitShader----》起名

即可生成一个最简单的带默认纹理的无光照模型着色器

### 1.9 关于优化以及编写建议

1.调试时，可以采用Color的色值采样，这种方法虽然原始，但可以作为一个数据来源的依据

2.调试时，可以使用FrameDebugger，查看渲染事件，大概在Window菜单栏下面，具体哪个子项目视版本情况而定

3.关于使用不同精度值的变量

由于移动平台下面对GPU性能的支持不一定好，所以可以采用低精度数值的方式来对Shader进行优化。

float（32位），half（16位，范围-60000 ~ +60000），fixed（11位，范围-2.0 ~ +2.0）

4.慎用if分支语句，for&while等循环语句

可以用step()函数来做大小与的判断

lerp函数（）

5.不要除以0

控制变量的时候尽量少用除法，尽可能用乘小数的方式进行。

# 2. 光照篇

### 

### 2.1 漫反射光照模型的实现

逐顶点光照（下方仅实现光照模型，如需贴图材质颜色等参数，自行相关参数添加）

平行光法线计算+环境光

新建shader ，拷贝下方代码到SubShader -> Pass里

```c
Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR;
            };

            v2f vert(a2v v) {
                v2f o;
                // 模型顶点数据，从模型空间转换到裁剪空间
                o.pos = UnityObjectToClipPos(v.vertex);

                // 将法线从模型空间转换到世界空间
                // 旧：fixed3 worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);

                // 获取环境光
                // 旧：fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                // ShadeSH9函数使用球谐光照，这里不做过多科普，感兴趣可以自行搜索，旧版是固定颜色环境光
                fixed3 ambient = ShadeSH9(fixed4(worldNormal,1));

                // 获取世界空间中的光照方向
                fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
                // 计算最终光照
                fixed3 diffuse = _LightColor0.rgb * saturate(dot(worldNormal, worldLight));
                // 添加环境光
                o.color = ambient + diffuse;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                return fixed4(i.color, 1.0);
            }

            ENDCG
        }
```

逐像素光照（下方仅实现光照模型，如需贴图材质颜色等参数，自行相关参数添加）

相比逐顶点，光照效果更加平滑。

```c
Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                // 旧：o.worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                // 旧：fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 ambient = ShadeSH9(fixed4(i.worldNormal,1));
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 diffuse = _LightColor0.rgb * saturate(dot(worldNormal, worldLightDir));
                fixed3 color = ambient + diffuse;
                return fixed4(color, 1.0);
            }

            ENDCG
        }
```

半兰伯特光照模型

过时的经验模型，早期应用于游戏《半条命》。现在多采用球谐光照作为环境光的补正，参考上方的 ShadeSH9 函数。

```c
Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            fixed4 _Diffuse;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = mul(v.normal, (float3x3)unity_WorldToObject);
                // 旧：o.worldNormal = normalize(mul(v.normal, (float3x3)unity_WorldToObject));
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed halfLambert = dot(worldNormal, worldLightDir) * 0.5 + 0.5;
                fixed3 diffuse = _LightColor0.rgb * halfLambert;
                fixed3 color = ambient + diffuse;
                return fixed4(color, 1.0);
            }

            ENDCG
        }
```

### 2.2 高光反射光照模型的实现

逐顶点高光

```c
Shader "TestShader/SpecularVertex-LevelOnly" {
    Properties {
        // 高光颜色
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        // 高光光泽度(反光度)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                // 在世界空间中获取光线反射方向
                fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
                // 在世界空间中获取观察方向
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - mul(unity_ObjectToWorld, v.vertex).xyz);
                // 融合光照，高光参数生成光照颜色
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
                o.color = specular;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                return fixed4(i.color, 1.0);
            }

            ENDCG
        }
    } 
    FallBack "Specular"
}
```

逐像素光照(部分代码可以参考逐顶点篇注释)

```c
Shader "TestShader/SpecularPixel-LevelOnly" {
    Properties {
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

                fixed3 reflectDir = normalize(reflect(-worldLightDir, worldNormal));
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(reflectDir, viewDir)), _Gloss);
                return fixed4(specular, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Specular"
}
```

Blin-Phong高光模型算法

```c
Shader "TestShader/Blinn-PhongOnly" {
    Properties {
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                // 在世界空间中获取half方向
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                // 融合高光反射参数
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                return fixed4(specular, 1.0);
            }

            ENDCG
        }
    } 
    FallBack "Specular"
}
```

### 2.3 Unity设置LightMode实现其他光源

关于渲染路径，一般从摄像机的Rendering Path中设置，默认使用Graphics Settings中的数值，前向渲染。

LightMode的Tag参数详情见编写指南2.2（Pass）Tags

##### 2.3.1Unity对不同光源光照处理的方式以及顺序

- 场景最亮的平行光总是按逐像素处理的。

- Render Mode 被设置成 Not Important 的光源，会按逐顶点或者SH处理

- Render Mode 被设置成 Important 的光源，会按逐像素处理

- 如果根据以上规则得到的逐像素光源数量小于 Quality Setting 中的 Pixel Light Count(逐像素光源数量)，会有更多的光源以逐像素的方式进行渲染。

下方实现了一个多光源的前向渲染Shader。

```c
Shader "TestShader/ForwardRendering" {
    Properties {
        _Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Tags { "RenderType"="Opaque" }
        Pass {
            // 主光源Pass
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            // 保证在Base Pass中使用光照衰减等光照变量可以被正确赋值
            #pragma multi_compile_fwdbase    
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);

                //fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 ambient = ShadeSH9(fixed4(i.worldNormal,1));
                 fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                 fixed3 halfDir = normalize(worldLightDir + viewDir);
                 fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                fixed atten = 1.0;
                return fixed4(ambient + (diffuse + specular) * atten, 1.0);
            }
            ENDCG
        }

        Pass {
            // 其他光照Pass
            Tags { "LightMode"="ForwardAdd" }
            // 混合模式为线性减淡（PS中常用于发光图层）
            Blend One One
            CGPROGRAM
            // 保证在Additional Pass中访问到正确的光照变量
            #pragma multi_compile_fwdadd
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            #include "AutoLight.cginc"

            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                #ifdef USING_DIRECTIONAL_LIGHT
                    // 使用平行光时，不计算衰减
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                #else
                    // 使用其他光源时计算衰减
                    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
                #endif
                fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
                fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                #ifdef USING_DIRECTIONAL_LIGHT
                    // 使用其他平行光
                    fixed atten = 1.0;
                #else
                    #if defined (POINT)
                    // 点光源处理
                        float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
                        fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
                    #elif defined (SPOT)
                    // 聚光源处理
                        float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
                        fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
                    #else
                        fixed atten = 1.0;
                    #endif
                #endif
                return fixed4((diffuse + specular) * atten, 1.0);
            }
            ENDCG
        }
    }
    FallBack "Specular"
}
```

### 2.4 投射阴影与接收阴影

##### 2.4.1设置Cast Shadows

开启阴影渲染之前，必须保证光源可以投射阴影，通过Light-Shadow Type 进行设置，可以开启硬阴影投射与软阴影投射

##### 2.4.1设置RecevieShadows

物体接收不接收阴影信息是由mesh renderer决定的，通过Mesh Renderer-Cast Shadows 设置成On，以开启接收阴影。

投射与接收阴影着色器案例：

```c
Shader "ShaderTest/Shadow" {
    Properties {
        _Diffuse ("Diffuse", Color) = (1, 1, 1, 1)
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Tags { "RenderType"="Opaque" }

        Pass {
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            #include "AutoLight.cginc"

            fixed4 _Diffuse;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                // 声明一个用于对阴影纹理采样的坐标
                SHADOW_COORDS(2)
            };

            v2f vert(a2v v) {
                 v2f o;
                 o.pos = UnityObjectToClipPos(v.vertex);
                 o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                 // 计算上一步中声明的阴影纹理坐标
                 TRANSFER_SHADOW(o);
                 return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);    
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                 fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
                 fixed3 viewDir = normalize(_WorldSpaceCameraPos.xyz - i.worldPos.xyz);
                 fixed3 halfDir = normalize(worldLightDir + viewDir);
                 fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                // 混合阴影颜色
                fixed shadow = SHADOW_ATTENUATION(i);
                return fixed4(ambient + (diffuse + specular) * atten * shadow, 1.0);
            }
            ENDCG
        }
    }
    // 如果不设置FallBack，则该物体不会向其他物体投射阴影，得到的阴影数据也会变得不正常
    FallBack "Specular"
}
```

# 3. 标准纹理篇

###### tex2D(sampler2D,Vector2)

从一张纹理中，对一个点进行采样的方法，返回值为一个float4的颜色值。sampler2D为贴图来源，Vector2为贴图对应的坐标。

###### texCUBE(samplerCUBE,Vector3)

立方体纹理中，对点进行采样的方法，返回值为一个float4的颜色值。samplerCUBE为立方体贴图来源，Vector3为中心点指向的方向坐标（这里提一点，由于立方体纹理中是六个面组成的，所以中心点为立方体模型的中心，中心点到采样点之间会形成一个方向矢量，此矢量为Vector3的值），另外由于该变量代表方向坐标，故不需要进行归一化处理。

### 3.1 关于UV（纹理映射坐标）：

在美术人员建模的时候，通常会在建模软件中利用纹理展开技术把纹理映射坐标存储在每个顶点上，纹理映射坐标定义了该顶点在纹理中对应的2d坐标。

通常，这些坐标使用一个二维变量（u，v）来表示，其中u是横向坐标，v是纵向坐标。因此纹理映射坐标也被称为UV坐标。

虽然纹理的大小多种多样，但顶点UV坐标的范围通常都被归一化到【0-1】范围内。

### 3.2 关于纹理（贴图）资源：

Unity支持以下格式的图片作为贴图资源：bmp，exr，gif，hdr，iff，jpg，pict，png，psd，tga，tiff。

### 3.3 纹理的资源的属性

本篇只做简要概括重要属性，其他属性详情见下方链接

[Unity - Manual: Texture Import Settings](https://docs.unity3d.com/Manual/class-TextureImporter.html)

##### 1.TextureType

声明纹理类型

| 类型                    | 描述                                                       |
| --------------------- | -------------------------------------------------------- |
| Default               | 默认是所有纹理最常用的设置。它提供对纹理导入的大多数属性的访问。                         |
| Normal Map            | 作为法线贴图时，将颜色通道转换为适合实时法线贴图的格式。                             |
| Editor GUI and Legacy | 如果您在任何 HUD 或 GUI 控件上使用纹理，请选择Editor GUI 和 Legacy GUI。     |
| Sprite (2D and UI)    | 正在使用的纹理在2D游戏中作为一个精灵（Sprite）。                             |
| Cursor                | 作为一个默认光标（替代鼠标）。                                          |
| Cookie                | 用于场景光照的基础参数缓存。                                           |
| Lightmap              | 光照贴图，此选项支持编码为特定格式（RGBM 或 dLDR，具体取决于平台），可用于存放屏幕后期处理的贴图数据。 |
| Single Channel        | 如果您只需要纹理中的一个通道，请选择单通道。                                   |

##### 2.WarpMode

该项决定了纹理坐标如果超出【0-1】这个范围，会如何平铺。

| 类型     | 描述                                 |
| ------ | ---------------------------------- |
| Repeat | 坐标超过1时，取小数部分，坐标多出的部分会重复。           |
| Clamp  | 坐标截取到1，小于0的部分截取到0，坐标多出的部分会按照边缘色延伸。 |

##### 3.FilterMode

该项决定了当纹理由于变换产生拉伸时，将使用哪种滤波模式。

| 类型                | 描述                                               |
| ----------------- | ------------------------------------------------ |
| Point (no filter) | 不使用滤波，拉伸时，采样像素只有一个                               |
| Bilinear          | 线性滤波，对于每个目标像素，会找到四个邻近像素，对他们进行线性插值混合得到最终像素，会变得模糊。 |
| Trilinear         | 在Bilinear的基础上，使用多级渐远纹理之间进行混合                     |

##### 4.Default-MaxSize

如果导入的纹理大小超过了这个选项对应的设置值。那么Unity将会把该纹理缩放为这个的最大分辨率。

可以通过调整该选项，来优化移动以及计算性能低的设备平台使之更流畅。

### 3.4 纹理映射案例

使用纹理时，不仅要声明纹理变量本身，还要声明一个 纹理名 + _ST 格式的float4变量。

纹理名 + _ST：xy存放缩放值，zw存放偏移值。用于纹理坐标转换。

```c
Shader "TestShader/TextureOnly" {
    Properties {
        _MainTex ("Main Tex", 2D) = "white" {}
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;

            struct a2v {
                float4 vertex : POSITION;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 position : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            v2f vert(a2v v) {
                 v2f o;
                 o.position = UnityObjectToClipPos(v.vertex);
                 // 转换UV坐标贴图，转换后的样式依据贴图资源设置而改变，该方法自动_MainTex_ST变量
                o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
                 return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed4 c = tex2D(_MainTex, i.uv);
                return fixed4(c.rgb, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Diffuse"
}
```

### 3.5 法线纹理映射案例

在介绍法线映射之前，要提到凹凸映射的两种贴图方案，一种是高度贴图，一种是法线贴图。

高度纹理用于模拟表面位移，亮的部分能遮挡暗的部分，该纹理的颜色用于表达表面海拔高度，颜色越亮值越高。

法线纹理直接储存表面法线，从而使光照信息按照法线贴图提供的数据进行计算。

在使用法线纹理之前，必须了解切线空间的存在意义，想了解TBN矩阵的可以参阅以下文章

https://zhuanlan.zhihu.com/p/139593847

切线空间就是，反应这个模型空间坐标相对应纹理坐标相的变换坡度。引自

http://www.cnitblog.com/wjk98550328/archive/2010/04/15/35112.html

法线贴图的颜色值代表的含义

法线贴图的rgb颜色中，r代表横向法线，y代表纵向法线。法线贴图的具体图解，可详见另一篇法线贴图文档部分。里面有详细的法线贴图数据映射关系。

```c
Shader "TestShader/MineBumpTestShader"
{
    Properties {
        _BumpMap("Normal Map", 2D) = "bump" {}
    }
    SubShader
    {
        Pass
        {
            Tags{"LightMode"="ForwardBase"}
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            #include "UnityCG.cginc"

            sampler2D _BumpMap;

            struct a2v{
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float2 uv : TEXCOORD0;
            };

            struct v2f {
                float3 worldPos : TEXCOORD0;
                float2 uv : TEXCOORD1;
                float4 pos : SV_POSITION;
                half3 wNormal : TEXCOORD2;
                half3 wTangent : TEXCOORD3;
                half3 wBitangent : TEXCOORD4;
            };

            // 顶点着色器现在也需要一个逐顶点的切线向量。
            // 在Unity中，切线是一个四维的向量，w分向量用于表现二次切线向量的方向
            // 我们仍然需要贴图来配合(翻译自Unity Manual 法线贴图篇)
            v2f vert (a2v v)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = v.uv;
                o.wTangent = UnityObjectToWorldDir(v.tangent.xyz);
                o.wNormal = UnityObjectToWorldNormal(v.normal);
                // compute bitangent from cross product of normal and tangent
                // 通过计算法线与切线的叉积，得到二次切线bitangent,以此来导出切线空间矩阵
                // output the tangent space matrix
                half tangentSign = v.tangent.w;
                o.wBitangent = cross(o.wNormal, o.wTangent) * tangentSign;

                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // 依据法线贴图，将rgba的范围（原【0-1】）规范到【-1~1】之间
                half3 tnormal = UnpackNormal(tex2D(_BumpMap, i.uv));
                // 上文提到的TBN矩阵，用于将法线方向从切线空间转换到世界空间中。
                float3x3 TBNMatrix = float3x3(i.wTangent,i.wBitangent,i.wNormal);
                // 根据解包后的法线数据，算出世界空间下的法线方向。
                half3 worldNormal = mul(tnormal,TBNMatrix);
                // 灯光方向
                half3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
                // 计算球谐光照，反应环境光（读者可以尝试只输出这个变量看看效果）
                fixed3 ambient = ShadeSH9(fixed4(worldNormal,1));
                // 计算主光源
                fixed3 diffuse = _LightColor0.rgb * saturate(dot(worldNormal, worldLightDir));
                // 融合环境光与主光源（需要添加高光反射的，去前面高光反射代码摘抄并在下方加算即可）
                fixed4 c = fixed4(ambient+diffuse,1);
                return c;
            }
            ENDCG
        }
    }
}
```

### 3.6 渐变纹理

用于渲染风格化的物体时，可以采用渐变纹理的方式，得到处理之后的阴影。比如卡通风格的渲染，可以采用一张二分图或者三分图当做渐变纹理，来进行渲染。

笔者准备了两张渐变纹理用于做下方shader的素材，详见img/gradient.jpg

```c
Shader "TestShader/RampTexture" {
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        // 渐变贴图声明
        _RampTex ("Ramp Tex", 2D) = "white" {}
        _SpeRampTex ("Spe Ramp Tex", 2D) = "white" {}
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }

            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _RampTex;
            float4 _RampTex_ST;
            sampler2D _SpeRampTex;
            float4 _SpeRampTex_ST;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldNormal : TEXCOORD0;
                float3 worldPos : TEXCOORD1;
                float2 uv : TEXCOORD2;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = TRANSFORM_TEX(v.texcoord, _RampTex);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));

                // 可根据需求调整环境光照
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                //fixed3 ambient = ShadeSH9(fixed4(worldNormal,1));
                // 计算半兰伯特光照
                fixed lambert = dot(worldNormal, worldLightDir);
                fixed halfLambert  = 0.5 * lambert + 0.5;
                
                fixed3 diffuseColor = tex2D(_RampTex, fixed2(halfLambert, halfLambert)).rgb * _Color.rgb;
                diffuseColor = fixed3(diffuseColor.r,diffuseColor.g,diffuseColor.b);
                fixed3 diffuse = _LightColor0.rgb * diffuseColor;

                fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
                fixed3 halfDir = normalize(worldLightDir + viewDir);
                fixed3 specular = pow(max(0, dot(worldNormal, halfDir)), _Gloss);
                specular = tex2D(_SpeRampTex, fixed2(specular.r, specular.r)).rgb/2 * _LightColor0.rgb * _Specular.rgb;
                return fixed4(ambient + diffuse + specular, 1.0);
            }

            ENDCG
        }
    } 
    FallBack "Specular"
}
```

### 3.7 遮罩纹理

遮罩纹理允许我们保护某些区域，使它们免于某些修改

使用遮罩纹理的流程：通过采样得到遮罩纹理的纹素值，然后使用其中的某个（或者某几个）通道的值（比如texture.r）来与某种表面属性进项相乘，这样，当该通道的值为0时，可以保护表面不受该属性的影响。

遮罩纹理在剔除某个数据的需求中起着至关重要的作用（例如更真实的砖瓦缝隙不受高光光照影响）

下方为一个应用于高光反射的遮罩纹理着色器。

```c
Shader "TestShader/MaskTexture" {
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _MainTex ("Main Tex", 2D) = "white" {}
        _BumpMap ("Normal Map", 2D) = "bump" {}
        _BumpScale("Bump Scale", Float) = 1.0
        _SpecularMask ("Specular Mask", 2D) = "white" {}
        _SpecularScale ("Specular Scale", Range(0.0,1.0)) = 1.0
        _Specular ("Specular", Color) = (1, 1, 1, 1)
        _Gloss ("Gloss", Range(8.0, 256)) = 20
    }
    SubShader {
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"

            fixed4 _Color;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            sampler2D _BumpMap;
            float _BumpScale;
            sampler2D _SpecularMask;
            float _SpecularScale;
            fixed4 _Specular;
            float _Gloss;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 lightDir: TEXCOORD1;
                float3 viewDir : TEXCOORD2;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv.xy = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
                TANGENT_SPACE_ROTATION;
                o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
                o.viewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                 fixed3 tangentLightDir = normalize(i.lightDir);
                fixed3 tangentViewDir = normalize(i.viewDir);
                fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uv));
                tangentNormal.xy *= _BumpScale;
                tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));
                fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
                fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));
                 fixed3 halfDir = normalize(tangentLightDir + tangentViewDir);
                 // 获取r通道下的遮罩数据值
                 fixed specularMask = tex2D(_SpecularMask, i.uv).r * _SpecularScale;
                 // 混合遮罩数据
                 fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(tangentNormal, halfDir)), _Gloss) * specularMask;
                return fixed4(ambient + diffuse + specular, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Specular"
}
```

## 立方体纹理

在图形学中，立方体纹理是环境映射的一种实现方法。环境映射可以模拟物体周围的环境。

立方体纹理一共包含了6张正方形图像，六张图像边缘连续，组成一个立方体。 

立方体纹理广泛应用于天空盒，反射折射模型中，等等。

### 3.8 反射

反射有两种方式实现，一种是获取天空盒的立方体纹理，另一种是直接使用立方体纹理（总之都采用了对应的立方体纹理）

##### 天空盒反射（摘自Unity Manual）

```c
Shader "Unlit/SkyReflection"
{
    SubShader
    {
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct v2f {
                half3 worldRefl : TEXCOORD0;
                float4 pos : SV_POSITION;
            };

            v2f vert (float4 vertex : POSITION, float3 normal : NORMAL)
            {
                v2f o;
                o.pos = UnityObjectToClipPos(vertex);
                // compute world space position of the vertex
                float3 worldPos = mul(_Object2World, vertex).xyz;
                // compute world space view direction
                float3 worldViewDir = normalize(UnityWorldSpaceViewDir(worldPos));
                // world space normal
                float3 worldNormal = UnityObjectToWorldNormal(normal);
                // world space reflection vector
                o.worldRefl = reflect(-worldViewDir, worldNormal);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the default reflection cubemap, using the reflection vector
                half4 skyData = UNITY_SAMPLE_TEXCUBE(unity_SpecCube0, i.worldRefl);
                // decode cubemap data into actual color
                half3 skyColor = DecodeHDR (skyData, unity_SpecCube0_HDR);
                // output it!
                fixed4 c = 0;
                c.rgb = skyColor;
                return c;
            }
            ENDCG
        }
    }
}
```

##### 立方体纹理反射

```c
Shader "TestShader/Reflection" {
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _ReflectColor ("Reflection Color", Color) = (1, 1, 1, 1)
        _ReflectAmount ("Reflect Amount", Range(0, 1)) = 1
        _Cubemap ("Reflection Cubemap", Cube) = "_Skybox" {}
    }
    SubShader {
        Tags { "RenderType"="Opaque" "Queue"="Geometry"}

        Pass { 
            Tags { "LightMode"="ForwardBase" }
            Cull Off
            CGPROGRAM
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag
            #include "Lighting.cginc"
            #include "AutoLight.cginc"

            fixed4 _Color;
            fixed4 _ReflectColor;
            fixed _ReflectAmount;
            samplerCUBE _Cubemap;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                fixed3 worldNormal : TEXCOORD1;
                fixed3 worldViewDir : TEXCOORD2;
                fixed3 worldRefl : TEXCOORD3;
                SHADOW_COORDS(4)
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
                // 计算世界空间下的反射参数
                o.worldRefl = reflect(-o.worldViewDir, o.worldNormal);
                // 投射阴影
                TRANSFER_SHADOW(o);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));        
                fixed3 worldViewDir = normalize(i.worldViewDir);
                // 设置环境光颜色
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                //fixed3 ambient = ShadeSH9(fixed4(worldNormal,1));
                // 计算漫反射光照
                fixed3 diffuse = _LightColor0.rgb * _Color.rgb * max(0, dot(worldNormal, worldLightDir));
                // 使用世界空间下的反射方向来映射立方体纹理
                fixed3 reflection = texCUBE(_Cubemap, i.worldRefl).rgb * _ReflectColor.rgb;
                // 混合阴影颜色
                UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
                // 混合最终颜色
                fixed3 color = ambient + lerp(diffuse, reflection, _ReflectAmount) * atten;
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    }
    FallBack "Reflective/VertexLit"
}
```

### 3.9 折射

当光线从一种介质斜射进入另一种介质时，传播方向一般会发生改变。

斯涅尔定律：n1 sinθ1 = n2 sinθ2

其中 n1、n2分别是两个介质的折射率。当光从介质1沿着和表面法线夹角为θ1的方向斜射入介质2时，我们可以使用上述公式计算折射光线与法线的夹角θ2

下方案例均仅模拟一次折射

##### 使用立方体纹理进行折射模拟

```c
Shader "TestShader/Refraction" {
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
        _RefractColor ("Refraction Color", Color) = (1, 1, 1, 1)
        _RefractAmount ("Refraction Amount", Range(0, 1)) = 1
        _RefractRatio ("Refraction Ratio", Range(0.1, 1)) = 0.5
        _Cubemap ("Refraction Cubemap", Cube) = "_Skybox" {}
    }
    SubShader {
        Tags { "RenderType"="Opaque" "Queue"="Geometry"}
        Pass { 
            Tags { "LightMode"="ForwardBase" }

            CGPROGRAM

            #pragma multi_compile_fwdbase    
            #pragma vertex vert
            #pragma fragment frag

            #include "Lighting.cginc"
            #include "AutoLight.cginc"

            fixed4 _Color;
            fixed4 _RefractColor;
            float _RefractAmount;
            fixed _RefractRatio;
            samplerCUBE _Cubemap;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                fixed3 worldNormal : TEXCOORD1;
                fixed3 worldViewDir : TEXCOORD2;
                fixed3 worldRefr : TEXCOORD3;
                SHADOW_COORDS(4)
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
                // Compute the refract dir in world space
                o.worldRefr = refract(-normalize(o.worldViewDir), normalize(o.worldNormal), _RefractRatio);
                TRANSFER_SHADOW(o);

                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
                fixed3 worldViewDir = normalize(i.worldViewDir);                
                fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
                fixed3 diffuse = _LightColor0.rgb * _Color.rgb * max(0, dot(worldNormal, worldLightDir));
                // Use the refract dir in world space to access the cubemap
                fixed3 refraction = texCUBE(_Cubemap, i.worldRefr).rgb * _RefractColor.rgb;
                UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
                // Mix the diffuse color with the refract color
                fixed3 color = ambient + lerp(diffuse, refraction, _RefractAmount) * atten;
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Reflective/VertexLit"
}
```

##### 使用渲染纹理进行折射模拟

亲手写的，附带法线纹理的转换（使用TBN矩阵），使用了上述的计算公式，渲染纹理详情见下章。效果酷似实体玻璃

```c
Shader "Unlit/TestRefractShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _BumpMap ("Normal Map", 2D) = "bump" {}
        _GlassColor ("Glass Color" ,COLOR) = (1,1,1,1) 
        _ScaleTest("Scale Vertex" ,Range(0,1)) = 1
        _TexTint("Tex Tint",Range(0.0,3.0)) = 0.0
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Transparent" }
        LOD 100
        GrabPass { "_RefractionTex2" }
        Cull Off
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            //#pragma multi_compile_fog

            #include "UnityCG.cginc"
            sampler2D _RefractionTex2;
            float4 _RefractionTex2_ST;
            sampler2D _BumpMap;
            float4 _BumpMap_ST;
            sampler2D _MainTex;
            float4 _MainTex_ST;
            float4 _GlassColor;
            float _TexTint;
            float _ScaleTest;


            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 tangent : TANGENT;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float4 vertex : SV_POSITION;
                float4 scrPos : TEXCOORD0;
                float2 uv : TEXCOORD1;
                half3 wNormal : TEXCOORD2;
                half3 wTangent : TEXCOORD3;
                half3 wBitangent : TEXCOORD4;
                float3 worldPos : TEXCOORD5;
                UNITY_FOG_COORDS(1)
            };



            v2f vert (a2v v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                //o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                //o.vertex = float4(o.vertex.x+(1+sin(o.vertex.x+_ScaleTest)),o.vertex.y,o.vertex.z,o.vertex.w);
                o.scrPos = ComputeGrabScreenPos(o.vertex);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.uv = v.uv;
                o.wTangent = UnityObjectToWorldDir(v.tangent.xyz);
                o.wNormal = UnityObjectToWorldNormal(v.normal);
                // compute bitangent from cross product of normal and tangent
                // 通过计算法线与切线的叉积，得到二次切线bitangent,叉积*切线方向
                // half tangentSign = v.tangent.w * unity_WorldTransformParams.w;
                // output the tangent space matrix
                half tangentSign = v.tangent.w;
                o.wBitangent = cross(o.wNormal, o.wTangent) * tangentSign;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 tex = tex2D(_MainTex, i.uv);
                half3 tnormal = UnpackNormal(tex2D(_BumpMap, i.uv));
                float3x3 TBNMatrix = float3x3(i.wTangent,i.wBitangent,i.wNormal);
                half3 worldNormal = mul(tnormal,TBNMatrix);
                half3 worldViewDir = UnityWorldSpaceViewDir(i.worldPos);

                half3 worldRefra = refract(worldViewDir,worldNormal,_ScaleTest);

                fixed4 texTint = _TexTint*fixed4(i.vertex.z,i.vertex.z,i.vertex.z,1);
                // sample the texture
                float2 offset = worldRefra.xy;
                i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy;
                //i.scrPos.xy = tex.xy * i.scrPos.z + i.scrPos.xy;
                fixed4 refrCol = tex2D(_RefractionTex2, i.scrPos.xy/i.scrPos.w);
                fixed4 col = tex2D(_RefractionTex2, i.uv);
                return refrCol*_GlassColor+tex*texTint;
                //return tex*_TexTint;
            }
            ENDCG
        }
    }
}
```

### 3.10 菲涅尔反射

真实世界的菲涅尔等式是非常复杂的，实时渲染中，通常会使用一些近似公式来计算。

##### Schlick菲涅尔近似等式

F (v,n) = F0 + ( 1 - F0 ) ( 1 - v · n ) ^ 5

##### Empricial菲涅尔近似等式 ( bias,scale,power是控制项 )

F (v,n) = max( 0 , min ( 1,  bias + scale * ( 1 - v · n ) ^ power) ) 

schlick菲涅尔示例

```c
Shader "Unlit/TestFresnelShader"
{
    Properties {
        _Color ("Color Tint", Color) = (0, 0, 0, 1)
        _ReflectColor ("Reflect Color",Color) = (0, 0, 1, 1)
        _FresnelScale ("Fresnel Scale", Range(0, 1)) = 0.5
    }
    SubShader {
        Tags { "RenderType"="Opaque" "Queue"="Geometry"}

        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            fixed4 _Color;
            fixed4 _ReflectColor;
            fixed _FresnelScale;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                  fixed3 worldNormal : TEXCOORD1;
                  fixed3 worldViewDir : TEXCOORD2;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldViewDir = normalize(i.worldViewDir);
                fixed3 ambient = ShadeSH9(fixed4(i.worldNormal,1));
                fixed3 reflection = _ReflectColor.rgb;
                fixed fresnel = _FresnelScale + (1 - _FresnelScale) * pow(1 - dot(worldViewDir, worldNormal), 5);
                fixed3 diffuse = _Color.rgb;
                fixed3 color = ambient + lerp(diffuse, reflection, saturate(fresnel));
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Reflective/VertexLit"
}
```

Empricial 菲涅尔示例

```c
Shader "Unlit/TestFresnelShader"
{
    Properties {
        _Color ("Color Tint", Color) = (0, 0, 0, 1)
        _ReflectColor ("Reflect Color",Color) = (0, 0, 1, 1)
        _FresnelScale ("Fresnel Scale", Range(0, 1)) = 1
        _FresnelBias ("Bias", Range(0, 1)) = 0
        _FresnelPower ("Power", Range(0, 5)) = 3
    }
    SubShader {
        Tags { "RenderType"="Opaque" "Queue"="Geometry"}
        Pass { 
            Tags { "LightMode"="ForwardBase" }
            CGPROGRAM
            #pragma multi_compile_fwdbase
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            fixed4 _Color;
            fixed4 _ReflectColor;
            fixed _FresnelScale;
            float _FresnelBias;
            int _FresnelPower;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 worldPos : TEXCOORD0;
                  fixed3 worldNormal : TEXCOORD1;
                  fixed3 worldViewDir : TEXCOORD2;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.worldNormal = UnityObjectToWorldNormal(v.normal);
                o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                o.worldViewDir = UnityWorldSpaceViewDir(o.worldPos);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 worldNormal = normalize(i.worldNormal);
                fixed3 worldViewDir = normalize(i.worldViewDir);
                fixed3 ambient = ShadeSH9(fixed4(i.worldNormal,1));
                fixed3 reflection = _ReflectColor.rgb;
                //fixed fresnel = _FresnelScale + (1 - _FresnelScale) * pow(1 - dot(worldViewDir, worldNormal), 5);
                fixed fresnel = max(0,min(1,_FresnelBias+_FresnelScale * pow(1 - dot(worldViewDir, worldNormal),_FresnelPower)));
                fixed3 diffuse = _Color.rgb;
                fixed3 color = ambient + lerp(diffuse, reflection, saturate(fresnel));
                return fixed4(color, 1.0);
            }
            ENDCG
        }
    } 
    FallBack "Reflective/VertexLit"
}
```

# 4. 系统内置纹理篇

## 渲染纹理

现代的GPU允许我们吧整个三维场景渲染到一个中间缓冲中，即渲染目标纹理。

与之相关的是 多重渲染目标 这种技术指的是GPU允许我们吧场景同事渲染到多个渲染目标纹理中，不再需要为每个渲染目标纹理单独渲染完整的场景。延迟渲染就是使用多重渲染目标的一个应用。

unity为渲染目标纹理定义了一种专门的纹理类型----渲染纹理。在Unity中使用渲染纹理通常有两种方式：

1.在Project面板下创建一个RenderTexture，通过场景中的Camera的渲染目标设置成该渲染纹理。

2.屏幕后处理时，使用GrabPass命令或OnRenderImage函数来获取当前屏幕图像。

### 4.1 使用RenderTexture

1.在Project面板下，右键 -> Create -> Render Texture ，并且起对应的名字

2.在Hierarchy面板下，右键创建一个Camera，并把Target Texture设置成上一步创建的Render Texture

3.创建一个Material，使用第一步创建的RenderTexture作为主颜色纹理。

4.在场景中的物体上赋予该Material，查看效果。

### 4.2 使用GrabPass

下方使用了两行代码完成了屏幕的抓取，分别是

GrabPass{}、fixed4 refrCol = tex2D(_RefractionTex, i.uv);

但此时我们会发现，抓取屏幕会陷入一个连续抓取的死循环中，这种错误的结果需要进行相关处理后才能使用。设置渲染顺序为Transparent，可以使得抓取屏幕时，确保其他不透明物体已经被渲染到屏幕上了，屏幕抓取不会陷入连续抓取的死循环中。

```c
Shader "Unlit/GrabPassShader1"
{
    Properties {}
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Transparent" }
        LOD 100
        // 在当前位置声明Grabpass,如果不使用名字的话，每一个使用该
        // shader的物体都会单独进行一次昂贵的屏幕抓取操作。
        // 使用名字的话，Unity只会执行一次抓取工作，但也意味着所有物体都会
        // 使用一张屏幕图像，大多数情况下是够用的。
        // GrabPass {}
        GrabPass { "_RefractionTex" }
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"
            sampler2D _RefractionTex;
            float4 _RefractionTex_TexelSize;

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                // 将抓取到的图像展示出来
                fixed4 refrCol = tex2D(_RefractionTex, i.uv);
                return refrCol;
            }
            ENDCG
        }
    }
}
```

### 4.3 使用 GrabPass 实现透明效果

ComputeGrabScreenPos(vertex)

该函数与1.7中的ComputeScreenPos函数基本类似，最大的不同是针对平台差异造成的采样坐标问题进行了处理。

使用上述函数获得片元在屏幕上的像素位置时，通常需要两个步骤：

第一步，把ComputeScreenPos的结果保存到scrPos中。

第二步，用scrPos.xy除以scrPos.w得到视口空间中的坐标。

```c
Shader "Unlit/GrabPassShader2"
{
    Properties
    {
        _Color ("Texture", COLOR) = (1,1,1,1)
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Transparent" }
        LOD 100
        GrabPass { "_RefractionTex" }
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            fixed4 _Color;
            sampler2D _RefractionTex;
            float4 _RefractionTex_TexelSize;

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
                float4 scrPos : TEXCOORD1;
            };

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv;
                o.scrPos = ComputeGrabScreenPos(o.vertex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                //fixed4 refrCol = tex2D(_RefractionTex,i.uv);
                fixed4 refrCol = tex2D(_RefractionTex,i.scrPos.xy/i.scrPos.w);
                return refrCol+_Color*0.2;
            }
            ENDCG
        }
    }
}
```

如果想修改画面使之产生相应的扭曲效果，以及通过透明图像产生折射效果，可以参考3.9折射篇，使用渲染纹理模拟折射效果其中的代码。部分爆炸特效的空气扭曲，可以通过法线纹理+折射模型实现的。

## 深度纹理

深度纹理实际就是一张渲染纹理，只不过它里面存储的像素值不是颜色值，而是一个高精度的深度值。

深度纹理中的深度值范围是（0,1），而且通常是非线性分布的。

获取深度纹理是非常简单的，可以通过下面的代码来获取深度纹理

camera.depthTextureMode = DepthTextureMode.Depth;

在shader中，我们仅仅需要声明以下变量便可使用

sampler2D _CameraDepthTexture;

如果想查看深度纹理，可以使用FrameDebugger 

Window -> Analysis ->FrameDebugger 中，Open可以查看渲染细节。

下方是一个深度纹理获取后，将深度纹理变为可视化的渲染器示例

```c
Shader "Unlit/DepthShader1"
{
	Properties
	{
		_DepthScale("DepthData",Float) = 0.1
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" "Queue"="Transparent" }
		LOD 100
		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			
			#include "UnityCG.cginc"

			sampler2D _CameraDepthTexture;
			float _DepthScale;

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				float4 vertex : SV_POSITION;
				float4 scrPos : TEXCOORD1;
			};
			
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = v.uv;
				o.scrPos = ComputeGrabScreenPos(o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				// sample the texture
				fixed4 refrCol = tex2D(_CameraDepthTexture, i.scrPos.xy/i.scrPos.w);
				float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
				// 解码深度纹理，输出线性深度值
                float linearDepth = LinearEyeDepth(refrCol.r);
				// 通过减去屏幕深度，获取正确的深度颜色（深度值越高，越白，反之则黑）
                float diff = linearDepth - i.scrPos.w;
                // 通过翻转颜色，使得深度值越深的值越低，并在深度纹理上添加系数调整。
				fixed4 intersect = fixed4(1,1,1,1)-fixed4(diff*_DepthScale,diff*_DepthScale,diff*_DepthScale,1);
				fixed4 border = saturate(intersect);
				return border;
			}
			ENDCG
		}
	}
}


```

深度纹理的进阶应用，可以参考本人在同项目中的另一篇文章，卡通着色器全解析。
