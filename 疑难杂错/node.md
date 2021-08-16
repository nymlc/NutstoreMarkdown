1. not executable. try chmod or run with root

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1603092518006.png" alt="image-20201019152835445" style="zoom:50%;" />

如图所示，`bin`文件夹下，`yx`为入口文件，按理而言`yx cert`可执行，却报图中错误，[查询可得](https://github.com/tj/commander.js/issues/1041)，文件加上`.js`后缀即可

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1603092716752.png" alt="image-20201019153155009" style="zoom:50%;" />

然而还会有如上错误：`Error: ENOENT: no such file or directory`，我们只需要在`bin`文件夹下创建一个`yx`空文件夹即可，[来自于此](https://github.com/tj/commander.js/issues/527)，`.gitkeep`是因为`git`不能追踪空文件夹

<img src="https://cdn.jsdelivr.net/gh/nymlc/picgo@master/uPic/1603093157409.png" alt="image-20201019153913721" style="zoom:50%;" />

