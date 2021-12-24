# 引言

--------------------------------

结合UnityShader 编写指南中的部分编码，该文档仅提供编写思路，对常用的几个渲染模型进行了编写。

# 0.所用数学函数

#### 

#### mul()

- mul(M,N):计算两个矩阵相乘
- mul(M,v):计算矩阵和向量相乘
- mul(v,M):计算向量和矩阵相乘

#### normalize()

- normalize():Cg语言标准函数库中的函数  
- 函数功能：对向量进行归一化（按比例缩短到单位长度，方向不变）

#### dot(A,B)

- 功能：返回A和B的点积
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

# 3. 纹理篇

### 关于UV（纹理映射坐标）：

在美术人员建模的时候，通常会在建模软件中利用纹理展开技术把纹理映射坐标存储在每个顶点上，纹理映射坐标定义了该顶点在纹理中对应的2d坐标。

通常，这些坐标使用一个二维变量（u，v）来表示，其中u是横向坐标，v是纵向坐标。因此纹理映射坐标也被称为UV坐标。

虽然纹理的大小多种多样，但顶点UV坐标的范围通常都被归一化到【0-1】范围内。
