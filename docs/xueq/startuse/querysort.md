## 基本功能
- 排序容器，可以动态生成lambda表达式;
- 支持升序和降序;

```cs
public class QuerySort : IEnumerable
{
    public QuerySort();
    public QuerySort Add(string name, SortEnum sort);
    public QuerySort Add(string name, string sort); 
    public bool HasData();
}
```

## Add方法
- 支持链式编写
> 参数1：字段名, 参数2：排序类型
```cs
var sort = new QuerySort();
sort.Add("Id", "Aesc");
sort.Add("PositionId", "ThenDesc");
var result=this.Db.SysUser.OrderBy(sort);
```
通过add方法添加的排序等效于以下lambda表达式
```cs
var result = this.Db.SysUser.OrderBy(n => n.Id).ThenByDescending(n => n.PositionId);
```