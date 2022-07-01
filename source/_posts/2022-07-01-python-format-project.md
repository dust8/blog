---
title: python项目统一格式化方案
date: 2022-07-01 06:15:31
tags:
  - python
---
在代码合作编写中, 可能编辑器不同, 个人写代码习惯不同, 会出现代码不美观, 严重的会导致合并代码困难. 所以需要把代码统一格式化.

## editorconfig
它可以解决不同编辑器默认的格式不统一问题. 编辑器会读取该文件来设置默认文件格式.
这里可以参考 `django` 项目里面的设置
```
# .editorconfig
# https://editorconfig.org/

root = true

[*]
indent_style = space
indent_size = 4
insert_final_newline = true
trim_trailing_whitespace = true
end_of_line = lf
charset = utf-8

# Docstrings and comments use max_line_length = 79
[*.py]
max_line_length = 88

# Use 2 spaces for the HTML files
[*.html]
indent_size = 2

# The JSON files contain newlines inconsistently
[*.json]
indent_size = 2
insert_final_newline = ignore

[**/admin/js/vendor/**]
indent_style = ignore
indent_size = ignore

# Minified JavaScript files shouldn't be changed
[**.min.js]
indent_style = ignore
insert_final_newline = ignore

# Makefiles always use tabs for indentation
[Makefile]
indent_style = tab

# Batch files use tabs for indentation
[*.bat]
indent_style = tab

[docs/**.txt]
max_line_length = 79

[*.yml]
indent_size = 2
```

## black 解决代码格式化问题
`django` 的代码现在也改用 `black` 格式化工具了.可以在 `django` 源码里面看到.
```
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 22.3.0
    hooks:
    - id: black
      exclude: \.py-tpl$
  - repo: https://github.com/PyCQA/isort
    rev: 5.10.1
    hooks:
      - id: isort
      
# setup.cfg
[isort]
profile = black
default_section = THIRDPARTY
known_first_party = django
```
### vscode 编辑器使用 black
在`vscode`中先打开`python`文件, 在右键单击就会出现`格式化文档`选项, 在点击`格式化文档`, 如果还没有安装格式化工具的, 就会出现选择哪个格式化工具, 这里我们选择 `black`. 安装好后在同样操作一遍, 就会自动格式化了.

### 命令行使用 black
因为可能文件太多,不可能一一用编辑打开来格式化.有一些命令行,非常有用. 
```
# 检查格式化前后的不同
black . --check --diff

# 执行格式化
black .
```

## isort 解决包引入格式化问题
但是默认的配置, `isort` 和 `black` 格式化 `import` 代码不一致, 导致有问题,所以需要配置 `isort` 来适应 `black`. 在前面的 `.editorconfig` 文件里面配置 `profile = black` 来统一格式.
```
# .editorconfig
[*.py]
max_line_length = 88
profile = black
```

### vscode 编辑器使用 black
在`vscode`中先打开`python`文件, 在右键单击就会出现`Sort Imports`选项, 在点击`Sort Imports`,就自动格式化了.


### 命令行使用 isort
`isort` 和`black`一样有一些命令行,非常有用. 
```
# isort 简单的介绍和使用
isort

# isort 详细的使用介绍
isort --help

# 检查当前目录,并列出有格式化前后不一致的地方
isort --check --diff .

# 执行格式项目
isort .
```

## 代码提交前检查
在代码提交前先检查一下 `black` 和 `isort` 是否全都统一格式化了. 如果有输出需要格式化的文件, 格式化后在提交. 
```
black . --check
isort . --check
```
除了手动执行检查外, 还可以使用 `pre-commit` 工具来在提交前自动检查, 避免忘记检查. 如果检查不通过, 是提交不了的.

## 参考链接
- [django/django](https://github.com/django/django)
- [psf/black](https://github.com/psf/black)
- [isort](https://pycqa.github.io/isort/index.html)
- [Python代码import语句整理格式化工具-isort简介及使用详解](https://juejin.cn/post/7026151419459665957#heading-56)