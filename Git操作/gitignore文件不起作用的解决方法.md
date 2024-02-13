### .gitignore 文件不起作用的解决方法

> 项目开发中使用 Git 作为版本管理工具时,有时并非在项目一开始就添加了.gitignore 文件来管理 Git 忽略规则,或是在项目开发过程中添加或移除了忽略规则,这时由于 Git 在本地维护着一份遵从创建本地项目时的 gitignore 规则的 Git 缓存,因此会造成.gitignore 文件不起作用的现象.

解决这个问题的方式就是清除掉本地项目的 Git 缓存,通过重新创建 Git 索引的方式来生成遵从新 .gitignore 文件中规则的本地 Git 版本,再将该 Git 版本提交到主干.

例如,如果先使用 Yarn 等工具创建了项目,然后才创建了如下内容的.gitignore 文件,那么在提交时本地 Git 并不会忽略相应的文件和路径.

# .gitignore

```text
.DS_Store
node_modules
package-lock.json
**/.yarn/cache
```

此时,可在 VSC、IDEA 等项目 IDE 的终端工具,或 iTerm、PowerShell 等独立的终端应用中通过如下 Git 命令解决这个问题：

- 0.进入项目路径
- 1.清除本地当前的 Git 缓存

```bash
git rm -r --cached .
```

- 2.应用.gitignore 等本地配置文件重新建立 Git 索引

```bash
git add .
```

- 3.(可选)提交当前 Git 版本并备注说明

```bash
git commit -m 'update .gitignore'
```

需要注意的是,第三步仅在调整过.gitignore 文件的设备上进行即可；其它设备可以选择重新 clone,或在 pull 之后执行前两步.
