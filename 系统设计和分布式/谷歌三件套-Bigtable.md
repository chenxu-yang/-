# Bigtable

## 如何在文件内快速查询

![截屏2020-08-29 上午9.19.41](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.19.41.png)

## 如何保存一个很大的表

![截屏2020-08-29 上午9.21.17](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.21.17.png)



## 如何保存一个超大表

![截屏2020-08-29 上午9.24.22](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.24.22.png)

## 如何向表写数据

![截屏2020-08-29 上午9.26.27](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.26.27.png)

内存表过大时：将memTable导入硬盘，成为一个新的SStable

## 如何避免丢失数据

![截屏2020-08-29 上午9.28.28](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.28.28.png)

## 如何读数据

![截屏2020-08-29 上午9.29.50](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.29.50.png)

加速读数据的方法：加索引

![截屏2020-08-29 上午9.32.14](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.32.14.png)

如何继续加速：bloomfilter 看b是否在这个table里 ![截屏2020-08-29 上午9.33.23](/Users/chenxu/Library/Application Support/typora-user-images/截屏2020-08-29 上午9.33.23.png)

构建bloomfilter:![截屏2020-08-29 上午9.34.56](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.34.56.png)

存一个数据时 生成两个hash，并将其位置1，查找时看对应两位是否为1，存在误判。

## 如何存进GFS

 ![截屏2020-08-29 上午9.51.09](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.51.09.png)

## 表的逻辑视图

![截屏2020-08-29 上午9.54.08](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.54.08.png)

## bigtable的架构



![截屏2020-08-29 上午9.58.24](/Users/chenxu/Documents/GitHub/learning-OS/imgs/截屏2020-08-29 上午9.58.24.png)



Bigtable使用<key,value>的物理结构存储了超大表

吃内存的bigtable和吃硬盘的GFS完美配合。