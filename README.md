# DllPlugin & DllReferencePlugin

## 关于 DllPlugin

[open link](https://www.jianshu.com/p/d6beed284f60)

## context 所引发的问题

* vendor invalid
cyber-scripts 4.0.5 & 4.0.7

![analysis plugin](/images/main-dev-vendor-analysi.png)

![network](/images/main-dev-vendor-analysi-network.png)

![build](/images/main-dev-vendor-analysi-build.png)

* 对比 `cyber-react-mfe-main` 目录 `node_modules` 重新执行 yarn run vendor

![analysis plugin](/images/main-dev-vendor-analysi-node-module-run-vendor.png)

![network](/images/main-dev-vendor-analysi-node-module-run-vendor-network.png)

![build](/images/main-dev-vendor-analysi-node-module-run-vendor-build.png)

## manifest.json

![compare manifest.json](/images/manifest-json.png)

左侧在 cyber-scripts 中打包

    context: __dirname

右侧在 项目 node_modules/.../cyber-scripts/ 目录想打包

    context: __dirname

右侧的这种方式是在本地重新执行 `yarn run vendor`, 改变了模块 id, 并且在运行 yarn start 时， DllReferencePlugin 可以匹配到模块 id， vendor 生效。

## 原因

* yarn run vendor

    context: __dirname

    option.context: /Users/xxx/Documents/Dev/tzzy/dev/cyber-scripts/scripts

    module context: /Users/xxx/Documents/Dev/tzzy/dev/cyber-scripts/node_modules/axios/index.js

webpack 

```javascript
    // `LibManifestPlugin.js` line 52
    const manifest = {
        name,
        type: this.options.type,
        content: Array.from(
            chunkGraph.getOrderedChunkModulesIterable(
                chunk,
                compareModulesById(chunkGraph)
            ),
            module => {
                if (
                    this.options.entryOnly &&
                    !moduleGraph
                        .getIncomingConnections(module)
                        .some(c => c.dependency instanceof EntryDependency)
                ) {
                    return;
                }
                const ident = module.libIdent({ // right here
                    context: this.options.context || compiler.options.context,
                    associatedObjectForCache: compiler.root
                });
                if (ident) {
                    const exportsInfo = moduleGraph.getExportsInfo(module);
                    const providedExports = exportsInfo.getProvidedExports();
                    /** @type {ManifestModuleData} */
                    const data = {
                        id: chunkGraph.getModuleId(module),
                        buildMeta: module.buildMeta,
                        exports: Array.isArray(providedExports)
                            ? providedExports
                            : undefined
                    };
                    return {
                        ident,
                        data
                    };
                }
            }
        )
            .filter(Boolean)
            .reduce((obj, item) => {
                obj[item.ident] = item.data;
                return obj;
            }, Object.create(null))
    };
```

```javascript
    // util/identifier.js line 129
    /**
     * @param {string} context absolute context path
     * @param {string} request any request string may containing absolute paths, query string, etc.
     * @returns {string} a new request string avoiding absolute paths when possible
     */
    const _contextify = (context, request) => {
        return request
            .split("!")
            .map(r => absoluteToRequest(context, r))
            .join("!");
};
    
```

最终生成的 模块 ident id `../node_modules/axios/index.js`

* `yarn start` or `yarn run build`

注意：这时执行的命令是在具体的项目中， `__dirname` 返回值发生了改变

    option.context: /Users/xxx/Documents/Dev/tzzy/dev/cyber-react-mfe-main/node_modules/@cyber-insight/cyber-scripts/scripts

    option.context: /Users/xxx/Documents/Dev/tzzy/dev/cyber-react-mfe-main/node_modules/axios

```javascript
    // DllReferencePlugin.js line 116

    // DelegatedModuleFactoryPlugin.js line 75
    normalModuleFactory.hooks.module.tap(
    "DelegatedModuleFactoryPlugin",
    module => {
            const request = module.libIdent(this.options); // right here
            if (request) {
                if (request in this.options.content) { // line 80
                    const resolved = this.options.content[request];
                    return new DelegatedModule(
                        this.options.source,
                        resolved,
                        this.options.type,
                        request,
                        module
                    );
                }
            }
            return module;
        }
    );
```

最终生成的 模块 ident id `../../../axios/index.js`

在 line: 80 没有匹配到 key， 所以将 axios 添加到 index.js 中

## 解决办法

* 修改 DllReferencePlugin `context`

```javascript

       new webpack.DllPlugin({
        path: `${vendorPath}/manifest.min-${vendorVersion}.json`,
        name: '[name]_min',
        context: __dirname //????
      }),

        new webpack.DllReferencePlugin({
            context: path.resolve(__dirname, '../../../../whaterver-not-save-packages'),
            manifest: require(`${vendorPath}/manifest-${vendorVersion}.json`)
        })

```

* 删除 context， 使用 webpack 默认的  context，项目根目录

```javascript

new webpack.DllPlugin({
    path: `${vendorPath}/manifest.min-${vendorVersion}.json`,
    name: '[name]_min',
}),

new webpack.DllReferencePlugin({
    manifest: require(`${vendorPath}/manifest-${vendorVersion}.json`)
})
```

* vendor 独立打包， 也是 webpack 现有的做法



[DllPlugin](https://www.webpackjs.com/plugins/dll-plugin/)

[dynamic-link library](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-libraries?redirectedfrom=MSDN)
