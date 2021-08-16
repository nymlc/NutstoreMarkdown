```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      chunks: 'async', // 提取 chunks 的时候从哪里提取，如果为 all 那么不管是不是 async 的都可能被抽出 chunk，为 initial 则会从非 async 里面提取。
      minSize: 30000, // byte, == 30 kb，越大那么单个文件越大，chunk 数就会变少（针对于提取公共 chunk 的时候，不管再大也不会把动态加载的模块合并到初始化模块中）当这个值很大的时候就不会做公共部分的抽取了
      maxSize: 0, // 文件的最大尺寸，优先级：maxInitialRequest/maxAsyncRequests < maxSize < minSize，需要注意的是这个如果配置了，umi.js 就可能被拆开，最后构建出来的 chunkMap 中可能就找不到 umi.js 了。
      minChunks: 1, // 被提取的一个模块至少需要在几个 chunk 中被引用，这个值越大，抽取出来的文件就越小
      maxAsyncRequests: 5, // 在做一次按需加载的时候最多有多少个异步请求，为 1 的时候就不会抽取公共 chunk 了
      maxInitialRequests: 3, // 针对一个 entry 做初始化模块分隔的时候的最大文件数，优先级高于 cacheGroup，所以为 1 的时候就不会抽取 initial common 了。
      automaticNameDelimiter: '~', // 文件名分隔符
      name: true, // chunk 的名称，如果设置为固定的字符串那么所有的 chunk 都会被合并成一个，这就是为什么 umi 默认只有一个 vendors.async.js。
      cacheGroups: { // 自定义规则，会继承和覆盖上面的配置
        vendors: {
          test: /[\\/]node_modules[\\/]/, // test 符合这个规则的才会加到对应的 group 中
          priority: -10 // 一个模块可能属于多个 chunkGroup，这里是优先级，自定义的 group 是 0
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true // 如果该chunk包含的modules都已经另一个被分割的chunk中存在，那么直接引用已存在的chunk，不会再重新产生一个
        },
		common: { // 这个不是默认的，我自己加的
		  filename: '[name].bundle.js', // chunks 为 initial 时有效。在 manifest 中最后会是 '[name].js': [name].bundle.js。在 umi 中该项默认值是 [name].async.js，webpack 默认值是 [name].js。
		  name: 'common', // 和 filename 的作用类似
          chunks: 'initial',
          minChunks: 1,
		  enforce: true, // 不管 maxInitialRequest maxAsyncRequests maxSize minSize 怎么样都会生成这个 chunk
		}
      }
    }
  }
};

```

