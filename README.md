今天接到一个需求，我朋友在网上下载了一堆`webp`图片，想请我转成`gif`图。

这种需求肯定是第一时间在网上找现成的工具给她用啦，但是找了一轮之后，不是有数量限制就是要开通会员。

我想着这个应该也不难，就打算手写一个程序来处理。

# 思路
解决问题的第一步，就是先上网查查有没有现成的处理webp转成gif的代码库。

在`GitHub`上逛了一圈没有发现js版本的`webp`转`gif`的工具库。

那第二步就是先降低难度，有没有进行图片格式转换的js库或者代码示例。功夫不负有心人，在`CDN`上找到一篇进行图片格式转换的文章。

文章的大体思路是：通过`input`元素获取到图片文件 -> 用`canvas`将图片绘制出来 -> 将`canvas`转换为对应格式的`blob` -> 触发下载。

# 第一版代码
根据这个思路，我很快写出第一版的代码：

```html
<input type="file" id="fileInput" multiple />
```

``` ts
      // 入口函数
      function main() {
        const input = document.getElementById("fileInput");
        const type = {
          mimeType: "image/gif",
          ext: ".gif",
        };
        input.addEventListener("input", (event) => {
          const files = event.target.files;
          if (files.length < 1) {
            return;
          }
          for (const file of files) {
            convert(file, type); // 处理文件
          }
        });
      }
      
      // 处理文件
      async function convert(file, type) {
        const image = await readImage(file); // 读取文件
        const canvas = await imgToCanvas(image); // 绘制
        const blob = await canvasToBlob(canvas, type); // 存储为新文件
        const fileName = file.name.split(".").shift();
        saveBlob(blob, fileName, type); // 触发下载
      }

      // 读取文件
      function readImage(file) {
        // 返回Promise方便处理异步
        return new Promise((resolve, reject) => {
          // 构建文件读取器
          const reader = new FileReader();
          reader.onload = () => {
            const image = new Image();
            image.src = reader.result;
            // 返回图片元素
            resolve(image);
          };
          // 读取为 DataURL
          reader.readAsDataURL(file);
        });
      }

      // 图片转换为canvas
      function imgToCanvas(image) {
        // 创建canvas
        const canvas = document.createElement("canvas");
        canvas.width = image.width;
        canvas.height = image.height;
        // 绘制图片
        canvas.getContext("2d").drawImage(image, 0, 0);
        // 返回canvas
        return canvas;
      }

      // canvas转文件
      function canvasToBlob(canvas, type) {
        // 返回Promise方便处理异步
        return new Promise((resolve, reject) => {
          // 转成文件
          canvas.toBlob((blob) => {
            // 返回
            resolve(blob);
          }, type.mimeType);
        });
      }
      
      // 触发下载
      function saveBlob(blob, fileName, type) {
        // 构建一个url
        const url = URL.createObjectURL(blob);
        // a标签
        const linker = document.createElement("a");
        linker.href = url;
        linker.download = fileName + type.ext;
        document.body.appendChild(linker);
        // 触发下载
        linker.click();
        // 垃圾回收
        document.body.removeChild(linker);
        URL.revokeObjectURL(url);
      }
```

还剩一个逻辑需要处理，就是**打包下载**。

# 打包下载

首先添加`JSZip`包和`file-saver`包，`JSZip`负责将多个文件打包成一个`zip`包，`file-saver`负责文件的下载。添加一个`save`函数代替`saveBlob`函数。

``` ts
import JSZip from 'jszip'
import { saveAs } from 'file-saver'

const zip = new JSZip()

...

function main() {
  const input = document.getElementById("fileInput");
  const type = {
    mimeType: "image/gif",
    ext: ".gif",
  };
  input.addEventListener("input", (event) => {
    const files = event.target.files;
    if (files.length < 1) {
      return;
    }
    for (const file of files) {
      const { fileName, blob } = convert(file, type); // 处理文件
      zip.file(fileName, blob) // 添加文件到zip
    }
    save() // 保存
  });
}

// 处理文件
async function convert(file, type) {
    const image = await readImage(file); // 读取文件
    const canvas = await imgToCanvas(image); // 绘制
    const blob = await canvasToBlob(canvas, type); // 存储为新文件
    const fileName = file.name.split(".").shift();
    // saveBlob(blob, fileName, type); // 触发下载
    return { fileName, blob }
}

...

// 保存
function save() {
  zip.generateAsync({type: 'blob'})
      .then((content) => {
        saveAs(content, 'results.zip')
      })
}
```

