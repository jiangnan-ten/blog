# 服务产品线 高级版

## 项目结构

> 只罗列出关键部分
```html
|- public
    |- plugins  -----------------------------------------------  静态资源文件
|- src/
    |- api  ---------------------------------------------------  api配置
    |- assets
        |- theme  ---------------------------------------------  主题文件
    |- components
        |- card
        |- echarts
        |- header
    |- config
        |- card.js  -------------------------------------------  卡片配置
        |- control.js  ----------------------------------------  图层,覆盖物控制面板配置
        |- map.js  --------------------------------------------  地图初始配置
        |- scene.js  ------------------------------------------  场景配置, 映射场景文件夹
        |- slider.js  -----------------------------------------  侧滑配置
        |- sys.js  --------------------------------------------  系统配置
    |- libs
    |- mapServices  -------------------------------------------  地图类服务封装
        |- index.js  ------------------------------------------  统一入口
        |- carService.js  -------------------------------------  走航车服务
        |- flexpartService.js  --------------------------------  flexpart服务
        |- gridService.js  ------------------------------------  栅格服务
        |- interpolationService.js  ---------------------------  插值服务
        |- mapSatelliteService.js  ----------------------------  卫星图层, 普通图层服务
        |- markerService.js  ----------------------------------  站点覆盖物服务
        |- radarService.js  -----------------------------------  雷达服务
        |- windyService.js  -----------------------------------  风场服务
    |- plugins  -----------------------------------------------  插件
        |- request --------------------------------------------  客户端http请求
    |- views
        |- home  ----------------------------------------------  首页(地图)
            |- card  ------------------------------------------  卡片
            |- slider  ----------------------------------------  侧滑
            |- components
            |- scene
                |- visitor  -----------------------------------  游客场景
                |- basic  -------------------------------------  基础场景
                |- pm25  --------------------------------------  pm25快速达标场景
                |- mixins
                    |- advanceOverlay.js  ---------------------  走航车, 雷达, 栅格mixin
                    |- interpolate.js  ------------------------  插值相关mixin
                    |- mixins.js  -----------------------------  通用mixin
                    |- siteMarker.js  -------------------------  站点marker mixin
    |- directive.js
|- .env.development
|- .env.beta
|- .env.production
|- .eslintignore
|- .eslintrc.js
|- deploy.conf.js
|- index.tpl
|- vue.config.js

```

## 静态文件抽离

为了尽可能的减少项目打包出来的体积, 抽离出项目中体积相对较大的libs, 并配置webpack, 不影响import导入, 同时直接跳过webpack打包构建过程, 抽离出echarts, moment, vue, antd, lodash

> vue.conf.js
```js
const configureWebpack = {
	externals: {
		moment: 'moment',
		echarts: 'echarts',
		lodash: '_'
	},
	plugins: []
}

if (isProd) {
	configureWebpack.plugins = [...configureWebpack.plugins]
	configureWebpack.externals.vue = 'Vue'
	configureWebpack.externals['ant-design-vue'] = 'antd'

	if (process.env.npm_config_report) {
		configureWebpack.plugins = [...configureWebpack.plugins, new BundleAnalyzerPlugin()]
	}
}
```

> index.tpl
```html
<script src="<%= BASE_URL %>plugins/echarts.v4.7.min.js"></script>
<script src="<%= BASE_URL %>plugins/echarts-liquidfill.v2.0.5.min.js"></script>
<script src="<%= BASE_URL %>plugins/moment.v2.24.0.min.js"></script>
<script src="<%= BASE_URL %>plugins/moment-with-locales.min.js"></script>
<script src="<%= BASE_URL %>plugins/lodash.v4.17.15.min.js"></script>
<script src="<%= BASE_URL %>plugins/stomp.js"></script>
<script src="<%= BASE_URL %>plugins/html2canvas.min.js"></script>


<% if (process.env.NODE_ENV === 'production') { %>
  <script src="<%= BASE_URL %>plugins/vue.runtime.v2.6.10.min.js"></script>
  <script src="<%= BASE_URL %>plugins/antd.v1.5.3.min.js"></script>
<% } %>
```


## webpack配置

> vue.config.js, 生产环境, 去除项目中的console, 图片小于10k做base64
```js
config.optimization.minimizer('terser').tap(args => {
  const { terserOptions } = args[0]
  terserOptions.compress.drop_console = true
  return args
})
config.plugins.delete('prefetch')
config.plugins.delete('preload')
config.module
  .rule('images')
  .use('url-loader')
  .loader('url-loader')
  .tap(options => Object.assign(options, { limit: 10240 }))
```

> 加入webpack-bundle-analyzer做打包分析
```js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
if (isProd) {
	if (process.env.npm_config_report) {
		configureWebpack.plugins = [...configureWebpack.plugins, new BundleAnalyzerPlugin()]
	}
}

// npm run build-prod --report 使用
```

## 代码规范

- eslint
- prettier
- gitHooks 每次代码commit都会检查规范

对于在项目中使用到的一些全局变量, 默认eslint会报错, 需配置.eslintrc.js
```js
globals: {
  AMap: true,
  Stomp: true,
  AMapLoader: true,
  html2canvas: true
}
```

