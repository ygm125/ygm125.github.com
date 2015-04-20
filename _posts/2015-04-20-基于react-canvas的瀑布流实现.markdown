---
layout: post
title:  "基于react-canvas的瀑布流实现"
categories: js
---

今年比较火的无疑是react了，官网地址 [http://facebook.github.io/react/](http://facebook.github.io/react/)

照着官网做了几个例子基本也很简单，用法相信会js的基本无障碍

demo做下来的基本感觉就是所有的东西都是组件，组件也可以和组件搭配组合使用，页面就由这些组件组成，

回想起来和.net还有点像，拖拖控件分分钟做完一个页面，只不过上面的组件大都人工合成，灵活力度更强

不过这里介绍的不是react，而是 [react-canvas](https://github.com/Flipboard/react-canvas)

它是一个基于react渲染在canvas上的项目。

始因是flipboard体验dom和动画在手机性能低下而做的一个 [新的尝试](http://engineering.flipboard.com/2015/02/mobile-web/)。

我司正好前段时间说瀑布流页面滑动一定程度就会出现闪退，

借着热风尝试搞起。

首先来到它的 [git地址](https://github.com/Flipboard/react-canvas)，可以看到已经封装好的一些组件。(建议先大概浏览器组件的用法)

	clone项目到本地
	npm install
	npm start


即可看到官方提供的例子。

例子里listview和我要做的感觉比较像，于是就在这上进行改造

我要做的就是 每行显示俩个单品，每个单品有图片和文字描述等信息，

数据动态获取，加载时有加载的提示，恩目前就这样。

查看例子代码，可以看到数据是直接引入的静态数据，

ListView的numberOfItemsGetter数也是写死的，

这些我们都先不管，先把页面绘制成我们想要的。

首先app.js下的renderItem每次只传给Item组件一个数据，

而我要每行展示俩个，这里进行改造

	renderItem: function (itemIndex, scrollTop) {
    	var rowNum = Item.getRowNums();
    	var rowData = [];

    	for (var i = itemIndex; i < itemIndex + rowNum; i++) {
        	rowData.push(data[itemIndex + i]);
   	 	};

      //loding相关
    	if(itemIndex + 1 === this.getNumberOfItems()){
        	var loding = true;
    	}

    	return (
      	<Item
        	loding={loding}
        	width={this.getSize().width}
        	height={Item.getItemHeight()}
        	data={rowData}
        	itemIndex={itemIndex} />
    	);
    	}

Item.getRowNums是Item里加的一个静态方法

	getRowNums: function(){
  		return 2; 
	}

这样Item每次接受俩个数据进行渲染，Item.js继续改造

	render: function () {
    var self = this;

    var rowItems = this.props.data.map(function(item,index){
        var rowImageStyle = self.getImageStyle(index);
        var rowTitStyle = self.getTitleStyle(index);
        var priceStyle = self.getPriceStyle(index);
        var btnStyle = self.getBtnStyle(index);

        var text = self.truncate(item.title,self.getTitleStyle(index));

        return (
            <Group>
              <Image style={rowImageStyle} src={item.pic} fadeIn={true}  />
              <Group>
                <Text style={rowTitStyle}>{text}</Text>
                <Text style={priceStyle}>{item.price}</Text>
                <Text onClick={self.btnClick(item)} style={btnStyle}>马上抢购</Text>
              </Group>
            </Group>
        );
    });

    //loding相关
    if(this.props.loding){
      return (
        <Group>
          <Text style= { { width:this.props.width,height:30,textAlign:'center' } }>loding...</Text>
        </Group>
      )
    }

    return (
      <Group style={this.getStyle()}>
        {rowItems}
      </Group>
    );
  	}

循环数据，进行绘制展示，CSSLayout与ListView样式冲突，这里都是基于定位布局。

这里碰到俩个问题

1、文字长度问题，每行显示俩个，文字信息超长就会不显示

这里写了个截取方法

	truncate : function(text,style){
    	var textMetrics = measureText(text, 1000, style.fontFace, style.fontSize, style.lineHeight);
    	var actualWidth = textMetrics.width;
    	if(actualWidth >= style.width){
      	var perLen = actualWidth / text.length;
      	var textNum = parseInt(style.width / perLen);
      	text = text.substr(0, textNum - 2 ) + '...';
    	}
    	return text;
  	}
  
利用canvas获取文字长度然后处理，大家可以看看measureText部分源码。

2、页面肯定需要交互，大多来自事件，绑定事件要获取一些参数，比如 点击跳转相应链接。

找到的一种方法是放到style里获取，另一种是绑了个闭包，如下
	
	btnClick : function(data){
    return function(){
       location.href = data.link;
    }
  	}
  
至此页面基本可看，但还有个主要问题，就是动态加载，

这里着实没找到好方法，在查找过程中，我把焦点放到了ListView的onScroll方法上，

每次滚动都会调用onScroll方法。

这里要做的就是判断何时去加载数据

	scroll:function(listViewIns,top){
      var self = this;
      if(typeof this.scrollState === 'undefined'){
          this.scrollState = 1;
      }

      var scrollHeight = this.getNumberOfItems() * Item.getItemHeight();
      var distance = Item.getItemHeight() * 2;

      if(this.scrollState && (top + distance >= scrollHeight) ){

        this.scrollState = 0;

        var style = this.getListViewStyle();

        listViewIns.scroller.setDimensions(style.width, style.height, style.width, scrollHeight - (Item.getItemHeight() - Item.getLodingHeight()));

        setTimeout(function(){
            data = data.concat(data);
            
            listViewIns.updateScrollingDimensions();
            self.scrollState = 1;
        },2000);
      }
  	}
  	

这里我改动了react-canvas ListView里的onScroll方法，多传了一个listView实例进来，因为后面要用 ==#

	handleScroll: function (left, top) {
    this.setState({ scrollTop: top });
    if (this.props.onScroll) {
      this.props.onScroll(this,top);
    }
  	}

ListView集成了scroller项目大家可以去看看基本用法。

listViewIns.scroller.setDimensions 做了一个区域的限制，也就是没数据的时候滚动要限制在范围内，

后面就是动态加载数据和更新滚动区域范围的操作，这里做了一个假的数据获取。

到此一个差不多的例子总算出来了。

全过程下来 感觉开发成本还是蛮高的，还有就是这个项目也在完善开发中，相关的资料甚少。

实际项目效果有待考量，代码写的相较零碎，后续如果有应用在更新吧。


最后补上例子代码地址 [传送门](https://github.com/ygm125/react-canvas/tree/master/examples/waterfall)


good job~



