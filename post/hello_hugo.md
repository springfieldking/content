---
title: "Hello Hugo"
date: 2019-11-16T11:20:04+08:00
tags: [ "hello"]
categories: ["hello"]
---

> 这是第一次使用hugo, 感觉不错, 希望以后能在后台领域写出更多的文章

<!--more-->

# 目的
写博客的目的是为了自身的技术沉淀以及技术的分享，和大家一起成长

# 结构
- gitlab 主仓库, 负责整个hugo工程，内含几个submodule
- content 子模块, 负责管理所有md的文章
- public 子模块, hugo的发布目录，也是github的静态页面的仓库
- themes/even 子模块， hugo的主题目录, 目前引用了even主题

# 部署

## 预览
``` shell
hugo server --theme=even --buildDrafts
```

## 生成
``` shell
hugo --theme=even --baseUrl="https://springfieldking.github.io/" --buildDrafts
```

## 发布
- 先提交子模块
先将子模块content和public分别提交

- 主仓库
合适的时候提交一下


## 测试特性


测试mermaid流程图

{{< mermaid >}}
sequenceDiagram
    participant Alice
    participant Bob
    Alice->John: Hello John, how are you?
    loop Healthcheck
        John->John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail...
    John-->Alice: Great!
    John->Bob: How about you?
    Bob-->John: Jolly good!
{{< /mermaid >}}

测试嵌入youtube视频

{{< youtube UIp-qO22kQo >}}

测试嵌入B站视频

{{< bilibili BV1es411D7sW >}}


测试vchart图表
{{< safe-html >}}
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<script src="https://cdn.jsdelivr.net/npm/echarts/dist/echarts.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/v-charts/lib/index.min.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/v-charts/lib/style.min.css">


<div id="app" style=" width:640px;">
    <ve-line :data="chartData"></ve-line>
</div>

<script>
    new Vue({
      el: '#app',
      data: function () {
        return {
          chartData: {
            columns: ['日期', '销售额'],
            rows: [
              { '日期': '1月1日', '销售额': 123 },
              { '日期': '1月2日', '销售额': 1223 },
              { '日期': '1月3日', '销售额': 2123 },
              { '日期': '1月4日', '销售额': 4123 },
              { '日期': '1月5日', '销售额': 3123 },
              { '日期': '1月6日', '销售额': 7123 }
            ]
          }
        }
      }
    })
</script>
{{< /safe-html >}}
