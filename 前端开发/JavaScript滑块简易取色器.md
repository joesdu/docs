该滑块取色器适用于对色彩精度要求不严格的场景.
这个东西的实现原理,是将滑块分为 100 等分.步进为 1,将背景按照色值分成 10 分.每一个渐变过程分为 10 个 step.
主要源码如下.

- 滑块的背景样式:

```css
background-image: linear-gradient(
  90deg,
  #ff0000 0%,
  #ff9a00 10%,
  #ccff00 20%,
  #33ff00 30%,
  #00ff67 40%,
  #00ffff 50%,
  #0066ff 60%,
  #3300ff 70%,
  #cc00ff 80%,
  #ff0099 90%,
  #ff0000 100%
);
```

- 色彩处理的 JavaScript 算法:

```javascript
/**
 *   颜色渐变
 *   num：颜色个数
 */
function fColorGradualChange(startColor, endColor, num) {
  let rgb =
    /^rgb|RGB\((([0-9]|[1-9]\d|1\d\d|2([0-4]\d|5[0-5])),){2}([0-9]|[1-9]\d|1\d\d|2([0-4]\d|5[0-5]))\)$/; //rgb
  let hex = /(^#[0-9A-F]{6}$)|(^#[0-9A-F]{3}$)/i; //16进制
  //颜色预处理
  let startRGB, endRGB;
  if (hex.test(startColor)) {
    startRGB = fAnalysisRGB(startColor);
  } else if (rgb.test(startColor)) {
    startRGB = startColor.substring(3, 15).split(",");
  }
  if (hex.test(endColor)) {
    endRGB = fAnalysisRGB(endColor);
  } else if (rgb.test(startColor)) {
    endRGB = endColor.substring(3, 15).split(",");
  }
  let startR = startRGB[0],
    startG = startRGB[1],
    startB = startRGB[2];
  let endR = endRGB[0],
    endG = endRGB[1],
    endB = endRGB[2];
  let sR = (endR - startR) / num;
  let sG = (endG - startG) / num;
  let sB = (endB - startB) / num;
  let colors = [];
  for (let i = 0; i < num; i++) {
    colors.push(
      fColorToHex(
        parseInt(sR * i + startR),
        parseInt(sG * i + startG),
        parseInt(sB * i + startB)
      )
    );
  }
  return colors;
}
/**
 *   解析rgb格式
 */
function fAnalysisRGB(temp) {
  temp = temp.toLowerCase().substring(1, temp.length);
  let colors = [];
  colors.push(parseInt(temp.substring(0, 2), 16));
  colors.push(parseInt(temp.substring(2, 4), 16));
  colors.push(parseInt(temp.substring(4, 6), 16));
  return colors;
}
/**
 *   rgb转hex
 */
function fColorToHex(r, g, b) {
  return (
    "#" +
    fAddZero(r.toString(16)) +
    fAddZero(g.toString(16)) +
    fAddZero(b.toString(16))
  );
}
/**
 *   加0补位
 */
function fAddZero(v) {
  let newv = "00" + v;
  return newv.substring(newv.length - 2, newv.length);
}

function getColor(value) {
  let step = 10;
  let colorZone = parseInt(value / step);
  let index = parseInt(value % step);
  let s = "";
  switch (colorZone) {
    case 0:
      s = fColorGradualChange("#FF0000", "#FF9A00", step);
      return s[index];
    case 1:
      s = fColorGradualChange("#FF9A00", "#CCFF00", step);
      return s[index];
    case 2:
      s = fColorGradualChange("#CCFF00", "#33FF00", step);
      return s[index];
    case 3:
      s = fColorGradualChange("#33FF00", "#00FF67", step);
      return s[index];
    case 4:
      s = fColorGradualChange("#00FF67", "#00FFFF", step);
      return s[index];
    case 5:
      s = fColorGradualChange("#00FFFF", "#0066FF", step);
      return s[index];
    case 6:
      s = fColorGradualChange("#0066FF", "#3300FF", step);
      return s[index];
    case 7:
      s = fColorGradualChange("#3300FF", "#CC00FF", step);
      return s[index];
    case 8:
      s = fColorGradualChange("#CC00FF", "#FF0099", step);
      return s[index];
    case 9:
      s = fColorGradualChange("#FF0099", "#FF0000", step);
      return s[index];
    case 10:
      return "#FF0000";
  }
}
module.exports = {
  GetColor: getColor,
};
```

上边的 JS 代码借鉴了网络上大佬的算法.只是自己新增了点,这样一来问题就很简单了.
在滑块的滑动事件中调用 getColor 函数并传入当前滑块的值就能获取到相应的色值了.
理论上该函数支持更多的色值,将滑块的最大值设为更大的数值,将 getColor 函数中的 step 值设置为最大值的对 10 的商值就行了.
