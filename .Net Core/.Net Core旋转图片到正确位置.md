今天开发群里一位大佬,发现他的图片用 Bitmap 处理后被旋转了 90°,于是勾起了我们的好奇,但是我本人在电脑上试了好几张图都没有问题.后来才发现
iPhone 等手机上传图片到服务器后,通常需要进行旋转处理,否则在进行图片压缩,缩放处理后会丢失正确的位置信息,导致显示的图片不处于正确的位置上.
处理的做法就是读取照片的 Exif 信息,并旋转到正确位置.代码如下:

```csharp
/// <summary>
/// 将图片旋转到正确位置
/// </summary>
/// <param name="image"></param>
/// <returns></returns>
public static void OrientationImage(Image image)
{
    if (Array.IndexOf(image.PropertyIdList, 274) > -1)
    {
        var orientation = (int)image.GetPropertyItem(274).Value[0];
        switch (orientation)
        {
            case 1:
                // No rotation required.
                break;
            case 2:
                image.RotateFlip(RotateFlipType.RotateNoneFlipX);
                break;
            case 3:
                image.RotateFlip(RotateFlipType.Rotate180FlipNone);
                break;
            case 4:
                image.RotateFlip(RotateFlipType.Rotate180FlipX);
                break;
            case 5:
                image.RotateFlip(RotateFlipType.Rotate90FlipX);
                break;
            case 6:
                image.RotateFlip(RotateFlipType.Rotate90FlipNone);
                break;
            case 7:
                image.RotateFlip(RotateFlipType.Rotate270FlipX);
                break;
            case 8:
                image.RotateFlip(RotateFlipType.Rotate270FlipNone);
                break;
        }
        image.RemovePropertyItem(274);
    }
}
```

![](https://img2020.cnblogs.com/blog/696130/202112/696130-20211215192345571-139014001.png)
