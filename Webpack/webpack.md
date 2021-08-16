# 初始化

## new Webpack
也就是`new webpack()`，说个常见的，就是`pkg.json`里的`scripts: { "build": "webpack" }`，这个其实是分为以下几步：
1. 执行`./node_modules/webpack/bin/webpack.js`
2. 跳转到`./node_modules/webpack-cli/bin/cli.js`
3. 执行到`./node_modules/webpack/lib/webpack.js`(`new webpack()`)

`webpack-cli`就是个命令行工具，主要是用来解析配置文件以及接收来自`CLI`的配置

## new Compiler

```js
// ./node_modules/webpack/lib/webpack.js
const webpack = (options, callback) => {
    // ...

    let compiler;
	if (Array.isArray(options)) {
		compiler = new MultiCompiler(
			Array.from(options).map(options => webpack(options))
		);
	} else if (typeof options === "object") {
        // ...
        compiler = new Compiler(options.context);
        
        // ...

		if (options.plugins && Array.isArray(options.plugins)) {
			for (const plugin of options.plugins) {
				if (typeof plugin === "function") {
					plugin.call(compiler, compiler);
				} else {
					plugin.apply(compiler);
				}
			}
        }
        // ...
		compiler.options = new WebpackOptionsApply().process(options, compiler);
        // ...
	} else {
		throw new Error("Invalid argument: options");
	}
	if (callback) {
        // ...
		compiler.run(callback);
	}
	return compiler;
};
```
把一些代码去掉之后就很清晰了，首先就是实例化`Compiler`对象，其次就是加载插件，最后是执行`compiler`的`run`方法

# 编译

## Compiler

```js
class Compiler extends Tapable {
	constructor(context) {
        super();
        // 初始化一些hooks，具体含义可见官网  https://www.webpackjs.com/api/compiler-hooks/
		this.hooks = {
			shouldEmit: new SyncBailHook(["compilation"]),
			done: new AsyncSeriesHook(["stats"]),
			additionalPass: new AsyncSeriesHook([]),
			beforeRun: new AsyncSeriesHook(["compiler"]),
			run: new AsyncSeriesHook(["compiler"]),
			emit: new AsyncSeriesHook(["compilation"]),
			assetEmitted: new AsyncSeriesHook(["file", "content"]),
			afterEmit: new AsyncSeriesHook(["compilation"]),

			compilation: new SyncHook(["compilation", "params"]),
			normalModuleFactory: new SyncHook(["normalModuleFactory"]),
			contextModuleFactory: new SyncHook(["contextModulefactory"]),

			beforeCompile: new AsyncSeriesHook(["params"]),
			compile: new SyncHook(["params"]),
			make: new AsyncParallelHook(["compilation"]),
			afterCompile: new AsyncSeriesHook(["compilation"]),
            // ...
		};
	}

	run(callback) {
		const finalCallback = (err, stats) => {
			this.running = false;

			if (err) {
				this.hooks.failed.call(err);
			}

			if (callback !== undefined) return callback(err, stats);
		};


		this.running = true;

		const onCompiled = (err, compilation) => {
			if (err) return finalCallback(err);

			if (this.hooks.shouldEmit.call(compilation) === false) {
				const stats = new Stats(compilation);
				this.hooks.done.callAsync(stats, err => {
					if (err) return finalCallback(err);
					return finalCallback(null, stats);
				});
				return;
			}

			this.emitAssets(compilation, err => {
				if (err) return finalCallback(err);

				if (compilation.hooks.needAdditionalPass.call()) {
					this.hooks.done.callAsync(stats, err => {
						if (err) return finalCallback(err);

						this.hooks.additionalPass.callAsync(err => {
							if (err) return finalCallback(err);
							this.compile(onCompiled);
						});
					});
					return;
				}

				this.emitRecords(err => {
					if (err) return finalCallback(err);

					this.hooks.done.callAsync(stats, err => {
						if (err) return finalCallback(err);
						return finalCallback(null, stats);
					});
				});
			});
		};

		this.hooks.beforeRun.callAsync(this, err => {
			if (err) return finalCallback(err);

			this.hooks.run.callAsync(this, err => {
				if (err) return finalCallback(err);

				this.readRecords(err => {
					if (err) return finalCallback(err);

					this.compile(onCompiled);
				});
			});
		});
	}

	createCompilation() {
		return new Compilation(this);
	}

	newCompilation(params) {
        const compilation = this.createCompilation();
        // ...
		this.hooks.compilation.call(compilation, params);
		return compilation;
	}

	newCompilationParams() {
		const params = {
			normalModuleFactory: this.createNormalModuleFactory(),
			contextModuleFactory: this.createContextModuleFactory(),
			compilationDependencies: new Set()
		};
		return params;
	}

	compile(callback) {
		const params = this.newCompilationParams();
		this.hooks.beforeCompile.callAsync(params, err => {
			if (err) return callback(err);

			this.hooks.compile.call(params);

			const compilation = this.newCompilation(params);

			this.hooks.make.callAsync(compilation, err => {
				if (err) return callback(err);

				compilation.finish(err => {
					if (err) return callback(err);

					compilation.seal(err => {
						if (err) return callback(err);

						this.hooks.afterCompile.callAsync(compilation, err => {
							if (err) return callback(err);

							return callback(null, compilation);
						});
					});
				});
			});
		});
	}
}

module.exports = Compiler;
```
[钩子详见](https://www.webpackjs.com/api/compiler-hooks/)
简单修剪下代码，我们可见主流程其实还是很简单的
1. 实例化时初始化了一堆`hooks`
2. 触发`hooks.beforeRun`钩子，在其回调里继续触发`hooks.run`且在回调里触发`compile`方法正式进入编译
3. 触发`hooks.beforeCompile`钩子，在其回调里触发`hooks.compile、hooks.make`，在`hooks.make`回调里触发`compilation.finish`


## Compilation
```js
// SingleEntryPlugin
class SingleEntryPlugin {
	constructor(context, entry, name) {
		this.context = context;
		this.entry = entry;
		this.name = name;
	}

	apply(compiler) {
		compiler.hooks.compilation.tap(
			"SingleEntryPlugin",
			(compilation, { normalModuleFactory }) => {
				compilation.dependencyFactories.set(
					SingleEntryDependency,
					normalModuleFactory
				);
			}
		);

		compiler.hooks.make.tapAsync(
			"SingleEntryPlugin",
			(compilation, callback) => {
				const { entry, name, context } = this;

				const dep = SingleEntryPlugin.createDependency(entry, name);
				compilation.addEntry(context, dep, name, callback);
			}
		);
	}

	static createDependency(entry, name) {
		const dep = new SingleEntryDependency(entry);
		dep.loc = { name };
		return dep;
	}
}

module.exports = SingleEntryPlugin;
```
1. 在入口文件`webpack`里执行到`new WebpackOptionsApply()`，而`WebpackOptionsApply`里又执行了`new EntryOptionPlugin().apply(compiler)`，最终到了`SingleEntryPlugin`
2. `SingleEntryPlugin`里给`hooks.make`注册了钩子，里面有关键逻辑`compilation.addEntry`，由此到了`Compilation`
3. 