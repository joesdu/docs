- 该算法来源于小程序开发社区一位大佬帮我搞的.

```typescript
/**
 * 获取网络SVG图片宽高
 * @param src SVG网络资源链接
 */
public static GetSvgInfoFormUrl = (src: string): Promise<any> =>
  new Promise((resolve, reject) => {
    request({ url: src }).then((res: any) => {
      const { data } = res;
      // <svg height="" width=""
      const r1: RegExp = /(?:[\S\s]+)?<svg(?=.*(height)="(\d+)(?:[^"]+)?")(?=.*(width)="(\d+)(?:[^"]+)?")(?:[\S\s]+)?/;
      // <svg viewBox="0 0 1103 711"
      const r2: RegExp = /(?:[\S\s]+)?viewBox="\d+ \d+ (\d+) (\d+)"(?:[\S\s]+)?/;
      // <svg style="height:***px;width:***px"
      const r3: RegExp = /(?:[\S\s]+)?(?:<svg[^>]+style="(?=.*(height):(\d+)(?:[^>]+)?)(?=.*(width):(\d+)(?:[^>]+)?))(?:[\S\s]+)?/;
      let str: string = '{}';
      let info: { width: number; height: number } = { width: 0, height: 0 };
      let isSuccess: boolean = false;
      try {
        str = data
          .replace(r1, '{"$1":$2,"$3":$4}')
          .replace(/\$(2|4)/, '0')
          .replace(/\$1/, 'height')
          .replace(/\$3/, 'width');
        if (str.length > 45) isSuccess = false;
        else {
          isSuccess = true;
          info = JSON.parse(str);
        }
        if (!isSuccess) {
          str = data.replace(r2, '{"width":$1,"height":$2}').replace(/\$(1|2)/, '0');
          if (str.length > 45) isSuccess = false;
          else {
            isSuccess = true;
            info = JSON.parse(str);
          }
          console.log('r2', info);
        }
        if (!isSuccess) {
          str = data
            .replace(r3, '{"$1":$2,"$3":$4}')
            .replace(/\$(2|4)/, '0')
            .replace(/\$1/, 'height')
            .replace(/\$3/, 'width');
          if (str.length > 45) isSuccess = false;
          else {
            isSuccess = true;
            info = JSON.parse(str);
          }
          console.log('r3', info);
        }
      } catch (error) {
        console.log(error);
        reject(error);
      }
      if (isSuccess) resolve(info);
      else reject(new Error('无法正确解析SVG数据'));
    });
  });
```
