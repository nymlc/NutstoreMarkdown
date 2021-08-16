其实它就是**发布/订阅**模式，不过看了源码发现和想的不大一样，它用了个优化方法

# 钩子类型

![image-20200826135035765](https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/image-20200826135035765.png)

借用一张网图，它总共提供10个钩子，主要分为两类：同步和异步。异步钩子也分为两类：并行和串行

<table>
    <tr>
        <td colspan="2">类型</td>
        <td>钩子名称</td>
        <td>说明</td>
  	</tr>
    <tr>
        <td rowspan="4" colspan="2">同步</td>
        <td>SyncHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>SyncBailHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>SyncWaterfallHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>SyncLoopHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td rowspan="6">异步</td>
        <td rowspan="2">并行</td>
        <td>AsyncParallelHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>AsyncParallelBailHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td rowspan="4">串行</td>
        <td>AsyncSeriesHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>AsyncSeriesBailHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>AsyncSeriesLoopHook</td>
        <td>同步钩子</td>
    </tr>
    <tr>
        <td>AsyncSeriesWaterfallHook</td>
        <td>同步钩子</td>
    </tr>
</table>



