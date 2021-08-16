这个就是处理`statusCode`的一个库，代码很简单，稍微过一下

```js
/*!
 * statuses
 * Copyright(c) 2014 Jonathan Ong
 * Copyright(c) 2016 Douglas Christopher Wilson
 * MIT Licensed
 */

'use strict'

/**
 * Module dependencies.
 * @private
 */

var codes = require('./codes.json')

/**
 * Module exports.
 * @public
 */

module.exports = status

// status code to message map
// 形如{ code: message }
status.message = codes

// status message (lower-case) to code map
// 形如{ message: code }
status.code = createMessageToStatusCodeMap(codes)

// array of status codes
// code列表
status.codes = createStatusCodeList(codes)

// status codes for redirects
// 重定向
status.redirect = {
    300: true,
    301: true,
    302: true,
    303: true,
    305: true,
    307: true,
    308: true
}

// status codes for empty bodies
// 三个空body的状态码
status.empty = {
    204: true,
    205: true,
    304: true
}

// status codes for when you should retry the request
// 三个需要重新请求的状态码
status.retry = {
    502: true,
    503: true,
    504: true
}

/**
 * Create a map of message to status code.
 * @private
 */
// 构建形如{ message: code }
function createMessageToStatusCodeMap(codes) {
    var map = {}

    Object.keys(codes).forEach(function forEachCode(code) {
        var message = codes[code]
        var status = Number(code)

        // populate map
        map[message.toLowerCase()] = status
    })

    return map
}

/**
 * Create a list of all status codes.
 * @private
 */
// 返回code列表
function createStatusCodeList(codes) {
    return Object.keys(codes).map(function mapCode(code) {
        return Number(code)
    })
}

/**
 * Get the status code for given message.
 * @private
 */

function getStatusCode(message) {
    var msg = message.toLowerCase()
		// 简单错误判定
    if (!Object.prototype.hasOwnProperty.call(status.code, msg)) {
        throw new Error('invalid status message: "' + message + '"')
    }

    return status.code[msg]
}

/**
 * Get the status message for given code.
 * @private
 */

function getStatusMessage(code) {
  	// 做个简单的判断，看看传入的code是不是status.message的自有属性
    if (!Object.prototype.hasOwnProperty.call(status.message, code)) {
        throw new Error('invalid status code: ' + code)
    }

    return status.message[code]
}

/**
 * Get the status code.
 *
 * Given a number, this will throw if it is not a known status
 * code, otherwise the code will be returned. Given a string,
 * the string will be parsed for a number and return the code
 * if valid, otherwise will lookup the code assuming this is
 * the status message.
 *
 * @param {string|number} code
 * @returns {number}
 * @public
 */

function status(code) {
  	// 返回对应的message
    if (typeof code === 'number') {
        return getStatusMessage(code)
    }
		// 报错，传入的必须是字符串或者数值
    if (typeof code !== 'string') {
        throw new TypeError('code must be a number or string')
    }

    // '403'
  	// 处理NaN
    var n = parseInt(code, 10)
    if (!isNaN(n)) {
        return getStatusMessage(n)
    }
		// 返回对应的code
    return getStatusCode(code)
}
```

可见它主体是`export`一个`status`方法，然后在该方法上挂载一些静态方法

这里关键在于`codes.json`，该库就是通过`code\message`来获取对应的`code\message`等一些操作

```json
{
    "100": "Continue",
    "101": "Switching Protocols",
    "102": "Processing"
    // ...
}
```

# API

+ status：这个就是做了个兼容了，传入`code`就返回对应的`message`，传入`message`就返回对应的`code
+ status.message: 形如`{ code: message }`
+ status.code: 形如`{ message: code }`
+ status.codes ：`code`列表，形如[100, 101]
+ status.redirect：用于判断指定的`statusCode`是否需要重定向
+ status.empty：用于判断指定的`statusCode`的`body`是否是空
+ status.retry：用于判断指定的`statusCode`是否需要重新请求

