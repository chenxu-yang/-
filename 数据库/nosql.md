# nosql

## 为什么会有nosql

![截屏2020-08-29 下午2.29.46](/Users/chenxu/Library/Application Support/typora-user-images/截屏2020-08-29 下午2.29.46.png)

![截屏2020-08-29 下午2.30.06](/Users/chenxu/Library/Application Support/typora-user-images/截屏2020-08-29 下午2.30.06.png)

## nosql分类

![截屏2020-08-29 下午2.31.31](/Users/chenxu/Library/Application Support/typora-user-images/截屏2020-08-29 下午2.31.31.png)

聚合性数据库还是有隐式的结构：比如使用mongodb还是要定义schema

## 一致性

* 逻辑一致性：多个用户造成的问题

  多个用户同时读写同一个文件

* 副本一致性：多个副本造成的问题

  多个副本未同时更新，可以使用版本号来管理

## 什么是CAP

在partition出现时如何平衡Consistency和availability



## MongoDB

![截屏2020-08-29 下午2.49.54](/Users/chenxu/Library/Application Support/typora-user-images/截屏2020-08-29 下午2.49.54.png)

## 总结：

* nosql的起源是把一个东西的数据聚集在一起
* cap是节点被分割后，如何平衡一致性和延迟
* 很多用户和很多副本都会带来一致性问题

