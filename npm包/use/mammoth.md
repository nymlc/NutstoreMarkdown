一个能够把`word`转成可看的`html`的工具，具体操作[可见于此](https://github.com/mwilliamson/mammoth.js)

有一些比如字体颜色之类的并不支持（[简单处理](https://github.com/mwilliamson/mammoth.js/issues/109)），不过瑕不掩瑜

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1606468516147.png" alt="image-20201127171512455" style="zoom:50%;" />

```js
// index.js
const mammoth = require('mammoth');
const mime = require("mime")
const fs = require('fs');
const _ = require('underscore');
const program = require('commander');

let fileName
let entryPath
let outputPath
let imagePath1
let imagePath2

function init([name = 'doc']) {
    fileName = name
    entryPath = `./word/${fileName}.docx`;
    outputPath = `./output/${fileName}.html`;
    imagePath1 = `./output/${fileName}/`
    imagePath2 = `./${fileName}/`
}

function transformElement(element) {
    if (element.children) {
        var children = _.map(element.children, transformElement);
        element = { ...element, children: children };
    }
    if (element.type === 'paragraph') {
        element = transformParagraph(element);
    }
    return element;
}

function transformParagraph(element) {
    if (element.alignment === 'center' && !element.styleId) {
        return { ...element, styleName: 'center' };
    } else {
        return element;
    }
}

var options = {
    styleMap: ['u => u', "p[style-name='center'] => p.center"],
    transformDocument: transformElement,
    convertImage: mammoth.images.imgElement(function (image) {
        return image.read("base64").then(async (imageBuffer) => {
            const path = await saveBase64Image(imageBuffer, image.contentType);
            return {
                src: path // 获取图片地址
            };
        });
    })
};
let imgIndex = 0
async function saveBase64Image(base64Image, contentType) {
    const ext = mime.getExtension(contentType)
    base64Image = base64Image.replace(/^data:image\/png;base64,/, "")
    const suffix = `${++imgIndex}.${ext}`
    return new Promise((resolve, reject) => {
      	// 先递归创建文件夹
        fs.mkdir(imagePath1, { recursive: true }, (err) => {
            if (err) reject(err)
          	// 把base64保存成文件图片
            fs.writeFile(`${imagePath1}${suffix}`, base64Image, 'base64', function (err) {
                if (err) {
                    reject(err)
                } else {
                    resolve(`${imagePath2}${suffix}`)
                }
            })
        })
    })
}


program.usage("<project-name>").parse(process.argv);


// 根据输入，初始化参数
init(program.args)

mammoth
    .convertToHtml({ path: entryPath }, options)
    .then(function (result) {
        var html = result.value; // The generated HTML
        html = html.replace(/color="/g, 'style="color:#')
        var messages = result.messages; // Any messages, such as warnings during conversion
        fs.writeFile(outputPath, html, res => {
            console.log('文件写入成功：', messages);
        })
    })
    .done();
```

使用也很简单，`node ./index.js doc`（就是转换`word`文件夹下的`doc.docx`）

