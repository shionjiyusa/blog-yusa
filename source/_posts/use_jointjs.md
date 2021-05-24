---
title: "Jointjs 使用总结"
url: use_jointjs.html
id: use_jointjs
categories:
  - 开发日志
date: 2021-05-24 09：30：00
tags:
---

![](/img/post/atri.png)

<!-- TOC -->

- [1. Jointjs 使用总结](#1-jointjs-使用总结)
  - [1.1. 版本](#11-版本)
  - [1.2. 初始化相关](#12-初始化相关)
    - [1.2.1. 代码示例](#121-代码示例)
  - [1.3. 图形元素](#13-图形元素)
    - [1.3.1. 绑定元素点击事件](#131-绑定元素点击事件)
    - [1.3.2. 绑定连线的操作工具](#132-绑定连线的操作工具)
    - [1.3.3. shapes.standard 标准插件](#133-shapesstandard-标准插件)
    - [1.3.4. 连线](#134-连线)
  - [1.4. 通用方法](#14-通用方法)
    - [1.4.1. 画布导入和导出](#141-画布导入和导出)
    - [1.4.2. 常用方法](#142-常用方法)
  - [1.5. 整体代码示例](#15-整体代码示例)

<!-- /TOC -->

# 1. Jointjs 使用总结

Jointjs 是一个使用 svg 绘制流程图、拓扑关系图的 JS 库，比起 G6 引擎来说，jointjs 的接口使用更直观，最终呈现效果也比较理想，特别是 Link 类型连线的自带工具非常好用，完美契合开发需求。但是这个库比较冷门，中文资料不多，而且多数都是踩坑记录，事实上我实际用起来也是踩了无数的坑，心累

## 1.1. 版本

目前官方最新版本为 3.3.1，项目里的 webpack 版本比较老，直接使用 npm 安装 3.3.1 会出现打包失败的情况，推测是因为最新版本引用的某个库代码兼容性不够导致的，所以以下内容均基于 3.1.0 版本。

> 文档相关链接

官方文档，版本不能切换时直接修改链接地址的版本 <https://resources.jointjs.com/docs/jointjs/v3.3/joint.html>

旧版文档翻译参考 <https://segmentfault.com/a/1190000004579894>

事件类型参考 <https://blog.csdn.net/qq_39324050/article/details/109530753>

svg 文档 <https://developer.mozilla.org/en-US/docs/Web/SVG/Element/tspan>

## 1.2. 初始化相关

1. 使用 import 方式引入

   - `npm i jointjs -S`，自带 jQuery、backbone、lodash，如果使用 script 标签引入，需要手动引入这些依赖
   - `import * as joint from 'jointjs'` 整体引入（常用）
   - `import { dia } from 'jointjs/src/core.mjs'` 按需引入

   项目里只用了 import 方式引入，所以也导致踩了一系列坑，建议尝试用 script 全局引入，坑应该少一点（x

1. 初始化画布

> 坑注意：import 方式引入的时候，joint 全局变量是没有全局挂载的，所以必须手动传入 cellNamespace 参数，不然报错

ES6 引入时命名空间报错问题参考 <https://stackoverflow.com/questions/57428402/getting-error-dia-elementview-markup-required-from-graph-fromjsonjson>

```js
import * as joint from "jointjs";

// 初始化画布模型，使用 ES6 引入 jointjs 时，joint 全局变量未注册，必须传入 cellNamespace 参数
const graph = new joint.dia.Graph({}, { cellNamespace: joint.shapes });

// 初始化画布视图
const element = document.getElementById("paper");
const height = element.clientHeight;
const width = element.clientWidth;
const paper = new joint.dia.Paper({
  // 挂载的 dom 元素
  el: element,
  // 关联声明的 graph
  model: graph,
  // 画布大小，初始化之后还可以动态调整，项目里没试过，自行查资料
  width,
  height,
  defaultAnchor: { name: "perpendicular" },
  gridSize: 5, // 画布上元素拖动时步进的为5像素，默认1
  drawGrid: true, // 显示步进点，方便对齐，3.1.0 版本貌似不起作用？
  // snapLinks: {
  //   radius: 5, // 距离元素连接点 5 像素时自动连接上
  // },
  snapLinks: true, // 坑注意，3.1.0 版本设置像素值时会以元素中心点来计算范围，像上面设置为 5，范围就会特别小
  interactive: {
    // 禁用元素拖拽、连线标签拖拽
    elementMove: true,
    labelMove: true,
    linkMove: true,
    arrowheadMove: true,
  },
  cellViewNamespace: joint.shapes, // 注意必须同步传入 cellViewNamespace 参数
  // linkConnectionPoint: joint.util.shapePerimeterConnectionPoint, // 链接将尝试在连接形状的周长上找到最佳的连接点，而不仅仅是在边界框上（3.1.0 貌似不起作用）
});
```

1. 清除画布所有元素

   - `graph.clear()` 清除元素
   - `paper.remove()` 这两个没具体研究，但是 paper.remove 好像同时会清理绑定的监听事件，一般我会同时调用这两个方法来清空画布

1. 画布缩放平移

jointjs 底层是用 svg 实现的，本身没有自带缩放平移的方法，需要手动绑定

平移参考 <https://stackoverflow.com/questions/28431384/how-to-make-a-paper-draggable>

缩放参考找不到了

### 1.2.1. 代码示例

> 多个动作需要绑定相同事件时，用空格分割，栗子：`'blank:pointerup blank:mouseout'`

```js
/**
 * 绑定鼠标滚轮缩放事件
 */
bindMouseWheel() {
  // 注意两个事件的回调参数不一致
  this.paper.on('blank:mousewheel', (e, x, y, delta) => this.handleCellMouseWheel(e, x, y, delta));
  this.paper.on('element:mousewheel', (cellView, e, x, y, delta) => this.handleCellMouseWheel(e, x, y, delta));
}
/**
 * 复制代码，抄就行了不解释
 */
handleCellMouseWheel(e, x, y, delta) {
  e.preventDefault();
  const oldScale = this.paper.scale().sx;
  const newScale = oldScale + delta * 0.1;
  this.scaleToPoint(newScale, x, y);
}
/**
 * 复制代码
 */
scaleToPoint(nextScale, x, y) {
  if (nextScale >= MIN_SCALE && nextScale <= MAX_SCALE) {
    const currentScale = this.paper.scale().sx;
    const beta = currentScale / nextScale;
    const ax = x - x * beta;
    const ay = y - y * beta;
    const translate = this.paper.translate();
    const nextTx = translate.tx - ax * nextScale;
    const nextTy = translate.ty - ay * nextScale;
    this.paper.translate(nextTx, nextTy);
    const ctm = this.paper.matrix();
    ctm.a = nextScale;
    ctm.d = nextScale;
    this.paper.matrix(ctm);
  }
}

/**
 * 绑定画布平移事件，mousedown 记录当前画布位置坐标（并计算缩放比），mouseup 清除当前画布位置坐标
 * 画布本身未定义鼠标移动相关事件，所以利用 dom 监听 mousemove 事件，将鼠标移动参数转换为画布移动
 * 组件卸载时要调用 destroy 事件清除事件监听
 */
bindDragPaper() {
  this.paper.on('blank:pointerdown', (e, x, y) => {
    const scale = this.paper.scale();
    this.dragPosition = [x * scale.sx, y * scale.sy];
    this.element.removeEventListener('mousemove', this.dragFunc);
    this.element.addEventListener('mousemove', this.dragFunc.bind(this));
  });
  this.paper.on('blank:pointerup blank:mouseout', (e) => {
    this.dragPosition = [];
    this.element.removeEventListener('mousemove', this.dragFunc);
  });
}
/**
 * 转换画布平移坐标
 * @param {Object} e mousemove 事件回调对象
 */
dragFunc(e) {
  if (this.dragPosition.length) {
    this.paper.translate(e.offsetX - this.dragPosition[0], e.offsetY - this.dragPosition[1]);
  }
}
```

附录

1. 禁止标签拖动，参考 <https://stackoverflow.com/questions/45382941/in-jointjs-how-to-prevent-element-with-ports-from-being-moved-but-still-allow>

1. 自定义元素引用报错，参考 <https://github.com/clientIO/joint/issues/1281>

## 1.3. 图形元素

### 1.3.1. 绑定元素点击事件

```js
/**
 * 绑定元素点击事件
 * @param {function} callback 回调函数
 * @param {boolean} highlight 点击元素是否高亮
 */
bindClick(callback, highlight) {
  const bindFunc = (cellView, evt, x, y) => {
    const cells = this.graph.getCells();
    cells.forEach((cell) => {
      // 取消其他元素高亮
      const view = this.paper.findViewByModel(cell);
      if (typeof view.unhighlight === 'function') {
        view.unhighlight();
      }
    });
    // 高亮当前点击的元素
    if (cellView && highlight) {
      if (typeof cellView.highlight === 'function') {
        cellView.highlight();
      }
    }
    // 点击事件回调
    if (typeof callback === 'function') {
      callback(cellView);
    }
  };
  // 绑定元素点击事件，坑注意：绑定 pointerclick 会有无法触发的 bug，所以退而监听 pointerup
  this.paper.on('cell:pointerup link:pointerup', (cellView, evt, x, y) => bindFunc(cellView, evt, x, y));
  this.paper.on('blank:pointerup', (evt, x, y) => bindFunc(null, evt, x, y));
}
```

### 1.3.2. 绑定连线的操作工具

```js
/**
 * @description 绑定连线相关属性，参考官方手册
 */
bindLinkEvent() {
  // 绑定连线的操作
  const verticesTool = new joint.linkTools.Vertices();
  const segmentsTool = new joint.linkTools.Segments();
  const sourceArrowheadTool = new joint.linkTools.SourceArrowhead();
  const targetArrowheadTool = new joint.linkTools.TargetArrowhead();
  const sourceAnchorTool = new joint.linkTools.SourceAnchor();
  const targetAnchorTool = new joint.linkTools.TargetAnchor();
  const boundaryTool = new joint.linkTools.Boundary();
  const removeButton = new joint.linkTools.Remove();
  const toolsView = new joint.dia.ToolsView({
    tools: [
      verticesTool,
      segmentsTool,
      sourceArrowheadTool,
      targetArrowheadTool,
      sourceAnchorTool,
      targetAnchorTool,
      boundaryTool,
      removeButton,
    ],
  });
  // 鼠标移入时显示工具，移除时隐藏工具
  this.paper.on('link:mouseenter', (linkView) => {
    linkView.addTools(toolsView);
    linkView.showTools(toolsView);
  });
  this.paper.on('link:mouseleave', (linkView) => {
    // 坑注意：3.1.0 版本不能用 addTools 和 removeTools 搭配，不然工具会 bug，显示不全
    linkView.hideTools();
  });
}
```

### 1.3.3. shapes.standard 标准插件

具体各个插件的用法可以参考官方文档

```js
const standard = joint.shapes.standard;
const rectangle = new standard.Rectangle();
const circle = new standard.Circle();
const ellipse = new standard.Ellipse();
const path = new standard.Path();
const polygon = new standard.Polygon();
const polyline = new standard.Polyline();
const cylinder = new standard.Cylinder();
const image = new standard.Image();
const borderedImage = new standard.BorderedImage();
const embeddedImage = new standard.EmbeddedImage();
const headeredRectangle = new standard.HeaderedRectangle();
const textBlock = new standard.TextBlock();
const link = new standard.Link();
const doubleLink = new standard.DoubleLink();
const shadowLink = new standard.ShadowLink();
```

> 使用方式以 circle 为例

```js
/**
 * @param {number} x 坐标
 * @param {number} y
 * @return {string} 元素在画布上的唯一 ID
 */
addCircle(x = 10, y = 10) {
  const circle = new joint.shapes.standard.Circle();
  circle.resize(50, 50); // 元素大小
  circle.position(x, y); // 元素位置
  circle.attr('root/title', '测试元素'); // 设置 dom 的 title 属性
  circle.attr('label/text', 'hello\nworld'); // 文字，注意 svg 控制换行非常麻烦，用 \n 能够解决换行的问题
  circle.attr('label/fontSize', '25'); // 文字大小
  circle.attr('body/fill', '#f79204'); // 背景颜色
  circle.attr('body/stroke', '#f79204'); // 边框颜色
  circle.attr('label/fill', '#fff'); // 文字颜色
  this.graph.addCell(circle); // 单个添加
  // this.graph.addCells([circle]); // 也可以批量添加
  if (id) {
    // 回显画布时，也可以覆盖元素自身生成的 ID
    circle.prop(['id'], id);
  }
  return circle.id;
  // joint.util.breakText('this is quite a long text', { width: 50 }) 自动根据宽度换行，3.1.0 貌似不起作用
}
```

> 使用方式分析

1. 画布添加元素有两种方法，批量添加和单个添加
1. 元素的属性设置可以使用两种方法，一般推荐使用 `item.attr()` 方法操作相应属性，但是有些上层属性是不属于 attr 方法的，比如连线的相关属性，此时可以尝试使用 `item.prop([...], ...)` 操作属性，其中第一个数组参数代表属性的层级，与 `attr` 方法的斜杠划分意义相同

### 1.3.4. 连线

连线单独列出来，因为连线的操作比较复杂，jointjs 本身是支持元素之间直接新建连接，没研究透，不会用

注意连线的标签属性在上层对象里，需要使用 `item.prop()` 方法进行控制

```js
/**
 * 添加连接线
 * @param {number} x 初始坐标
 * @param {number} y 初始坐标
 */
addLink(x = 50, y = 50) {
  const link = new joint.shapes.standard.Link();
  link.source({ x, y }); // 起点，可以传元素字符串 ID
  // link.source(id);
  link.target({ x: x + 80, y }); // 终点
  link.labels([
    {
      attrs: {
        text: {
          text: '', // 标签文字
        },
      },
    },
  ]);
  link.attr('line/stroke', '#2f76ff'); // 线条颜色
  link.prop(['doubleLinkTools'], 'true'); // 显示双箭头
  link.prop(['labels', 0, 'attrs', 'rect', 'fill'], 'rgba(0, 0, 0, 0)'); // 标签背景矩形的颜色
  this.graph.addCell(link);
}
```

## 1.4. 通用方法

### 1.4.1. 画布导入和导出

jointjs 支持直接将画布导出为 json 对象，也可以将对象导入到画布上

```js
/**
 * 导出画布元素
 * @return {Object} JSON
 */
exportGraph() {
  const jsonData = this.graph.toJSON();
  return jsonData;
}
/**
 * 导入画布数据，JSON 对象，注意并非 JSON 字符串
 * @param {Object} jsonData JSON
 * @example {"cells": [{...}, {...}]}
 */
importGraph(jsonData) {
  this.graph.fromJSON(jsonData);
}
```

### 1.4.2. 常用方法

```js
/**
 * 移除单个元素
 * @param {Object} cellView 元素对象，视图对象或者模型对象均可，可以使用下述 getCellByID() 方法来获取
 */
removeItem(cellView) {
  if (cellView.model && typeof cellView.model.remove === 'function') {
    cellView.model.remove();
  } else if (typeof cellView.remove === 'function') {
    cellView.remove();
  }
}

/**
 * 获取当前画布所有元素，和导出画布好像差不多
 * @return {Array} 元素数组
 */
getCells() {
  return this.graph.getCells();
}

/**
 * 用元素 ID 获取元素视图对象，可以用于操作元素属性、删除元素
 * @param {string} id
 * @return {Object} 元素视图对象
 */
getCellByID(id) {
  return this.graph.getCell(id);
}

/**
 * 获取画布当前的缩放比例和平移位置
 * @return {Object} {scale, position}
 */
getPaperScaleAndPos() {
  const scale = this.paper.scale();
  const position = this.paper.translate();
  return { scale, position };
}

/**
 * 清除当前画布
 */
clearGraph() {
  this.graph.clear();
}

/**
 * 销毁当前画布，并且销毁画布所有监听事件
 */
destroy() {
  this.clearGraph();
  this.paper.remove();
  // 移除画布平移绑定的 dom 事件
  this.element.removeEventListener('mousemove', this.dragFunc);
}
```

## 1.5. 整体代码示例

项目需求比较简单，所以也没做彻底研究，写的类也只能基本适应简单需求，权当参考，自定义元素这部分实在没理解透，只能退而求其次使用单独添加渲染的方式来自定义元素，逻辑也就显得有点啰嗦了

```js
/* eslint-disable class-methods-use-this */
import * as joint from 'jointjs';
import _ from 'lodash';
import $ from 'jquery';

const MIN_SCALE = 0.1;
const MAX_SCALE = 10;

/**
 * @classdesc jointjs 封装
 * @param {string} id 绑定的 dom ID
 * @param {boolean} elementMove 是否禁用元素拖拽
 * @date 2021-05-8
 * @author zhw
 */
class JointJS {
  /**
   * @private
   * @desc 画布模型对象
   */
  graph = null;

  /**
   * @private
   * @desc 画布视图对象
   */
  paper = null;

  /**
   * @private
   * @desc 画布渲染绑定的 dom 对象
   */
  element = null;

  /**
   * @private
   * @desc 自定义元素组
   */
  customElement = {};

  /**
   * @private
   * @desc 拖拽画布时保存画布初始位置，用于计算拖拽距离
   */
  dragPosition = [];

  /**
   * @param {string} id 绑定的 dom ID
   * @param {boolean} elementMove 是否禁用元素拖拽
   */
  constructor(id, elementMove = true) {
    try {
      this.element = document.getElementById(id);
      if (!this.element) {
        throw new Error('找不到挂载点');
      }
      this.initGraph();
      this.initPaper(elementMove);
      this.initCustomElement();
      this.bindEvent();
    } catch (e) {
      console.log('jointjs error:', e.message);
    }
  }

  /**
   * @protected
   * @description 初始化画布模型，使用 ES6 引入 jointjs 时，joint 全局变量未注册，必须传入 cellNamespace 参数
   */
  initGraph() {
    this.graph = new joint.dia.Graph({}, { cellNamespace: joint.shapes });
  }

  /**
   * @protected
   * @description 初始化画布视图，必须传入 cellViewNamespace 参数
   * @param {boolean} movable 元素是否可拖拽
   */
  initPaper(movable) {
    const height = this.element.clientHeight;
    const width = this.element.clientWidth;
    this.paper = new joint.dia.Paper({
      // 挂载的 dom 元素
      el: this.element,
      // 关联声明的 graph
      model: this.graph,
      width,
      height,
      defaultAnchor: { name: 'perpendicular' },
      gridSize: 5, // 画布上元素拖动时步进的为5像素，默认1
      drawGrid: true, // 显示步进点，方便对齐
      // snapLinks: {
      //   radius: 5, // 距离元素连接点 5 像素时自动连接上
      // },
      snapLinks: true, // 坑注意，3.1 版本设置像素值时会以元素中心点来计算范围
      interactive: { elementMove: movable, labelMove: movable, linkMove: movable, arrowheadMove: movable }, // 禁用元素拖拽、连线标签拖拽
      cellViewNamespace: joint.shapes,
      // linkConnectionPoint: joint.util.shapePerimeterConnectionPoint, // 链接将尝试在连接形状的周长上找到最佳的连接点，而不仅仅是在边界框上（貌似不起作用）
    });
  }

  /**
   * @protected
   * @description 初始化自定义元素
   * 由于元素无法绑定在 joint.shapes 上，所以用 this.customElement 保存自定义元素设置，在导入 graph JSON 时，自定义元素只能手动添加
   */
  initCustomElement() {
    // 此处自定义了一个带有标题头部的矩形元素
    this.customElement.HeaderRect = joint.dia.Element.define('standard.Header', {
      // default attributes
      // markup 为元素模型参数，tagName 决定 svg 元素类型，selector 为属性选择器名称。整体类似于用 html 标签写的布局框架，后续 attrs 等参数设置相应 svg 标签的属性
      markup: [
        {
          tagName: 'rect',
          selector: 'header',
        },
        {
          tagName: 'rect',
          selector: 'body',
        },
        {
          tagName: 'text',
          selector: 'headerText',
        },
        {
          tagName: 'text',
          selector: 'bodyText',
          children: [
            {
              tagName: 'tspan',
              selector: 'bodyText1',
            },
            {
              tagName: 'tspan',
              selector: 'bodyText2',
            },
          ],
        },
      ],
      attrs: {
        header: {
          refWidth: '100%', // 相对宽度
          height: 30, // 绝对高度
          strokeWidth: 1, // 边框宽度
          stroke: '#000000', // 边框颜色
          fill: '#eee', // 背景颜色
          fontSize: '16', // 字体大小
        },
        body: {
          refWidth: '100%',
          refHeight: '70%', // 相对高度
          refY: 30, // 相对位置
          strokeWidth: 1,
          stroke: '#000000',
          fill: '#fff',
        },
        headerText: {
          textVerticalAnchor: 'middle', // 文字垂直对齐
          textAnchor: 'middle', // 文字水平对齐
          refX: '50%',
          refY: 15,
          fontSize: 16,
          fill: '#000', // 在 tspan, text 标签中，fill 为字体颜色，具体参考 svg 文档
        },
        bodyText: {
          // textAnchor: 'middle',
          refX: '5%',
          refY: 30,
          fontSize: 14,
          fill: '#000',
        },
        bodyText1: {
          fontSize: 14,
          fill: 'red',
        },
        bodyText2: {
          fontSize: 14,
          fill: '#000',
        },
      },
    });
  }

  /**
   * @protected
   * @description 决定需要绑定哪些画布事件，可重写
   * @param {function} callback 仅设置点击事件的回调函数
   */
  bindEvent(callback, highlight) {
    this.bindMouseWheel();
    this.bindDragPaper();
    this.bindClick(callback, highlight);
    this.bindLinkEvent();
  }

  /**
   * @private
   * @description 绑定鼠标滚轮缩放事件
   */
  bindMouseWheel() {
    this.paper.on('blank:mousewheel', (e, x, y, delta) => this.handleCellMouseWheel(e, x, y, delta));
    this.paper.on('element:mousewheel', (cellView, e, x, y, delta) => this.handleCellMouseWheel(e, x, y, delta));
  }

  /**
   * @private
   * 复制代码
   */
  handleCellMouseWheel(e, x, y, delta) {
    e.preventDefault();
    const oldScale = this.paper.scale().sx;
    const newScale = oldScale + delta * 0.1;
    this.scaleToPoint(newScale, x, y);
  }

  /**
   * @private
   * 复制代码
   */
  scaleToPoint(nextScale, x, y) {
    if (nextScale >= MIN_SCALE && nextScale <= MAX_SCALE) {
      const currentScale = this.paper.scale().sx;
      const beta = currentScale / nextScale;
      const ax = x - x * beta;
      const ay = y - y * beta;
      const translate = this.paper.translate();
      const nextTx = translate.tx - ax * nextScale;
      const nextTy = translate.ty - ay * nextScale;
      this.paper.translate(nextTx, nextTy);
      const ctm = this.paper.matrix();
      ctm.a = nextScale;
      ctm.d = nextScale;
      this.paper.matrix(ctm);
    }
  }

  /**
   * @private
   * @description 绑定画布平移事件，mousedown 记录当前画布位置坐标（并计算缩放比），mouseup 清除当前画布位置坐标
   * 画布本身未定义鼠标移动相关事件，所以利用 dom 监听 mousemove 事件，将鼠标移动参数转换为画布移动
   * 组件卸载时要调用 destroy 事件清除事件监听
   */
  bindDragPaper() {
    this.paper.on('blank:pointerdown', (e, x, y) => {
      const scale = this.paper.scale();
      this.dragPosition = [x * scale.sx, y * scale.sy];
      this.element.removeEventListener('mousemove', this.dragFunc);
      this.element.addEventListener('mousemove', this.dragFunc.bind(this));
    });
    this.paper.on('blank:pointerup blank:mouseout', (e) => {
      this.dragPosition = [];
      this.element.removeEventListener('mousemove', this.dragFunc);
    });
  }

  /**
   * @private
   * @description 转换画布平移坐标
   * @param {Object} e mousemove 事件回调对象
   */
  dragFunc(e) {
    if (this.dragPosition.length) {
      this.paper.translate(e.offsetX - this.dragPosition[0], e.offsetY - this.dragPosition[1]);
    }
  }

  /**
   * @private
   * @description 绑定元素点击事件
   * @param {function} callback 回调函数
   */
  bindClick(callback, highlight) {
    const bindFunc = (cellView, evt, x, y) => {
      const cells = this.graph.getCells();
      cells.forEach((cell) => {
        // 取消其他元素高亮
        const view = this.paper.findViewByModel(cell);
        if (typeof view.unhighlight === 'function') {
          view.unhighlight();
        }
      });
      // 高亮当前点击的元素
      if (cellView && highlight) {
        if (typeof cellView.highlight === 'function') {
          cellView.highlight();
        }
      }
      // 点击事件回调
      if (typeof callback === 'function') {
        callback(cellView);
      }
    };
    // 绑定元素点击事件，坑注意：绑定 pointerclick 会有无法触发的 bug，退而监听 pointerup
    this.paper.on('cell:pointerup link:pointerup', (cellView, evt, x, y) => bindFunc(cellView, evt, x, y));
    this.paper.on('blank:pointerup', (evt, x, y) => bindFunc(null, evt, x, y));
  }

  /**
   * @private
   * @description 绑定连线相关属性，参考官方手册
   */
  bindLinkEvent() {
    // 绑定连线的操作
    const verticesTool = new joint.linkTools.Vertices();
    const segmentsTool = new joint.linkTools.Segments();
    const sourceArrowheadTool = new joint.linkTools.SourceArrowhead();
    const targetArrowheadTool = new joint.linkTools.TargetArrowhead();
    const sourceAnchorTool = new joint.linkTools.SourceAnchor();
    const targetAnchorTool = new joint.linkTools.TargetAnchor();
    const boundaryTool = new joint.linkTools.Boundary();
    const removeButton = new joint.linkTools.Remove();

    const toolsView = new joint.dia.ToolsView({
      tools: [
        verticesTool,
        segmentsTool,
        sourceArrowheadTool,
        targetArrowheadTool,
        sourceAnchorTool,
        targetAnchorTool,
        boundaryTool,
        removeButton,
      ],
    });

    this.paper.on('link:mouseenter', (linkView) => {
      linkView.addTools(toolsView);
      linkView.showTools(toolsView);
    });
    this.paper.on('link:mouseleave', (linkView) => {
      // 坑注意：3.1 版本不能用 addTools 和 removeTools 搭配，不然工具会 bug，显示不全
      linkView.hideTools();
    });
  }

  /**
   * @public
   * @description 添加圆形
   * @return {string} 返回圆形元素的 ID
   */
  addCircle(x = 10, y = 10) {
    const circle = new joint.shapes.standard.Circle();
    circle.resize(50, 50);
    circle.position(x, y);
    circle.attr('root/title', '测站号'); // dom 的 title 属性
    circle.attr('label/text', ''); // 文字
    circle.attr('label/fontSize', '25'); // 文字大小
    circle.attr('body/fill', '#f79204'); // 背景颜色
    circle.attr('body/stroke', '#f79204'); // 边框颜色
    circle.attr('label/fill', '#fff');
    this.graph.addCell(circle);

    return circle.id;
    // joint.util.breakText('this is quite a long text', { width: 50 }) 自动根据宽度换行
  }

  // 自定义元素失败，仅当参考
  addParamRect() {
    const that = this;
    joint.shapes.html = {};
    joint.shapes.html.Element = joint.shapes.basic.Rect.extend({
      defaults: joint.util.deepSupplement(
        {
          type: 'html.Element',
          attrs: {
            rect: { stroke: 'none', 'fill-opacity': 0 },
          },
        },
        joint.shapes.basic.Rect.prototype.defaults
      ),
    });

    joint.shapes.html.ElementView = joint.dia.ElementView.extend({
      template: [
        '<div class="html-element-rect">',
        '<div class="html-element-rect-header"></div>',
        '<div class="html-element-rect-body1"></div>',
        '<div class="html-element-rect-body2"></div>',
        '<div class="html-element-rect-body3"></div>',
        '</div>',
      ].join(''),
      initialize() {
        _.bindAll(this, 'updateBox');
        joint.dia.ElementView.prototype.initialize.apply(this, arguments);

        this.$box = $(_.template(this.template)());
        // // This is an example of reacting on the input change and storing the input data in the cell model.
        // this.$box.find('input').on(
        //   'change',
        //   _.bind(function (evt) {
        //     this.model.set('input', $(evt.target).val());
        //   }, this)
        // );
        // this.$box.find('select').on(
        //   'change',
        //   _.bind(function (evt) {
        //     this.model.set('select', $(evt.target).val());
        //   }, this)
        // );
        // this.$box.find('select').val(this.model.get('select'));
        // this.$box.find('.delete').on('click', _.bind(this.model.remove, this.model));
        // // Update the box position whenever the underlying model changes.
        this.model.on('change', this.updateBox, this);
        // Remove the box when the model gets removed from the graph.
        this.model.on('remove', this.removeBox, this);

        this.updateBox();
      },
      render() {
        joint.dia.ElementView.prototype.render.apply(this, arguments);
        this.paper.$el.prepend(this.$box);
        this.listenTo(this.paper, 'scale', this.updateBox);
        this.listenTo(this.paper, 'translate', this.updateBox);
        this.updateBox();
        return this;
      },
      updateBox() {
        // Set the position and dimension of the box so that it covers the JointJS element.
        const bbox = this.model.getBBox();
        const scale = that.paper.scale();
        const trans = that.paper.translate();
        // Example of updating the HTML with a data stored in the cell model.
        this.$box.find('html-element-rect-header').text(this.model.get('header'));
        this.$box.find('html-element-rect-body1').text(this.model.get('body1'));
        this.$box.find('html-element-rect-body2').text(this.model.get('body2'));
        this.$box.find('html-element-rect-body3').text(this.model.get('body3'));
        this.$box.css({
          // transform: `scale(${scale.sx},${scale.sy})`,
          transformOrigin: '0 0',
          width: bbox.width * scale.sx,
          height: bbox.height * scale.sy,
          left: bbox.x * scale.sx + trans.tx,
          top: bbox.y * scale.sy + trans.ty,
        });
      },
      removeBox(evt) {
        this.$box.remove();
      },
    });

    const el1 = new joint.shapes.html.Element({
      position: { x: 80, y: 80 },
      size: { width: 170, height: 100 },
      header: 'I am HTML',
      body1: '1111',
      body2: '2222',
      body3: '3333',
    });
    this.graph.addCell(el1);
    return el1.id;
  }

  /**
   * @public
   * @description 添加自定义的标题框，参数均为可选
   * @param {number} x 坐标
   * @param {number} y 坐标
   * @param {number} width 宽度
   * @param {number} height 高度
   * @param {string} headerText 标题文字
   * @param {string} bodyText1 红色预警文字
   * @param {string} bodyText2 普通文字
   * @param {sting} id 覆盖自身生成的 ID
   * @return {string} 返回元素ID
   */
  addHeaderRect(x = 100, y = 10, width = 150, height = 100, headerText = '', bodyText1 = '', bodyText2 = '', id = '') {
    const headeredRectangle = new this.customElement.HeaderRect();
    headeredRectangle.resize(width, height);
    headeredRectangle.position(x, y);
    headeredRectangle.attr('headerText/text', headerText);
    headeredRectangle.attr('bodyText1/text', bodyText1);
    headeredRectangle.attr('bodyText2/text', `\n${bodyText2}`); // svg 控制换行非常麻烦，用 \n 能够解决换行的问题
    if (id) {
      // 回显画布时，覆盖自身生成的 ID
      headeredRectangle.prop(['id'], id);
    }
    this.graph.addCell(headeredRectangle);
    return headeredRectangle.id;
  }

  /**
   * @public
   * @description 添加矩形
   * @param {number} x 初始坐标
   * @param {number} y 初始坐标
   * @return {string} 返回矩形 ID
   */
  addRect(x = 50, y = 10) {
    const rect = new joint.shapes.standard.Rectangle();
    rect.resize(90, 40);
    rect.position(x, y);
    rect.attr('root/title', 'joint.shapes.standard.Rectangle');
    rect.attr('label/text', '');
    rect.attr('label/fontSize', '18');
    rect.attr('body/stroke', '#6cbd16');
    rect.attr('body/fill', '#6cbd16');
    rect.attr('label/fill', '#fff');

    this.graph.addCell(rect);
    return rect.id;
  }

  /**
   * @public
   * @description 添加连接线
   * @param {number} x 初始坐标
   * @param {number} y 初始坐标
   */
  addLink(x = 50, y = 50) {
    const link = new joint.shapes.standard.Link();
    link.source({ x, y }); // 起点，可以传元素字符串 ID
    link.target({ x: x + 80, y }); // 终点

    link.labels([
      {
        attrs: {
          text: {
            text: '', // 标签文字
          },
        },
      },
    ]);
    link.attr('line/stroke', '#2f76ff'); // 线条颜色
    link.prop(['doubleLinkTools'], 'true'); // 显示双箭头
    link.prop(['labels', 0, 'attrs', 'rect', 'fill'], 'rgba(0, 0, 0, 0)'); // 标签背景矩形颜色
    this.graph.addCell(link);
  }

  /**
   * @public
   * @description 移除单个元素
   * @param {Object} cellView 元素对象，视图对象或者模型对象均可
   */
  removeItem(cellView) {
    if (cellView.model && typeof cellView.model.remove === 'function') {
      cellView.model.remove();
    } else if (typeof cellView.remove === 'function') {
      cellView.remove();
    }
  }

  /**
   * @public
   * @description 获取当前画布所有元素
   * @return {Array} 元素数组
   */
  getCells() {
    return this.graph.getCells();
  }

  /**
   * @public
   * @description 用元素 ID 获取元素视图对象
   * @param {string} id
   * @return {Object} 元素视图对象
   */
  getCellByID(id) {
    return this.graph.getCell(id);
  }

  /**
   * @public
   * @description 获取画布当前的缩放比例和平移位置
   * @return {Object} {scale, position}
   */
  getPaperScaleAndPos() {
    const scale = this.paper.scale();
    const position = this.paper.translate();
    return { scale, position };
  }

  /**
   * @public
   * @description 清除当前画布
   */
  clearGraph() {
    this.graph.clear();
  }

  /**
   * @public
   * @description 销毁当前画布，并且销毁画布所有监听事件
   */
  destroy() {
    this.clearGraph();
    this.paper.remove();
    this.element.removeEventListener('mousemove', this.dragFunc);
  }

  /**
   * @public
   * @description 导出画布元素
   * @return {Object} JSON
   */
  exportGraph() {
    const jsonData = this.graph.toJSON();
    return jsonData;
  }

  /**
   * @public
   * @description 导入画布数据，JSON 对象，非 JSON 字符串
   * @param {Object} jsonData JSON
   * @example {"cells": [{...}, {...}]}
   */
  importGraph(jsonData) {
    this.graph.fromJSON(jsonData);
  }
}

export default JointJS;
```
