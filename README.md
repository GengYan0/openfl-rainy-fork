# rainy-openfl

OpenFL 9.5.2 的 Rainy 构建。渲染层与 Stage3D 后端的改动，`openfl` 的公开 API 全部保留。

## 能力一览

| 能力 | 入口 | 默认 |
|---|---|---|
| 8 种扩展混合模式 | `openfl.display.BlendMode` | — |
| Stage3D GL 状态影子 | 自动 | 开 |
| 渲染器状态缓存自愈 | 自动 | 开 |
| 帧缓冲深度纹理附件 | `TextureBase.depthTextureEnabled` | `false` |
| CPU 侧填充 | `BitmapData.__cpuFill` | 自动按路径选择 |

## 安装

```bash
haxelib dev openfl "D:/libs/rainy-openfl"
```

或者：

```bash
haxelib git openfl https://github.com/GengYan0/openfl-rainy-fork
```

配套 `flixel` 与 `lime` 见各自仓库。三者要一起用，单换一个不会坏，但拿不到跨层收益。

## 1. 扩展混合模式

`BlendMode` 新增 8 个值。`EXCLUSION` 与 `LINEAR_BURN` 走固定管线；其余七种需要背景与
前景两张纹理同趟采样，由 `FlxCamera` 的图层合成处理（见 `rainy-flixel`）。

| 值 | 字面量 | 需要图层合成 |
|---|---|---|
| `EXCLUSION` | `"exclusion"` | 否 |
| `COLOR_DODGE` | `"colordodge"` | 是 |
| `COLOR_BURN` | `"colorburn"` | 是 |
| `SOFT_LIGHT` | `"softlight"` | 是 |
| `HUE` | `"hue"` | 是 |
| `SATURATION` | `"saturation"` | 是 |
| `COLOR` | `"color"` | 是 |
| `LUMINOSITY` | `"luminosity"` | 是 |

数值形式与字符串形式都认：

```haxe
sprite.blend = BlendMode.SOFT_LIGHT;
graphics.__blendMode = BlendMode.fromString('soft-light');
trace(BlendMode.toString(BlendMode.COLOR_BURN));   // "colorburn"
```

字符串解析兼容 `colorburn` / `color_burn` / `color-burn` 三种写法，配置文件和谱面事件里
三种都见过。

## 2. Stage3D GL 状态影子

`Context3D` 原来每次 `drawTriangles` 都重绑一次采样器纹理，即使槽位上已经绑着同一张。
缓存之后只在以下情况才真正下发：

- 槽位为空或存的是别的纹理；
- 该槽位被绕开管线的路径动过（纹理上传、视频帧、直接绑定）。这些路径会置
  `glSamplerStale[i]`，刷新时先检查它；
- 上下文被别的渲染器接管。

AGAL 的 alpha 采样器（`sampler + 4`）在「纹理未变」的分支上也照样绑定。原来这里被跳过，
结果是只换 alpha 纹理而彩色纹理不动时 alpha 还是旧的，关掉 `textureAlpha` 后
「已启用」uniform 仍停在 1。

视口同样有影子，`OpenGLRenderer.setViewport` 走它。直接写 `gl.viewport` 会让 Stage3D
以为旧矩形还在，下一次想用那个矩形的绘制就跳过写入，整帧落到错误区域。

```haxe
// 绕过 Context3D 直接改 GL 状态后，让影子失效
Context3D.__invalidateGLCache();
```

## 3. 渲染器状态缓存自愈

一块 GL 上下文被多个 `OpenGLRenderer` 实例轮流用时（离屏渲染、`BitmapData.draw`、
图层合成都这样），后写入的一方会覆盖前者的混合状态，前者下一次绘制就带着别人的状态画的。

现在在状态写入点自己发现：

```haxe
@:noCompletion inline function __claimContext():Void
{
	if (__contextOwner == this) return;
	__contextOwner = this;
	__resetStateCache();
}
```

`__setBlendMode` / `setShader` / `__render` 三个入口先认领上下文。`BitmapData.draw` 一定走
自己的 renderer 实例，控制权交接就变成「owner 变了」，一次引用比较即自愈。

`resetStageStateCache()` 保留为公开 API，用途收窄成「绕过 renderer 直接改 GL 状态」：

```haxe
OpenGLRenderer.resetStageStateCache();
```

## 4. 帧缓冲深度纹理附件

默认用 renderbuffer 存深度。需要之后采样深度（SSAO、软粒子、深度雾）时打开纹理附件：

```haxe
TextureBase.depthTextureEnabled = true;

var target = new Texture(1024, 1024, false, 1, Context3DTextureFormat.BGRA, true);
var depth:GLTexture = target.getDepthTextureID();   // 0 表示没拿到
```

附件挂上后会做帧缓冲完整性检查，驱动不接受的格式自动退回 renderbuffer，不会画出空白。
`depthTextureEnabled` 是静态开关，同一进程内所有 `Texture` 共享这个选择。

## 5. CPU 侧填充

`BitmapData.fillRect` 走 CPU 缓冲，不再用带裁剪的帧缓冲清除。后者每次要切两次渲染目标，
UI 一帧重绘几百个小矩形时代价明显。

GPU 副本在下次绑定时按 `image.version` 变化重传，`ImageDataUtil.fillRect` 会推进版本，
所以两边保持一致。

```haxe
bitmap.fillRect(new Rectangle(0, 0, 100, 40), 0x00000000);
```

## 边界

- 深度纹理与 renderbuffer 二选一，同一个 `Texture` 上不能来回切。
- 图层合成模式在 `FlxCamera.layerBlendEnabled = false` 时退回普通 source-over，
  视觉上等于颜色变淡，不是出错。
- 状态影子只覆盖 `Context3D` 与 `OpenGLRenderer` 已知的写入口。新增自定义 GL 直调要自己
  调 `__invalidateGLCache()` 或 `resetStageStateCache()`。

改动逐条记录在 `RAINY-FORK.md`。
