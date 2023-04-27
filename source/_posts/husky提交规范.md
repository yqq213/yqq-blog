---
title: 使用git hooks工具husky结合eslint实现提交规范
date: 2021-08-22 20:10:33
categories: [Git]
tags: git
---

在团队协作开发项目时，如果每个人的提交规范不一致，或者提交时代码有错误，那么在定位git提交记录或者review代码时将会是一团糟，因此我们可以使用husky结合eslint来实现git提交规范化，保证错误的commit不能提交成功。

# commitlint

- 什么是commitlint

  当我们运行`git commit -m 'xxx'`时，commitlint是用来检查提交信息是否满足固定格式的工具

- 为什么使用commitlint

  团队中规范了 commit 可以更清晰的查看每一次代码提交记录，还可以根据自定义的规则，自动生成 changeLog 文件。

- commitlint安装

  ```
  npm install @commitlint/config-conventional @commitlint/cli -D
  ```

- 生成配置文件`commitlint.config.js`

  ```
  echo "module.exports = {extends: ['@commitlint/config-conventional']};" > commitlint.config.js
  ```

  文件配置如下：

  ```javascript
  'module.exports = {extends: [\'@commitlint/config-conventional\']}'
  module.exports = {
    extends: ['@commitlint/config-conventional'],
    rules: {
      'type-enum': [
        2,
        'always',
        [
          'feat',	// 新功能
          'upd',
          'del',
          'fix',	// 修复bug
          'docs',	// 文档修改
          'style',	// 代码格式修改
          'refactor',	// 代码重构
          'test',		// 测试用例修改
          'chore',	// 其他修改, 比如改变构建流程、或者增加依赖库、工具等
          'revert',	// 回滚到上一个版本
        ],
      ],
      'subject-full-stop': [0, 'never'],
      'subject-case': [0, 'never'],
      // 'space-before-function-paren': 0,
    },
  }
  ```




# husky

husky是git hooks工具，作用是对git执行的一些命令，通过对应的hooks钩子触发，执行自定义的脚本程序

```
npm install husky --save-dev
```

在package.json中引入husky

```json
"husky": {
  "hooks": {
  	"commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
  	"pre-commit": "lint-staged"
  }
}
```

这段配置告诉了git hooks，当我们在当前项目中执行 `git commit -m '测试提交'` 时将触发`commit-msg`事件钩子并通知husky，从而执行 `commitlint -E HUSKY_GIT_PARAMS`命令，也就是我们刚开始安装的commitlint，它将读取`commitlint.config.js`配置规则并对我们刚刚提交的测试提交这串文字进行校验，若校验不通过，则在终端输出错误，commit终止。并且提交时还会触发`pre-commit`命令，执行eslint检查。

> 如果想跳过钩子检查，也可以在commit时在后面加上`--no-verify`来跳过，但是并不推荐这样做



# lint-staged

- `lint-staged` 可以让你当前的代码检查**只检查本次修改更新的代码**
- `lint-staged` 无需安装，生成项目时，vue-cli 已经帮我们安装了

需要在`package.json`中添加如下配置，作用是匹配暂存区所有的js、jsx、vue文件，并执行命令

```json
"lint-staged": {
  "*.{js,jsx,vue}": [
  	"npm run lint",
  	"git add"
  ]
 }
```

每次它在你本地 commit 之前，校验你所提的内容是否符合你本地配置的 eslint 规则，如果符合规则，则提交成功，如果不符合，则会终止提交，待修复完成后才可提交



# .eslintrc.js

最后附上eslint的配置文件，如下：

```javascript
module.exports = {
  env: {
    browser: true,
    es6: true,
  },
  extends: 'plugin:vue/essential',
  globals: {
    Atomics: 'readonly',
    SharedArrayBuffer: 'readonly',
  },
  parser: 'vue-eslint-parser',
  parserOptions: {
    ecmaVersion: 2018,
    sourceType: 'module',
    parser: 'babel-eslint',
  },
  plugins: ['vue'],
  rules: {
    'no-console': 0,
    'no-alert': 1, //提示alert
    'no-var': 2, //禁用var，用let和const代替
    'no-constant-condition': 2, //禁止在条件中使用常量表达式 if(true) if(1)
    'no-empty': 1, //块语句中的内容不能为空
    'no-irregular-whitespace': 2, //不能有不规则的空格
    'no-multi-spaces': 1, //不能用多余的空格
    'no-redeclare': 2, //禁止重复声明变量
    indent: [{ SwitchCase: 1 }], // 使用缩进量为2       // 解决eslint对Switch case语句重复缩进警告
    semi: [2, 'never'], // 不使用分号
    quotes: [2, 'single'], // 使用单引号
    'vue/no-parsing-error': [2, { 'x-invalid-end-tag': false }],
    'vue/valid-template-root': 0, // vue3 不需要唯一root
  },
}
```

