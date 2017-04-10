# ESLint 执行规则

ESLint 支持几种格式的配置文件，如果同一目录下有多个配置文件，ESLint 只会使用一个，优先级顺序如下：

1. Javascript：使用 .eslintrc.js 输出的配置对象
2. YAML：使用 .eslintrc.yaml 或 .eslintrc.yml 去定义配置的结构
3. JSON：使用 .eslintrc.json 去定义配置的结构
4. Deprecated：使用 .eslintrc，可以是 JSON 也可以是 YAML
5. package.json：package.json 中定义的 eslintConfig 属性，在 eslintConfig 中配置