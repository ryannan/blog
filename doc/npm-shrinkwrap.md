# npm shrinkwrap 管理项目依赖

在使用 npm 包管理时，我们经常需要安装和升级对应项目的依赖版本，造成项目版本的不确定性，主要的场景：

* 在开发阶段执行的版本和后续测试、生产部署得到的版本可能不一样。
* 项目组新来的同事安装的对应的版本依赖版本也可能不一样。
* npm 安装包是有层次结构的，通过 semver 手动控制依赖版本，但是只能精确控制你的直接依赖包，多层依赖无法控制。
* 项目组中多个项目应用使用的共用库的版本不统一。

这个时候，可以使用 npm shrinkwrap 来管理你的项目依赖。

### shrinkwrap

`npm shrinkwrap` 是按照当前项目 node_modules 目录内的安装包情况生成的配置文件（npm-shrinkwrap.json）。在项目中执行 `npm install` 时，npm 会检查在根目录下是否存在 npm-shrinkwrap.json 文件，如果存在，npm 会使用它而不是 package.json 来安装各个包的版本依赖。这样可以确保你安装的依赖版本是固定的。

比方说，有一个包 A

```
{
  "name": "A",
  "version": "0.1.0",
  "dependencies": {
  	"B": "<0.1.0"
  }
}
```

还有一个包 B

```
{
  "name": "B",
  "version": "0.0.1",
  "dependencies": {
  	"C": "<0.1.0"
  }
}
```

以及 B 所依赖的 C

```
{
  "name": "C",
  "version": "0.0.1"
}
```

执行 npm shrinkwrap，得到的 npm-shrinkwrap.json 如下：

```
{
  "name": "A",
  "version": "0.1.0",
  "dependencies": {
    "B": {
      "version": "0.0.1",
      "dependencies": {
        "C": {
          "version": "0.0.1"
        }
      }
    }
  }
}
```