对于引入的一些第三方库文件, 可能需要忽略eslint检查, 则配置.eslintignore, 例如
```js
/dist/
/public/
/src/libs/windy.js
/src/libs/grid.js
/src/libs/mapshot/
```

## 动态组件,导入
在项目中可以看到很多widget, 例如 /home/scene/widget.vue, /home/card/widget.vue, /home/slider/widget.vue, 像这些卡片, 场景, 侧滑都是根据权限, 配置接口获取然后渲染, 展示什么内容完全取决于配置, 所以需要动态组件帮我们处理, 当然也可以不这么做, 除非你不嫌弃页面上密密麻麻的if else if else......

vue中通过 `component` 实现动态组件, 但是组件仍然需要事先import进来, 那就意味着我们的页面充斥着大量的import xxx from ...., 非常不美观, 所以通过上层再封装一层wiget, 可以完美解决这个问题, 它会自动导入指定的文件夹里面的文件

```
<div style="padding: 0 16px 0 8px">
	<widget v-for="card in cardsData" :key="card.name" :name="card.name" />
</div>
```

```
<template>
	<component :is="componentFile" v-bind="$attrs" v-on="$listeners"> </component>
</template>

<script>
export default {
	props: {
		name: {
			type: String,
			required: true
		}
	},
	computed: {
		componentFile() {
			const component = this.name
			if (!component) {
				return
			}
			return () => import(/* webpackMode: "eager" */ /* webpackExclude: /\.index\.vue/ */ `./${component}`)
		}
	}
}
</script>
```

## 封装
项目中, 有多种场景, 例如 游客场景, 基础场景, pm25快速达标场景等等, 每个场景既有相同的, 也有不同的功能模块, 所以我们需要将这些功能模块抽离,封装
- ui 层面封装成 `组件`
	- 图层, 覆盖物控制面板
	- 时间轴
	- 卡片
	- 侧滑
	- 报警
	- 场景切换
	- ...
- 数据, 逻辑, 业务层面封装成 `mixin`
	- 站点, 覆盖物mixin
	- 走行车, 雷达, fp, 栅格mixin
	- 插值mixin
	- 基础通用mixin
	- ...
- 地图功能模块封装成 `mapService`
	- 走航车service
	- 雷达service
	- 站点覆盖物service
	- 栅格service
	- 风场service
	- ...

## 响应式?
众所周知, 挂载在vue data上的数据, 会被vue做defineproperty, 也就是做get,set操作, 如果是简单的值还好, 但如果是很复杂的object呢, vue会递归调用defineproperty处理对象的子子孙孙属性值, 这可是一个不小的性能开销, 在项目中, 我们经常会使用到高德的地图实例mapIns, 这个对象里面包含的大大小小, 子子孙孙对象非常庞大, 所以是绝对不能绑定在data对象中的, 那作何处理?

- Object.freeze(mapIns)
- 将mapIns 用单例模式再做一层抽象, 通过mix传入

> 传入
```js
import proxyMapSingleton from '@/libs/map'
const mapIns = proxyMapSingleton()

export default {
	mixins: [sceneMixin(mapIns)]
}
```

> 使用
```js
let mapIns = null

export default map => {
	mapIns = map
}
```

## 控制面板交互逻辑

<img src="_media/p1/process.png" align="right"/>


## 互斥逻辑
示例: 在地图上, 功能A和功能B, 功能C, 功能D....由于业务需求, 不能同时存在, 打开A, 则需要关闭其他:

* 雷达, 走航车互斥
* flexpart和插值互斥
* 插值(温度, 湿度, 气压)互斥
* 其他


> 为此我设计了一个数据结构, data 里面的元素互斥, type: 1 自我互斥 type: 2 自我不互斥
```js
[
	{
		groups: ['flexpart', 'temperature', 'humidity'],
		data: {
			flexpart: {type: 1, data: []},
			temperature: {type: 2, data: []},
			humidity: {type:2, data: []}
		}
	},
	{
		groups: ['radar', 'aerialvehicles'],
		data: {
			radar: {type: 1, data: []},
			aerialvehicles: {type: 1, data: []}
		}
	}
]
```
这样, 比如
* 现在flexpart自我互斥, 只需要将flexpart里面的data非当前id的值取反操作
* 当前值为humidity湿度, 那么就可以获取和他互斥的flexpart和temperature, 取他们的data中所有的数据即可

## vuex足够了么?
一般来说, vuex处理子子孙孙, 兄兄弟弟的事件通讯机制足够了, 但是对于强交互的操作逻辑, 还是显得比较捉襟见肘, 比如
* 首页的卡片和侧滑, 点击什么类型的卡片, 右侧展开对应的侧滑, 同时之前打开过的侧滑需要关闭
* 控制面板的开关按钮, 开关状态对应地图上的渲染
* 第三方库中也需要一种发布订阅机制, 在这里vuex力不从心
* ...

这里我们就需要自己实现一个发布订阅机制, 我们称之为 [better-event-manager](tools/bem.md), 并且在项目中大量使用
```js
BemInsForControl.fire('toggleControl', 'card', {
	code: 'grid',
	status: {
		show: false,
		icon: false
	}
})


BemInsForControl.on('toggleControl', (type, data) => {
	this.$store.commit('updateControl', { type, data })
	const { code, status } = data
	commandIns.exec(code, { ...data, mapIns })
})
```