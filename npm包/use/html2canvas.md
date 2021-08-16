# 图片的object-fit问题

基于`html2canvas 1.0.0-rc.7`

就是图片样式为`object-fit: contain`的话就会导致截出来的图片填满整个`img`区域，导致显示拉伸

```js
function polyfill(container, element, box) {
    let isPolyfill = true
    switch(element.tagName.toLocaleLowerCase()) {
        case 'img':
            // 简单解决object-fit: contain
            const { width, height, left, top } = box
            const { intrinsicWidth, intrinsicHeight } = container
            const intrinsicScale = intrinsicWidth / intrinsicHeight
            const scale = width / height
            
            const resultBox = {
                left, top, width, height
            }
            if (intrinsicScale < scale) {
                // 以box高度为准，高度撑满容器
                resultBox.width = intrinsicScale * height
                resultBox.left = (width - resultBox.width) / 2 + left
            } else if (intrinsicScale > scale) {
                // 以box宽度为准，宽度撑满容器
                resultBox.height = width / intrinsicScale
                resultBox.top = (height - resultBox.height) / 2 + top
            } else {
                // 比例一致，不予处理
            }
            this.ctx.drawImage(element, 0, 0, intrinsicWidth, intrinsicHeight, resultBox.left, resultBox.top, resultBox.width, resultBox.height);
            break
        default:
            isPolyfill = false
    }
    return isPolyfill
}

CanvasRenderer.prototype.renderReplacedElement = function (container, curves, image) {
    if (image && container.intrinsicWidth > 0 && container.intrinsicHeight > 0) {
        var box = contentBox(container);
        var path = calculatePaddingBoxPath(curves);
        this.path(path);
        this.ctx.save();
        this.ctx.clip();
        if (!polyfill.call(this, container, image, box)) {
            this.ctx.drawImage(image, 0, 0, container.intrinsicWidth, container.intrinsicHeight, box.left, box.top, box.width, box.height);
        }
        this.ctx.restore();
    }
};
```

这里就是根据图片原本的长宽比例，调用`drawImage`方法来绘画，防止图片被拉伸

> ```js
> drawImage(image, sx, sy, sw, sh, dx, dy, dw, dh)
> ```
>
> 

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1616663301805.png" alt="image-20210325170814897" style="zoom:50%;" />