这样就解决了图片格式转换的问题。

# 处理webp格式

但是这还没完，因为`webp`格式是支持动图格式的，我一开始没发现，但是我朋友告诉我转换后的图片不会动了。

也是，这样仅仅绘制一个图片然后导出怎么可能会动嘛。

接着我继续使出`GitHub`大法，甚至翻看了`webp`的官网，可惜`webp`官网只提供了`c语言`的**开发包**，以及**命令行工具**。好在功夫不负有心人，我在GitHub上找到了一个`webp`的编解码工具：[WebPXMux.js](https://github.com/sumimakito/webpxmux.js), 它可以处理`webp`的解码，将`webp`动图转成一帧一帧的格式。虽然不能一步到位解决`webp`到`gif`的转换，但是按照上面的思路，先将`webp`转换成一帧一帧的图片，再将这些图片再合成一个`gif`图不就行了？

怎么将多帧的图片转换成`gif`图呢？这个相对简单，这种需求肯定有大把的解决方案。最终我选择了[gif.js](https://github.com/jnordberg/gif.js/tree/master)这个库。

通过这两个工具，解题思路就变成了：通过`input`元素获取到图片文件 -> 将图片文件使用`WebPXMux`转换成多帧图片 -> 用`canvas`将每一帧图片绘制出来 -> 将多帧图片通过`gif.js`合成为一张`gif`图片 -> 触发下载。


# 第二版代码

在原版基础上添加以下代码：

首先创建解码器。
``` ts
const xMux = WebPXMux("./webpxmux.wasm")
```
解码每一个`webp`文件。
``` ts
// 解码
async function decode(file: File) {
  await xMux.waitRuntime()
  const data = await fileReader(file)
  const frames = await xMux.decodeFrames(data)
  return frames
}
```
绘制**帧序列**
``` ts
function drawFrames(frames: Frames, fileName: string) {
  if (frames.frameCount < 1) { return }
  
  return new Promise((resolve) => {
    const height = frames.height
    const width = frames.width
    
    // 这里增加了一个canvas工厂用于处理canvas创建的问题
    const canvas = canvasFactory(width, height)
    const ctx = canvas.getContext("2d")!

    // 创建一个gif对象
    const gif = new GIF({
      workerScript: "./gif.worker.js", // 需要设置worker.js来进行异步处理
      workers: 2, // worker线程的数量
      quality: 10, // gif图片的质量
      width,
      height,
    })
    
    // 先通过canvas绘制每一帧
    for (let i = 0; i < frames.frameCount; i++) {
      // 取一帧
      const frame = frames.frames[i]
      // 清屏
      ctx.clearRect(0, 0, width, height)
      
      // 获取图片背景色，这里提供了一个函数来解析颜色值
      const {r,g,b,a} = getRGBA(frames.bgColor)
      // 设置填充色
      ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a})`
      // 绘制背景色
      ctx.fillRect(0, 0, width, height)
      // 绘制一帧
      drawFrame(frame, width, height, ctx)
      // 将这一帧加入到gif对象中
      gif.addFrame(ctx, { copy: true, delay: i === 0 ? 0.00000001 : frame.duration })
    }
    // 设置完成时间
    gif.on('finished', function (blob: Blob) {
      resolve({
        fileName,
        blob,
      })
    })
    // 生成一个gif图片
    gif.render()
  })
}
```

这一段信息量比较大，我简单解释一下：

`xMux`是一个`webp`编解码对象，`xMux.decodeFrames`可以将`FileReader`读取到的数据(也就是`webp`的数据)进行解码。它会返回一个`Frames`对象，对象的结构是这样的：

``` ts
// 帧序列
interface Frames {
  frameCount: number; // 帧数量
  width: number;
  height: number;
  loopCount: number; // 每次播放的循环数
  bgColor: number; // 背景色
  frames: Frame[]; // 帧数组
}
```
``` ts
// 帧对象
interface Frame {
  duration: number; // 该帧持续时间
  isKeyframe: boolean; // 是否为关键帧
  rgba: Uint32Array; // 位图信息
}
```
一个`Frames`对象包含一个帧序列的相关信息，这里最关键的是背景色和位图信息是怎么样的。

位图信息是一个`Uint32Array`对象，简单来说就是包含`Uint32`的数组，也就是`32位无符号整数`的数组。背景色也是一个`32位无符号整数`。按顺序`0xRRGGBBAA`排列，即前8位为**红色**的色值信息；9-16位为**绿色**的色值信息；17-24位为**蓝色**的色值信息；25-32位为**透明度**的色值信息。

所以我构建了一个辅助函数来解析一个`32位无符号整数`，将其转换为`rgba`各值：
``` ts
// 获取RGBA值
function getRGBA(uint32: number): {r: number, g: number, b: number, a: number} {
  const r = (uint32 & (0xFF << 24)) >>> 24
  const g = (uint32 & (0xFF << 16)) >>> 16
  const b = (uint32 & (0xFF << 8)) >>> 8
  const a = (uint32 & (0xFF)) / 255.0

  return {
    r, g, b, a
  }
}
```
上面这段代码相信大家都能看懂，我就不详细展开了。（不懂的回去复习一下**位运算**的相关知识）

然后由于`xMux`解码出来的是位图信息，所以需要循环绘制每一位的颜色：

``` ts
function drawFrame(frame: Frame, width: number, height: number, ctx: CanvasRenderingContext2D) {
  for (let y = 0; y < height; y++) {
    for (let x = 0; x < width; x++) {
      const bitColor = frame.rgba[x + y * width]
      const {r, g, b, a} = getRGBA(bitColor)
      ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a})
      ctx.fillRect(x, y, 1, 1)
    }
  }
}
```
改一下`convert`函数：
``` ts
async function convert(file: File) {
  const frames = decode(file)
  const fileName = file.name.split(".").shift()!
  const blob = await drawFrames(frames)
  return { fileName, blob }
}
```

最后优化一下`main`函数：

```js
function main() {
  const input = document.getElementById("fileInput");
  const type = {
    mimeType: "image/gif",
    ext: ".gif",
  };
  input.addEventListener("input", async (event) => {
    const files = event.target.files;
    if (files.length < 1) {
      return;
    }
    
    await Promise.all(Array.from(files).map((file: File) => {
      const { fileName, blob } = await convert(file, type); // 处理文件
      zip.file(fileName, blob) // 添加文件到zip
    }))
    save() // 保存
  });
}
```

使用这两个库还有几点需要注意：
1. `WebPXMux`库还依赖了`ws`这个库，所以需要一同install。
2. 创建`xMux`对象的时候需要使用到`webpxmux.wasm`文件，这个文件`WebPXMux`库中自带了，但是在浏览器端需要放置在一个外部能够访问到的地方，我将这个文件从`webpxmux/dist`中拷贝了一份到本项目的`public`目录中；
3. `gif.js`需要使用`gif.worker.js`来启动`Worker`，我也从`gif.js/dist`目录拷贝了一份到本项目的`public`目录中。

至此，基本算大功告成。只是还留有两个问题：

1. 生成的`gif`图片体积太大；
2. 因为`canvas`绘图api的问题，无法绘制**透明底**；

第一个问题应该可以通过`gif.js`设置`gif` **质量(quality)** 这个参数进行调整，但是朋友他想要更高质量的图片，可以稍微忍受一点体积的问题。

第二个问题我查了MDN关于`canvas`的`api`，目前没有可以设置透明底的方法，或者说有但我没找到。网友的建议是使用`webgl`来绘制，`webgl`支持透明底设置。

那等我过两天学完`webgl`再来处理吧。

由于代码量较多，项目我将会在 `GitHub` 上开源，查[这个地址](https://github.com/Tz-george/webp2gif_demo)即可查看项目的完整代码。