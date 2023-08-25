---
layout: post
title: UITableView的Cell重用机制以及如何解决卡顿
description: UITableView中Cell的重用机制，以及如何解决有大量Cell时卡顿的问题
category: blog
tag: ios
---

## UITableView的重用机制

对于UITableView,大部分情况下它的Cell的样式都是一样的，不同的就是其Cell上的展示数据不同。

UITableView的重用机制是它只会创建一屏幕(一般都是一屏幕多1—2个)的Cell,当用户滑动时，它会取出以前创建的Cell去填充数据来重用以前创建的Cell。通过这种机制来提高加载效率和优化内存，避免不停地创建和销毁Cell . 

我们在开发中一般都是通过这种方式来写其重用机制：

```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static NSString *cellIdentifer = @"Cell_identifer";
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifer];
    if (!cell) {
        cell  = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:cellIdentifer];
    }
    cell.textLabel.text = @"text";
    
    return cell;
}
```

通过调用tableView的`dequenceReusableCellWithIdentifier:`方法看指定的identifer是否有可以重复使用的，如果有则返回可复用的cell,cell就绪之后便可以为其填充数据；如果不可以复用，则返回nil，然后会创建新的cell并对其设置cell样式并标记identifer 。 只有创建了足够数量的可覆盖整个屏幕的cell之后，才会复用之前的。

我们可以用以下的方法来验证TableView的复用机制：

1. tableView设置为1组，每组有30个数据。
2. 在`tableView:cellforRowAtIndexPath:`方法中，记录复用的是哪一个cell . 

```
#pragma mark UITableViewDataSource
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
{
    return 1;
}

- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    return 30;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static NSString *cellIdentifer = @"Cell_identifer";
    static int total_num = 1;
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellIdentifer];
    if (!cell) {
        cell  = [[UITableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:cellIdentifer];
        cell.textLabel.text = [NSString stringWithFormat:@"%i",total_num];
        total_num++;
    }
    
    return cell;
}
``` 

在iphone8模拟器上运行结果如图：

![](https://raw.githubusercontent.com/MaxwellQi/MaxwellQi.github.io/master/images/blog/ios_tableview/tableView_01.png)

如上图所示，整个屏幕容纳15个cell，但是系统创建了16个cell 。 

![](https://raw.githubusercontent.com/MaxwellQi/MaxwellQi.github.io/master/images/blog/ios_tableview/tableView_02.png)

当向下滑动时，系统开始复用前面创建好的cell . 



## UITableView优化防卡顿

我们在使用UITableView时，经常出现滑动过程中卡顿的现象，造成用户体验非常差，由此我们可以通过以下方式来优化。

### 减少cellForRowAtIndexPath代理方法中的计算量

* 首先要提前计算每个cell中需要的一些基本数据，代理方法在需要用的时候直接取出。
* 图片要异步加载，加载完成后再根据cell内部UIImageView的引用设置图片
* 图片数量多时，图片的尺寸要根据需要提前讲过transform矩阵变幻压缩好(直接设置图片的contentMode让其自行压缩仍然会影响滑动效果)，必要的时候要准备好预览图和高清图，需要时再加载高清图。
* 图片的懒加载方法，即在需要时加载，当滑动速度很快时避免频繁请求服务器数据。
* 尽量手动Drawing视图替身流畅性，而不是直接子类化UITableViewCell,然后覆盖drawRect方法，因为cell中不是只有一个contentView，绘制cell不建议使用UIView，建议使用CALayer 。 

### 减少heightForRowAtIndexPath代理方法中的计算量

* 由于每次TableView进行update更新都会重新对每一个cell调用`heightForRowAtIndexPath`代理取得最新的height，会大大增加计算时间。如果表格的所有cell高度都是固定的，那么去掉`heightForRowAtIndexPath`代理，直接设置TableView的rowHeight属性为期固定高度。
* 如果高度不固定，应尽量将cell的高度数值计算好存起来，代理调用时直接取。

### 使用不透明视图

不透明的视图可以极大地提高渲染的速度。因此，可以将Cell机器自视图的opaque属性设置为YES 。 

其中的特例包括背景色，它的alpha值应该是1（例如不要使用clearColor）;图像的alpha值也应该为1，或者在画图时设置为不透明。

### 预渲染图像

你会发现及时做到了上述几点，当新的图像出现时，仍然会有短暂的停顿现象。解决办法就是在bit map context里面先将其画一遍，到处成UIImage对象，然后再绘制到屏幕。