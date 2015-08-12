---
author: admin
comments: true
date: 2014-05-12 05:31:35+00:00
layout: post
slug: "\n            web%e8%87%aa%e5%8a%a8%e5%8c%96%e6%b5%8b%e8%af%95%e4%b8%ad%e7%9a%84%e9%a1%b5%e9%9d%a2%e6%a8%a1%e5%9e%8b%ef%bc%88%e4%b8%89%ef%bc%89-%e7%bb%84%e4%bb%b6%e8%ae%be%e8%ae%a1\n\
  \        "
title: WEB自动化测试中的页面模型（三）–组件设计（列表组件）
wordpress_id: 129
categories:
- Selenium WebDriver
- 自动化
---
上一章中，我们在首页页面中调用了一些组件。简单的来说，组件也可以看做一块小的页面，因此，它也会有自己的属性、方法，甚至组件里也能有组件。
很简单吧，只要把相关的属性，方法写进去就可以。

_但是，通常情况下会遇到一些个小问题：由于组件是通用的，因此不能把组件内控件的选择器用绝对路径写死，并且有的组件（如列表）是灵活多变的，可能在这个页面里有5列，在另一个页面里就只有3列了_。

如何灵活的设计组件能在各个页面中都能正确调用？我们看一个例子。

**例子**

首先，我们要确定一个东西：该组件的特征或者说是作用区域。

我们以邮箱中的通讯录列表为例，通过简单的分析，我们发现所有的通讯录列表都在一个class为nui-table的div下的table中，他的列头所在的HTML元素为thead，表体区域所在的HTML元素为nui-table-body。知道了这些信息后，我们简单的设计下该控件的各属性（以xpath选择器为例）：

**定义属性**
```java
/**
 * Grid component
 * @author Assilzm
 * createTime 2014-5-12.
 */
class Grid extends WebActions {

    /**
     * default container tag name
     */
    final static String DEFAULT_CONTAINER_TAGNAME = "div"

    /**
     * default container attribute
     */
    final static String DEFAULT_CONTAINER_ATTRIBUTE = "class"

    /**
     * default container attribute value
     */
    final static String DEFAULT_CONTAINER_ATTRIBUTEVALUE = "nui-table"

    final static String TABLE_TAGNAME = "table"

    final static String TR_TAGNAME = "tr"

    final static String TD_TAGNAME = "td"

    final static String TABLE_HEAD_TAGNAME = "thead"

    final static String TABLE_BODY_TAGNAME = "tbody"


    /**
     * current grid container selector
     */
    String containerSelector

    /**
     * current grid container selector with default value
     */
    Grid() {
        containerSelector = "//$DEFAULT_CONTAINER_TAGNAME[@$DEFAULT_CONTAINER_ATTRIBUTE='$DEFAULT_CONTAINER_ATTRIBUTEVALUE']"
    }

    /**
     * current grid container selector with custom value
     * @param selector
     */
    Grid(String selector) {
        containerSelector = selector
    }
}
    ```

在以上代码中，我们定义了一个列表（Grid）控件的默认区域，在初始化的时候，如果不传参数，那么默认这个列表所在页面中的区域为//div[@class='nui-table']，如果传入参数初始化，就以传入的参数作于列表所在的区域。这样就可以处理列表中的元素和页面中的元素存在相同的相对路径或者是页面中有多个列表控件的情况。

以后我们对某个具体列表中的元素处理都会在这个区域下进行操作，如根据行列号取得某个单元格元素(findElement为根据xpah获取元素的方法)：

**通过行列号获取单元格**

```java
        /**
         * find a cell element with row index and colum index
         * @param row
         * @param column
         * @return cell element
         */
        WebElement getCell(int rowIndex,int columnIndex){
            return findElement("$containerSelector/$TABLE_TAGNAME/$TABLE_BODY_TAGNAME/$TR_TAGNAME[$rowIndex]/$TD_TAGNAME[$columnIndex]")
        }
        ```

在以上方法中，table、tr、td的标签是写死的，不过这些可以作为属性定义出来，同样可以支持div之类的标签组成的表格。

我们在页面类里加入该控件后，外部调用的时候就可以直接使用如下方法来取得一个单元格（异常处理大家自己解决就是啦）：
```java
    ContactsPage contactsPage=new ContactsPage()
    contactsPage.grid.getCell(row,column)
    ```
这个控件现在就可以放到任意的页面中使用，调用方法完全一致。

另外，更常使用的是根据列头及单元格显示文本来获取单元格，同样很简单，这次我们需要多一个用到获取所有列头文本的方法，这两个方法如下：

**通过显示文本获取单元格**
```java
/**
 * find a cell element with column head text and cell text
 * @param headText
 * @param cellText
 * @return cell element
 */
WebElement getCell(String headText,String cellText){
    WebElement cell=null
    List<String> headTextList=getHeadTextList()
    assertThat("table has head [$headText]",headTextList,hasItem(headText))
    int headIndex=headTextList.indexOf(headText)+1
    List<String> columnTexts=getTexts("$containerSelector/$TABLE_TAGNAME/$TABLE_BODY_TAGNAME/$TR_TAGNAME/$TD_TAGNAME[$headIndex]")
    int trIndex=columnTexts.indexOf(cellText)+1
    if (trIndex>0)
        cell= findElement("$containerSelector/$TABLE_TAGNAME/$TABLE_BODY_TAGNAME/$TR_TAGNAME[$trIndex]/$TD_TAGNAME[$headIndex]")
    return  cell
}

/**
 * get all head text
 * @return
 */
List<String> getHeadTextList(){
    return getTexts("$containerSelector/$TABLE_TAGNAME/$TABLE_HEAD_TAGNAME/$TR_TAGNAME/$TD_TAGNAME")
}
        ```

**特殊的情况**

当然，实际情况中可能遇到这样的问题，在有的页面中控件在一个frame当中，这种时候我们可以使用getter方法来初始化控件，如：
```java
final static String FRAME="frame"

/**
 * grid component
 */
Grid grid

/**
 * init component with getter
 * @return
 */
Grid getGrid(){
    switchFrame(FRAME)
    return new Grid()
}
        ```
组件设计大概就是这些内容，对组件的处理无非就是拼接字符串。因此使用xpath，css之类的方法会比单纯的id、name更为实用：他们能让多次的元素查找变为一次，大大的提高效率。

下一章为本章补充：[WEB自动化测试中的页面模型（三点五）-树组件设计](http://www.assilzm.com/?p=138)

接下来可能会讲一些复杂情况的页面设计，找到合适的例子我会来更新。
