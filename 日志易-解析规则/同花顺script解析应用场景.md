## 问题表象

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207155625.png)

从上图可以看到，不以功能号group by成功率很高，但是根据功能号group by 后，出现了一大部分响应率为0的情况

- 不以功能号分组SPL

  ```sql
  index=thsjy appname:thsjy thsjy.ANSTYPE:* NOT  (thsjy.ANSTYPE:1 OR thsjy.ANSTYPE:0)
  | stats count() as total
  | eval t="A"
  | join  type=left t [[
  	index=thsjy appname:thsjy (thsjy.ANSTYPE:1 OR thsjy.ANSTYPE:0)
  	| stats count() as succ
  	| eval t="A"
  ]]
  |  eval health=if(succ/total>1,100,succ/total*100)
  |eval health=todouble(health)
  |eval health=format("%.2f%%",health)
  ```

- 以功能号分组SPL

  ```sql
  index=thsjy appname:thsjy AND thsjy.ANSTYPE:* NOT (thsjy.ANSTYPE:0 OR thsjy.ANSTYPE:1)
  |stats count() as total by thsjy.type,thsjy.funccn
  | join  type=left thsjy.type [[
  	index=thsjy appname:thsjy (thsjy.ANSTYPE:1 OR thsjy.ANSTYPE:0)
  	| stats count() as succ by thsjy.type
  ]]
  |eval succ=if(empty(succ),0,succ)
  | eval health=if(succ/total*100>100,100,succ/total*100)
  |sort by +health
  |eval health=format("%.2f%%",todouble(health))
  | fields thsjy.type,health,thsjy.funccn
  | rename thsjy.funccn as "功能名称",health as "响应率",thsjy.type as "功能号"
  ```

## 问题根因

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207155442.png)

从上图可以看到，在请求日志中，用户退出这一类型功能号，其`thsjy.type`就有三种值，而应答日志中只有一种

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207155551.png)

由于算响应率要以`thsjy.type`和`thsjy.funcc` join，基于上面出现的情况，就会导致有其他俩相同请求功能号的响应数量均为0，但实际上总和是相等的，也就是说正常情况下，`用户退出`功能号响应率应为`100%`

------

`thsjy.type`值的由来是通过字典表映射的，如下：

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207161002.png)

可以看到，一部分功能号是单个值，一部分是 `x-x`的性能，而有问题的部分就是这个单值，同时字段提取中的`thsjy.type`值又是通过同花顺日志中的`REQTYPE-MMLB`合并而来的

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207161202.png)

## 解决方案

由于只有单值存在该问题，所以我们只要取单值中的功能号进行匹配即可，不需要再去匹配`MMLB`，但同时也要兼容`REQTYPE-MMLB`合并值-->`type`，所以最终的方案就是：

1. 将`REQTYPE-MMLB`合并后的值再做一次拆分，取`-`左边的值即可
2. 判断取出来的值是否在单值功能号列表中
3. 在，则将`type`值替换，不在则沿用原来的值

------

用到的功能就是script解析规则，配置如下:

```json
{
    "__codec_type":"script",
    "script":"source[\"types\"]=split(source[\"type\"],\"-\");source[\"a\"]=source[\"types\"][0];source[\"func\"]=[\"1\",\"2\",\"V\",\"S\",\"4\",\"5\",\"B\"];for (k:source[\"func\"]) if(source[\"a\"]==k) source[\"type\"]=source[\"a\"]"
}
```

------

逻辑如下：

1. 将`type`值切割，取出下标为0的值
2. 新建一个list，存单值的功能号
3. for循环list
4. 判断`1`中切割出来的值是否在list中，在，则`type`值覆写，不在则采用原值

------

效果如下:

![](https://cdn.jsdelivr.net/gh/AlphaBrock/md_img/macos/20201207163124.png)