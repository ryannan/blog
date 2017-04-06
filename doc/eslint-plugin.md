# Eslint Plugin 开发

> 项目地址：https://github.com/ryannan/eslint-plugin-catchlogs

### 开发流程

* 安装脚手架

  ```
  npm i -g yo
  npm i -g generator-eslint
  ```

* 安装 plugin

  ```
  yo eslint:plugin
  ```

* 安装 rule

  ```
  yo eslint:rule
  ```

* 开发对应的 rule

  在 lib/rules/ 下的规则文件进行开发，meta 中配置 rule 对应的配置信息，create 返回的是一个 AST 抽象状态树对象，执行时，会将 js 解析成 AST 抽象状态对象，比如 

  ```
  import lodash from 'lodash';
  ```

  在 rules 中会解析成对应的 ImportDeclaration 声明语句。在对应的声明语句中完成你需要开发的规则。eslint plugin 中提供的 context 对 AST 节点进行了一些封装，比如获取源码、节点位置等。可参考 eslint 默认提供的规则配置的[源码](https://github.com/eslint/eslint/tree/master/lib/rules)。

* 测试用例

  在 tests/lib/rules/ 下的规则文件进行测试用例的编写，invalid 中需要配置errors 对应的 message，如果 code 中有 es6 语法，需要配置 parserOptions 来进行转义。

* 运行

  ```
  npm test
  ```

  根据你的测试用例输出对应的结果，有效、无效规则都是验证通过，无效通过需要 report 对应的错误信息。

  ​

### 资料

* [eslint 脚手架](https://github.com/eslint/generator-eslint)
* [eslint plugin 开发指南](http://eslint.cn/docs/developer-guide/working-with-rules)
* [eslint 默认规则源码](https://github.com/eslint/eslint/tree/master/lib/rules)