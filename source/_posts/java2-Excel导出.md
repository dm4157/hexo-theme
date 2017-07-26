id: java2
title: Excel导出
categories: core java
tags: [excel,magic,java]
---

## 故事
> 导报表，会宕机，勿动！

这是发生在我系统中的故（shi）事（gu）。报表导出是很稀松平常的功能，随便搞搞就好。然而当导出量增大时，问题可能就棘手了。
我系统中常有20W左右的报表导出，导出时CPU一路高歌、内存余额紧缺，最终结果往往是这样的：
- CPU过高导致memcached客户端连接超时，系统**宕机**；
- 内存余额清零`OOM`，系统**宕机**；
- 系统心情好，顺产~；

最近第一条故障频发，心力交瘁，疲于应付。痛定思痛，是时候梳理下报表导出了。

## 硬币的两面
> 产品与技术是一个硬币的两面。  -- 邓爷爷

### 产品
报表导出产品设计通常简单随意，不妨重新思考下极限与体验：
- 产品上对导出极限值进行限制，避免系统**宕机**；比如时间范围限制、最终结果限制；
- 在导出时进行二次确认，保证用户执行了正确的报表导出动作，避免无意义的导出操作；
- 精心筛选导出字段，避免不必要的字段增加导出负担；

### 技术
报表导出开发技术通常简单粗暴， 不妨重新思考下效率与健壮：
- 内在导出上限设置，超出报错；
- 性能更优的代码，或者更适合业务的代码；比如有的方法适合小数据量导出，有的则适合大量导出；
- 解决问题的思路是否可以变换一下，比如需要导出的数据分页查出，逐批写入；

---
## 技术篇

### 基础
> Excel导出常见工具包为POI和JXL,功能上POI更多些，然而我也用不上，其实我是看牌子的，POI是Apache家的。

POI采用了较好的面向对象的思想，Excel表格中的一些概念都变成了类，
比如工作簿(`WorkBook`)、工作表(`Sheet`)、行(`Row`)、单元格(`Cell`)。
创建一个Excel文件的基本步骤是这样的：

```java
// 创建工作簿 2003版的, .xls
Workbook wb = new HSSFWorkbook();
// 2007以后, .xlsx
// Workbook wb = new XSSFWorkbook();
CreationHelper createHelper = wb.getCreationHelper();

// 用工作簿创建工作表
Sheet sheet = wb.createSheet("new sheet");

// 用工作表创建行, 参数是行号, 从0开始
Row row = sheet.createRow((short) 0);

// 用行创建单元格, 参数是列号, 从0开始
Cell cell = row.createCell(0);
cell.setCellValue(1);

row.createCell(1).setCellValue(1.2);
row.createCell(2).setCellValue(
        createHelper.createRichTextString("This is a string"));
row.createCell(3).setCellValue(true);

FileOutputStream fileOut = new FileOutputStream("workbook.xls");
// 调用工作簿write方法写入流
wb.write(fileOut);
fileOut.close();
```
导入导出不同之处就是一个读一个写（废话）， 简单用用还是不难
> PS. Excel2003版最大行数是65536行，Excel2007开始的版本最大行数是1048576行

### “大数据”
当数据量激增，达到几万几十万的时候，导出似乎就没那么简单了。如何应对“大数据”对内存的冲击？使用`SXSSFWorkbook`即可

```java
// 内存中只驻留100行数据
SXSSFWorkbook workbook = new SXSSFWorkbook(100);
```

据说`SXSSFWorkbook`采用流式写法，可以将多余的数据写到磁盘临时文件中去，从而保证内存的安全。
但有了好处自然麻烦（平衡之道），使用结束时记得**销毁**临时文件。

```java
// 内存中只驻留100行数据
SXSSFWorkbook workbook = new SXSSFWorkbook(100);
...
workbook.dispose();
```

至于缓存100行还是1000行，这需要自己去尝试，我也是在路上。

### 新玩法
> 每次导出Excel，都需要去编写特定的导出逻辑么？

> 担心*大数据量*导出时的性能损耗么？

> 现推出新式玩法，简单高效，已加入肯德基豪华午餐；

### 核心方法

```java
/**
 * 针对只参与一张excel导出的实体,可使用此方法
 * 即实体上只有一个@Excel注解
 * @param response
 * @param data          将要导出的数据
 * @param modelClass    数据的实体类型信息, 需要使用@Excel
 * @param <T>           实体泛型
 */
public <T> void exportToExcel(HttpServletResponse response, List<T> data, Class<T> modelClass)
```

