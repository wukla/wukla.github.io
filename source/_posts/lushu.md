---
title: 基于百度地图制作带动态数据标签的路书
tags:
  - 百度地图
  - JavaScript
categories:
  - 项目笔记
description: 百度地图自带有路书功能，可以让标志物在规定线上移动，但是原生的路书功能太少。需求就是在标志物上带上一个标签动态显示速度、时间等信息。
date: 2019-01-24 17:27:49
---


# 路书

百度地图的路书是基于Baidu Map API 1.2+版本的一个功能，他可以实现marker沿路线运动，并且提供暂停等功能。

这是github上的地址：

[baidumap/LuShu](https://github.com/baidumap/LuShu)

以及它的Demo：

[Demo](http://lbsyun.baidu.com/jsdemo.htm#c2_8)

需要的同学可以自行下载。

## 使用之前

在正式写代码之前要先把百度地图的SDK引入。

```javascript
<script src="http://api.map.baidu.com/api?v=2.0&ak=你的密钥">
<script type="text/javascript" src="http://api.map.baidu.com/library/LuShu/1.2/src/LuShu_min.js">
```

## 带标签的路书

我们的需求是让定制路线上的marker带有一个动态显示数据的标签，然而原版的路书Demo中是指定一个起始点和终点，由百度地图规划出路线。那么我们就需要解决两个问题：

1. 如何制定出我们想要的路线，而不是依靠百度地图搜索规划出的路线。
2. 如何随着标志物运动时显示带有动态数据的标签。

* 扩展：还可以在暂停启动的基础上加入播放条功能。

### 定制路径覆盖物

由于数据都已经采集完成放在数据库中的，因此定制路线并不困难。首先在回调函数中把从数据库中取出的的经纬度信息和其他信息保存在`result`对象中。

由于采集到的数据是WGS84的，然而百度地图的GPS坐标是采用GPS09定位，因此还需要遍历每个点对经纬度坐标进行转换。最后将点全部放入`arrPois`数组中，创建地图实例时将根据这个数组创建覆盖物（我们定制的路径）。

这里仅贴出**部分**关键代码，ajax请求部分和坐标转换函数省略。

```javascript
       	var arrPois = [];
			$.ajax(function(){
            /**上面省略**/
            success: function (result) {
                if(result.length <= 0 ){
                    alert("未查询到行驶信息！");
                    return;
                }
                //转换坐标
                var coord;
                for(var i=0; i<result.length;i++){
                    coord = CoordinateTransform.transformWGS84ToBD09(result[i]												["lng"],result[i]["lat"]);
                    //转换Unix时间戳
                    result[i]["lng"] = coord[0];
                    result[i]["lat"] = coord[1];
                }
                //把点存到路径数组中
                for(var i = 0; i < result.length-1 ; i++){
                    arrPois.push(new BMap.Point(result[i]["lng"], result[i]["lat"]));
                }
               //创建地图实例并加载路径和标志物（小车）
                buildMap(arrPois,result);
            }
            /**略**/
         });
                 
```

创建地图的时候加载路径、标志物：

```javascript
/**
*	@param arrPois 用于创建路径的所有BMap.Point点组成的数组
*	@param result 带有其他对点的信息
*	funcaiton buildMap()
**/
function buildMap(arrPois,result){
          //路书停止标记
          var isStop;
          //路书暂停播放标记
          var isPause;
          //路书开始运动的点(arrPois[startPois])初始值为1；当暂停时先保存当前值，开始时根据保存的值继续运行；停止操作刷新startPoint为1
          var startPoint = 1;
          //以路径起点为中心
          //var point = new BMap.Point(result[0]["lng"],result[0]["lat"]);
          map.centerAndZoom(arrPois[0], 12);
          map.enableScrollWheelZoom();
          var polyline = 
              new BMap.Polyline(arrPois, {strokeColor: '#000080',strokeWeight : 2});
          //创建路径覆盖物
          map.addOverlay(polyline);
   		//调用clearOverlays()时不会清除该覆盖物
          polyline.disableMassClear();
          map.setViewport(arrPois);
          //创建车辆标志物
          var truckIcon = new BMap.Icon(static+"/monitor/images/car.png", new BMap.Size(29, 29));
          var truckMk = new BMap.Marker(arrPois[0],{icon: truckIcon});
          truckMk.disableMassClear();
          map.addOverlay(truckMk);
          addButton();
        }
```

地图实例

{% asset_img map.png 地图实例 %}

`arrPois`中某个点的实例：

伪代码：

`arrPois[0]=Point{'lat':23.274253731583542,'lng':113.32783864974276}`，上面的路径图由545个点组成。

### 为标志物(小车)添加标签

重头戏来了，因为百度地图自带的路书其实是可以在定制的路径上移动的，但是不能给标志物添加标签显示数据，更不用说随着点的遍历动态显示数据。

**在阅读了路书的源码之后，发现路书是通过让标志物依次从当前坐标跳到下一个坐标，在跳转两个点之间循环定位，显得小车是是在路径上“移动”，而不是“跳动”。**

在这基础上，给标志物加上标签应该不难。加上标志变量，标识当前播放状态。因此除了地图以外还有四个控件：开始、暂停、停止、速度调整。

思路：

* 下列函数放在在`buildMap()`中，在第一次初始化地图时，将`startPoint`设为1。
* 点击暂停时，将当前`startPoint`保存，再下次点击时从这里开始。
* 点击停止时，将当前`startPoint`重置为`1`

```javascript
$('#run').click(function(){
       isPause = false;
       isStop = false;
       var content;
       //创建卡车marker的标签
       var label= new BMap.Label('',{offset : new BMap.Size(-30,-60)
       //首次开始的进入函数
       resetMk(startPoint);
       function resetMk(step){
           //暂停操作
           if(isPause == true){
               truckPause(step);
               $('#run').attr('disabled',false);
           }
           //停止操作
           if(isStop == true){
               $('#run').attr('disabled',false);
           }
           //车辆移动
           if(isPause == false && isStop == false){
               move(arrPois[step-1],arrPois[step],linear,truckMk);
           }
           //卡车标签的内容
           content = "时间"+result[step]["gps_time"]+
               "<br/>速度"+result[step]["speed"];
           ;
           label.setContent(content);
           truckMk.setLabel(label);
           map.addOverlay(truckMk);
			//依据选择的播放速度进行循环；-2是因为move调用到arrPois最后第一个点时会发生数组越界
           if(step<result.length-2 && isPause !=true && isStop != true){
               var t = setTimeout(function(){
                   step++;
                   resetMk(step);
                   map.clearOverlays();
               },changeSpeed());
           }
           if(step >= result.length-2){
               $('#run').attr('disabled',false);
           }
       }
       //按下开始之后开始键被禁用
       $('#run').attr('disabled',true);
   //按下暂停置当前点为下一个开始点
   $('#pause').click(function(){
       isPause=true;
   });
   function truckPause(step){
       startPoint = step;
   }
   //按下停止后刷新开始点为1
   $('#stop').click(function(){
       isStop = true;
       startPoint = 1;
   });
```

### 标志物移动

* 用`setInterval()`来达到可以控制播放速度的效果。
* 调整速度其实很简单，只是给出一个速度值供其调用，但是值得注意的是，`move()`函数中的循环速度，和`restMk(step)`中的调用定时器是相互配合的，如果速度配不好就会出现"反复横跳的情况"。

```javascript
 /**
  * 小车移动
  * @param {Point} prvePoint 开始坐标(PrvePoint)
  * @param {Point} newPoint 目标点坐标
  * @param {Function} 动画效果
  * @param {BMap.Marker} 车辆标记
 *  @return  无返回值
  */
 function move(prvePoint, newPoint, effect,mk) {
     		//当前帧数
     		var   currentCount = 0,
         //初始坐标
         //将球面坐标转换为平面坐标
         _prvePoint = map.getMapType().getProjection().lngLatToPoint(prvePoint),
         //获取结束点的(x,y)坐标
         _newPoint = map.getMapType().getProjection().lngLatToPoint(newPoint),
         //两点之间要循环定位的次数
         count = 10,
         //循环标记
         _intervalFlag;
         //两点之间匀速移动
         _intervalFlag = setInterval(function () {
         //两点之间当前帧数大于总帧数的时候，则说明已经完成移动
         if (currentCount >= count) {
             clearInterval(_intervalFlag);
         } else {
             //动画移动
             currentCount++;//计数
             var x = effect(_prvePoint.x, _newPoint.x, currentCount, count),
                 y = effect(_prvePoint.y, _newPoint.y, currentCount, count);
             //根据平面坐标转化为球面坐标
             var pos = 
                map.getMapType().getProjection().pointToLngLat(new BMap.Pixel(x, y));
            //正在移动
             mk.setPosition(pos);
     }, changeSpeed()/10);
 }
 //车辆运动的缓速效果
 function linear(initPos, targetPos, currentCount, count) {
     var b = initPos,
         c = targetPos - initPos,
         t = currentCount,
         d = count;
     return c * t / d + b;
 }
```

### 角度转动

上述内容基本上已经把带标签的路书完成了，但是仔细运行的时候发现这车子不会转头啊！就是一个静态图片，一点都不像真正的车子啊。

路书的源码中其实早就给出了答案。

```javascript
function setRotation(curPos, targetPos, marker) {
    var deg = 0;
    curPos = map.pointToPixel(curPos);
    targetPos = map.pointToPixel(targetPos);
    if (targetPos.x != curPos.x) {
       //算出终点和起点之间连线的斜率
        var tan = (targetPos.y - curPos.y) / (targetPos.x - curPos.x),
            atan = Math.atan(tan);
       //求出角度
        deg = atan /180 * Math.PI;
       //如果终点值x值小于起始值，就要先转180°
        if (targetPos.x < curPos.x) {
            deg = -deg + 90 + 90;
        } else {
            deg = -deg;
        }
        marker.setRotation(-deg);
    } else {
        var disy = targetPos.y - curPos.y;
        var bias = 0;
        if (disy > 0)
            bias = -1
        else
            bias = 1
        marker.setRotation(-bias * 90);
    }
    return;
}
```

在`move()`函数中进行移动之前中加入下列语句：

```javascript
              if (currentCount == 1) {
                  //转换角度
                  setRotation(prvePoint,newPoint, mk);
              }
```

因为`move`函数是在相邻两点间循环定位，因此必定为直线，所以只需要在第一次移动时转动一次。

### 演示



# 总结

自定义一个带标签的路书总共有几个步骤：

1. 加载用来绘制路径的**一堆**经纬度点。
2. 创建地图实例，同时绘制标志物、路径覆盖物。
3. 为标志物添加标签，循环调用move函数移动标志。

本文提到的内容有一定的局限性，特别是绘制路径这里，需要多个几乎是连续的点，如果要引用请注意。