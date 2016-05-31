<!-- 
.. title: thymeleaf的坑一记
.. slug: thymeleafde-keng-yi-ji
.. date: 2016-05-31 08:37:04 UTC+08:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

记录下thymeleaf 3.0新增的`th:field`与`th:each`中生成的变量结合使用时的坑。

代码就是官方例子[https://github.com/thymeleaf/thymeleafexamples-stsm.git](https://github.com/thymeleaf/thymeleafexamples-stsm.git)，在`webapp/WEB-INF/seedstartermng.html`中118-131行

```html
<tr th:each="row,rowStat : *{rows}">
  <td th:text="${rowStat.count}">1</td>
  <td>
    <select th:field="*{rows[__${rowStat.index}__].variety}">
      <option th:each="var : ${allVarieties}" th:value="${var.id}" th:text="${var.name}">Thymus Thymi</option>
    </select>
  </td>
  <td>
    <input type="text" th:field="*{rows[__${rowStat.index}__].seedsPerCell}" th:errorclass="fieldError" />
  </td>
  <td>
    <button type="submit" name="removeRow" th:value="${rowStat.index}" th:text="#{seedstarter.row.remove}">Remove row</button>
  </td>
</tr>
```

其中第二个td取出rows中的每一行的variety，做成一个select下拉菜单，当我尝试用`${row.variety}`替换原文的`*{rows[__${rowStat.index}__].variety}`时，页面渲染抛出异常`Neither BindingResult nor plain target object for bean name 'row' available as request attribute`。多方探索之后得出结论：这是我们使用的问题，设计时特意在遇到这种情况时为了避免服务端无法处理提交后的表单，就抛出异常。

用例子中的代码渲染完生成后的页面为：

```html
<tr>
  <td>1</td>
  <td>
    <select id="rows0.variety" name="rows[0].variety">
      <option value="1">Thymus vulgaris</option>
      <option value="2">Thymus x citriodorus</option>
      <option value="3">Thymus herba-barona</option>
      <option value="4">Thymus pseudolaginosus</option>
      <option value="5">Thymus serpyllum</option>
    </select>
  </td>
  <td>
    <input type="text" id="rows0.seedsPerCell" name="rows[0].seedsPerCell" value="">
  </td>
  <td>
    <button type="submit" name="removeRow" value="0">Remove row</button>
  </td>
</tr>

```

其中，`th:field`在渲染时会给目标元素加上id和name属性，name属性取`*{...}`中表达式的值。由于`__${rowStat.index}__`是个预处理语句，在渲染之前会直接文本替换，所以最终的表达式为`rows[0].variety`。

但是如果是`${row.variety}`，则最终name会被替换为`row.variety`，由于浏览器作为客户端无法知道服务端的逻辑，这样之后的每个tr中的元素都是这个表达式，表单提交的时候无法被spring解析成一个list对象，就会出现错误。

官方站在实现角度的解释可以看这个[issue](https://github.com/thymeleaf/thymeleaf/issues/505#issuecomment-222557959)