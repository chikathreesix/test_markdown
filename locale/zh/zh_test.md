## Webpack 构建

默认情况下, [vue-loader](http://vue-loader.vuejs.org/en/)自动使用 `css-loader` 和Vue模板编译器来编译处理vue文件中的样式和模板。在此编译过程中，所有的资源URL例如 `<img src="...">`、 `background: url(...)` 和 CSS中的 `@import` 均会被解析成模块通过 `require` 引用。
