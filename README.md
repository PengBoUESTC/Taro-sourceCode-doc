# Taro-sourceCode-doc
Taro framework source code learn


**本文将以`build` 为主线对`taro`打包流程&方案进行分析**

### `taro-cli`主流程

1. 命令&参数解析

2. 实例化`Kernel`，将```taro```内部的预设导入到 `Kernel` 实例中，这些`presets` 主要包括一些待注册的`hook`(`modifyWebpackChain, modifyMiniConfigs`)，以及生成打包文件的`hook`(`generateFrameworkInfo, generateProjectConfig`)
```javascript
const kernel = new Kernel({
  appPath,
  presets: [
    path.resolve(__dirname, '.', 'presets', 'index.js')
  ],
  plugins: []
})
```

3. 将`CLI`支持的命令（`build, init, info... `）**导入**到`Kernel`实例中
```javascript
kernel.optsPlugins.push(path.resolve(commandsPath, targetPlugin))
```

4. 根据命令行参数确定当前打包的`platform`(`weapp, swan, tt...`)，并将`platform`插件导入到`Kernel`实例中
```javascript
kernel.optsPlugins.push(`@tarojs/plugin-platform-${platform}`)
```

5. 根据命配置文件中的`framework`配置确定当前项目的前端框架（`vue, vue3, react...`），并将`framework`插件导入到 `Kernel` 实例中。
```javascript
kernel.optsPlugins.push('@tarojs/plugin-framework-react')
```

6. 通过 `customCommand` 执行 `build` 命令
```javascript
kernel.run({
  name: command,
  opts: {
    _: args._,
    options,
    isHelp: args.h
  }
})
```

### `Kernel`统一处理流程

`Kernel` 在 `@tarojs/service` 包中，是`taro`的核心，主要作用是存储&注册插件、hook、命令，接下来还是跟着`build`命令来研究其主要作用。`Kernel` 中的 `run` 方法主要流程如下

1. 注册`taro-cli`存储到`kernel` 中的`presets、plugin`
```javascript
this.initPresetsAndPlugins()
```

2. `onReady` hook 调用
```javascript
await this.applyPlugins('onReady')
```

3. `osStart` hook 调用
```javascript
await this.applyPlugins('osStart')
```

4. `modifyRunnerOpts` hook 调用，用于修改项目配置

5. `build` 命令行命令调用
```javascript
await this.applyPlugins({
  name,
  opts
})
```

其中 `applyPlugins` 方法用于调用不同的命令、hook
```javascript
// ...
const hooks = this.hooks.get(name) || []
// ...
const res = await hook.fn(opts, arg)
// ...
```

### `build` 命令

`build` 命令用于将项目构建为目标平台的小程序代码，主要流程如下

1. 配置信息校验
```javascript
const checkResult = await checkConfig({
  configPath,
  projectConfig: ctx.initialConfig
})
```

2. 通过 `applyPlugins`调用打包过程中需要的操作

    1. `onBuildStart` hook 调用
    ```javascript
    await ctx.applyPlugins(hooks.ON_BUILD_START)
    ```

    2. `platform` 方法调用
    ```javascript
    ctx.applyPlugins({
        name: platform,
        opts: {
          config: {
            // ...
            // 声明 build 过程中需要的 hook
            async modifyWebpackChain (chain, webpack, data) {
              await ctx.applyPlugins({
                name: hooks.MODIFY_WEBPACK_CHAIN,
                initialVal: chain,
                opts: {/** */}
              })
            },
            async modifyBuildAssets (assets, miniPlugin) {
              await ctx.applyPlugins({
                name: hooks.MODIFY_BUILD_ASSETS,
                initialVal: assets,
                opts: {/** */}
              })
            },
            async modifyMiniConfigs (configMap) {
              await ctx.applyPlugins({
                name: hooks.MODIFY_MINI_CONFIGS,
                initialVal: configMap,
                opts: {/** */}
              })
            },
            async modifyComponentConfig (componentConfig, config) {
              await ctx.applyPlugins({
                name: hooks.MODIFY_COMPONENT_CONFIG,
                opts: {/** */}
              })
            },
            async onCompilerMake (compilation, compiler, plugin) {
              await ctx.applyPlugins({
                name: hooks.ON_COMPILER_MAKE,
                opts: {/** */}
              })
            },
            async onParseCreateElement (nodeName, componentConfig) {
              await ctx.applyPlugins({
                name: hooks.ON_PARSE_CREATE_ELEMENT,
                opts: {/** */}
              })
            },
            async onBuildFinish ({ error, stats, isWatch }) {
              await ctx.applyPlugins({
                name: hooks.ON_BUILD_FINISH,
                opts: {/** */}
              })
            }

          }
        }
    })
    ```

  3. 调用 `onBuildComplete` hook
  ```javascript
  await ctx.applyPlugins(hooks.ON_BUILD_COMPLETE)
  ```

### `platform`

经过对 `build` 命令的分析，打包过程中最关键的就是 `platform` hook的调用，`platform` hook 是在 `taro-cli` 中存储到`kernel`中，然后在 `kernel.initPresetsAndPlugins` 中完成注册的，这里以 `weapp`(微信小程序)为例进行分析。`weapp` hook对应的包为`@tarojs/@tarojs/plugin-platform-weapp`。

```javascript
async fn ({ config }) {
  const program = new Weapp(ctx, config, options || {})
  await program.start()
}
```

 `Weapp` 继承自 `TaroPlatformBase` (@tarojs/service)，其内部的`start`方法如下
```javascript
public async start () {
  // 清空打包目录，生成 project.config.json， 打印提示信息
  await this.setup()
  // 获取 打包工具，调用打包工具开始打包
  await this.build()
}
```
其中比较关键的方法为 `buildImpl`, 该方法用于根据配置信息获取打包工具，并调用打包方法
```javascript
 private async buildImpl (extraOptions = {}) {
  // runner 可以是 '@tarojs/webpack5-runner 或者 @tarojs/mini-runner（webpack4）
    const runner = await this.getRunner()
    const options = this.getOptions(Object.assign({
      runtimePath: this.runtimePath,
      taroComponentsPath: this.taroComponentsPath
    }, extraOptions))
    await runner(options)
  }
```

## `@tarojs/webpack5-runner`

这个包的作用就是封装 `webpack5`实现最终的代码转换&打包

## 关键流程图
<img src="./taro-drawio.svg"/>
