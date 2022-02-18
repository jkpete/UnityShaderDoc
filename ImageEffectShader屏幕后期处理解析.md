# 屏幕后期处理效果

在渲染完整个场景之后，得到的屏幕图像进行了一系列操作，最终实现各种屏幕特效，如Bloom，SSAO等等。

要实现屏幕后期处理的基础是，必须得到渲染后的屏幕图像，即抓取屏幕，而Unity为我们提供了这样一个方便的接口---OnRenderImage函数。



## 写在前面

简单介绍几个常用名词。

### 0.1 UV

U和V分别是图片在显示器水平、垂直方向上的坐标，取值范围是0~1。通常是从左往右，从下往上从0开始依次递增。





## 1.基础类

在进行屏幕后期处理之前，我们需要检查一系列条件是否满足，例如当前平台是否支持渲染纹理跟屏幕特效。是否支持当前使用的shader等。为此，我们创建了一个用于屏幕后期处理效果的基类。在实现各种屏幕特效时，我们只需继承基础类。在实现派生类中不同操作即可。

```csharp
using UnityEngine;
using System.Collections;

[ExecuteInEditMode]
[RequireComponent (typeof(Camera))]
public class PostEffectsBase : MonoBehaviour {

	// 开始检查
	protected void CheckResources() {
		bool isSupported = CheckSupport();
		
		if (isSupported == false) {
			NotSupported();
		}
	}

	// 在检查是否支持屏幕后期shader的时候调用
	protected bool CheckSupport() {
		if (SystemInfo.supportsImageEffects == false || SystemInfo.supportsRenderTextures == false) {
			Debug.LogWarning("This platform does not support image effects or render textures.");
			return false;
		}
		
		return true;
	}

	// 在不支持屏幕后期shader的时候调用。
	protected void NotSupported() {
		enabled = false;
	}
	
	protected void Start() {
		CheckResources();
	}

	// 在需要通过该屏幕后期shader创建相应材质的时候调用。
	protected Material CheckShaderAndCreateMaterial(Shader shader, Material material) {
		if (shader == null) {
			return null;
		}
		
		if (shader.isSupported && material && material.shader == shader)
			return material;
		
		if (!shader.isSupported) {
			return null;
		}
		else {
			material = new Material(shader);
			material.hideFlags = HideFlags.DontSave;
			if (material)
				return material;
			else 
				return null;
		}
	}
}

```

基础类使用案例(继承PostEffectBase)

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class ImageEffect_Scanner : PostEffectsBase{
    //
    public Shader scannerShader;
    private Material scannerMaterial;
	public Material material
	{
		get
		{
            // 检查是否生成对应材质
			scannerMaterial = CheckShaderAndCreateMaterial(scannerShader, scannerMaterial);
			return scannerMaterial;
		}
	}
	public float scannerScale = 1.0f;
	public new void Start() {
		//如果需要提前规定Camera获取深度纹理或者法线纹理，需要提前声明
        //GetComponent<Camera>().depthTextureMode = DepthTextureMode.DepthNormals;
	}

	void OnRenderImage(RenderTexture src, RenderTexture dest)
	{
		//GetComponent<Camera>().depthTextureMode = DepthTextureMode.DepthNormals;
		if (material != null)
		{
			material.SetFloat("_ScannerScale", scannerScale);
			Graphics.Blit(src, dest, material);
		}
		else
		{
			Graphics.Blit(src, dest);
		}
	}
}


```

## 2.使用渲染纹理

在Unity中，获取深度纹理是非常简单的，直接在脚本中设置摄像机的depthTextureMode，就可以获取对应的纹理数据。

深度纹理获取

```csharp
GetComponent<Camera>().depthTextureMode = DepthTextureMode.Depth;
```

法线纹理+深度纹理获取：

```csharp
GetComponent<Camera>().depthTextureMode = DepthTextureMode.DepthNormals;
```
