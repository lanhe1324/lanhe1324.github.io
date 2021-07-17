## 描述
> [!Tip|label:标准功能的提供类]
   - 泛型基类,继承自`BaseService`重写部分方法和属性;    
   - T类型参数:对应数据实体`Entity`;
   - V类型参数:对应数据视图`View`;
   - 实现了`IServiceWin`和`IServiceWeb`接口，支持UI基类对数据操作;
   - 使用`BindingSource`控件实现对数据绑定,提供对winform程序进行数据双向绑定的支持;
   - 实现了主从表数据更新;
 
```cs
public abstract class BaseServiceProvider<T, V> : BaseService<T, V>, IServiceWin,IServiceWeb,  IDisposable where T : class, new() where V : class, new()
{
    public BaseServiceProvider()
    {
        this.ChildServices = new List<AbsService>();

    }
}
```

## IServiceWin接口
- winform程序使用
```cs
public interface IServiceWin 
{   
    XueQ.Model.View.SysUserView User { get; set; }      
    event ListChangedHandel OnListChanged;       
    BindingSource BindSource { get; }
    BindingSource BindSourceView { get; }
    DbModelConfig ModelConfig { get; }
    List<XueQ.Model.View.SysEntityFieldConfigView> EntityFieldConfig { get;}       
    bool IsViewOwn { get; set; }
    object Current { get; }        
    bool HasChanges { get; }
    bool IsMaster { get; }      
    object GetPrimaryKeyVal(object entitys);
    void AddNew();       
    void AddRow();
    void InsertRow();
    void DelRow();      
    ExecuteResult Update(MethodType methodName);       
    void Cancel();
    void Refresh();      
    void BindViewData(Stone.AQH.QueryCondition condition);
    void BindViewPagedData(ref long total, int startIndex, int count, Stone.AQH.QuerySort sort, Stone.AQH.QueryCondition condition = null);
    void BindEditData(Stone.AQH.QueryCondition condition = null);
    void BindEditData(object[] entitys);
    System.Threading.Tasks.Task<WorkFlow.TaskFlowFormat> GetWorkTaskInfo();//取审批流信息
   
}
```

## IServiceWeb接口

- Mvc的BaseController使用

```cs
public interface IServiceWeb
{       
    XueQ.Model.View.SysUserView User { get; set; }      
    List<XueQ.Model.View.SysEntityFieldConfigView> EntityFieldConfig { get; }
    ExecuteResult GetAddNewData();
    ExecuteResult AddNew(string value);   
    ExecuteResult Delete(params object[] values);
    ExecuteResult Edit(string value);   
    ExecuteResult SelectEditViewData(object value);
    object SelectViewData(QueryCondition condition, QuerySort sort);
    object SelectPagedViewData(ref long total, int startIndex, int count, QueryCondition condition, QuerySort sort);
    List<ComboBoxData> SelectComboBox(string value, string text);
}
```

## 重写DbContext
> [!Tip|label:保证主从表使用同一个DbContext]
> 为什么要重写呢？  
> 为了保证数据的一致性，主从表更新数据时必须处于同一个`DbContext`作用域内，也就是处于同一个事务里面，`DbContext`作用域内本身就是事务提交，只要出现异常数据就会自动回滚
```cs
public override XueQDbContext Db
{
    get
    {
        if (ParentService != null)
        {
            this.Db = ParentService.Db;
            return ParentService.Db;
        }
        return base.Db;
    }
    set { _db = value; }
}
```