- response来自Controller, 导出怎么能少了他；
- data即是需要导出的数据列表，自己写查询语句；
- modelClass是数据列表中放的核心实体，即要导出的对象

### 实体要求
> 平衡之道告诉我们，想得到先付出。

作为被导出的实体，需要提前注册下面两个注解。
#### @Excel
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Excel {

    /** 导出的文件叫什么? */
    String[] value() default "没有名字的导出表";

    /** 最大导出条数 */
    int limit() default 1040000;
}
```

#### @Column

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Columns.class)
public @interface Column{

    /** excel列名,就是表头 */
    String value() default "";

    /** 列表排序, 小号在前面, 默认的话就按实体里字段顺序了 */
    int index() default 0;

    /** 属于哪个@Excel, 填@Excel.value */
    String[] belong() default "";
}
```

#### 实体例子

```java
// 导出文件名为《账号管理_费用记录.xlsx》,
// 最大条数限制为90w，超出后报ExcelException
@Excel(value = "账号管理_费用记录", limit = 900000)
public class VExpenseDetail {

    @Column("网站ID")
    private String webAccountId;

    // 导出后将排在`网站ID`之前
    @Column(value = "网站", index = -1)
    private String webName;

    // 导出后列名为字段名即`actionName`
    @Column
    private String actionName;
...
}
```

### 用起来，少年郎

```java
@Autowired
private ExcelHandler excelHandler;
...
List<VExpenseDetail> exportData = actionService.queryFeeRecord(paraMap);
excelHandler.exportToExcel(response, exportData, VExpenseDetail.class);
```

### 更多玩法
通常，我们不会为导出某张报表创造一个新类，如果一个业务实体面临多张报表导出，该如何？

```java
// 实体支持两张表导出
@Excel(value = {"账号管理_费用记录", "操作记录"}, limit = 900000)
public class VExpenseDetail {

    // 默认会在两张表里都出现
    @Column("网站ID")
    private String webAccountId;

    // 与上面的默认没什么区别， 两张表都有此字段
    @Column(value = "网站", belong = {"账号管理_费用记录", "操作记录"})
    private String webName;

    // 两张表都有此字段，但有不同的列名
    // ** 不建议此做法，同样的字段应该具有同样的含义，不论身处何地 **
    @Column(value = "费用类型", belong = "账号管理_费用记录")
    @Column(value = "操作类型", belong = "操作记录")
    private String actionName;
    ...
}
```

## 性能
> 比较方能出优劣

这里将上述导出方法与广告系统中的原有导出方法做比较，在同样的条件下看看CPU和内存方面的表现。
```
机器配置
CPU i5
内存 8G

导出表格列数 16
```
### 不同数量导出的新旧对比
> old-392-10 => 旧版-导出392条-10次尝试

#### 392条导出10次
**old-392-10**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-392-10.png)
**new-392-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-392-10.png)
#### 11812条导出10次
**old-11812-10**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-11812-10.png)
**new-11812-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-11812-10.png)
#### 96437条导出10次
**old-96437-10**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-96437-13.png)
**new-96437-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-96437-10.png)
#### 271658条导出10次
**old-271658-10**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-271658-10.png)
**new-271658-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-271658-10.png)
#### 448750条导出10次
**old-448750-1-崩溃**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-448750-1-%E5%B4%A9%E6%BA%83.png)
**new-448750-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-448750-10.png)
#### 601248条导出10次
**old-601248-1-崩溃**
![旧版](http://7xrbi6.com1.z0.glb.clouddn.com/img-old-601248-1-%E5%B4%A9%E6%BA%83.png)
**new-601248-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-601248-10.png)
#### 1012896条导出10次
**new-1012896-10**
![新版](http://7xrbi6.com1.z0.glb.clouddn.com/img-new-1012896-10.png)

### 看图说话
在数据量少（1w以内）， 新旧没有明显的性能差异，旧版在cpu上更优而新版在内存上略胜。
但从9w数据量开始，新版的优势就非常明显了，并且随着数据量的增加急剧扩大。
老版在44w处崩溃，而新版可以一直导出到接近单页极限值101w（没有继续试验）。
结合新版的易用性，我还是喜欢新版（自己的孩子）。

## 缺陷与展望
- 只支持单sheet导出，即最大值为1048576条。
- 不支持设置表格式，比如列宽、字体、颜色等。
- 可按照同样的逻辑去编写Excel导入方法。
- Excel导出时可尝试分页读取数据。**尽请期待**
