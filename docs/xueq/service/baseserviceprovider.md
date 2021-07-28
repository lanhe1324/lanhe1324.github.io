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

## 定义ListChange事件
> [!TIP]
> 更新数据时，通知UI窗体
```cs
public delegate void ListChangedHandel(object sender, ServiceArgs e);

private ServiceArgs ServiceArgs;

public event ListChangedHandel OnListChanged;

private NotifyTypeEnum _notifyType { get; set; }

protected NotifyTypeEnum NotifyType
{
    get { return _notifyType; }
    set
    {
        _notifyType = value;
        if (ServiceArgs == null) ServiceArgs = new ServiceArgs();
        //IsAddNew默认值为false, 在新增时设置为true,初始化和保存后设置为false
        switch (_notifyType)
        {
            case NotifyTypeEnum.None:
                break;
            case NotifyTypeEnum.AddNew:
                ServiceArgs.IsAddNew = true;
                break;
            case NotifyTypeEnum.AddRow:
                break;
            case NotifyTypeEnum.DelRow:
                break;
            case NotifyTypeEnum.Modify:
                break;
            case NotifyTypeEnum.AfterSaveUpdate:
            case NotifyTypeEnum.AfterDeleteUpdate:
            case NotifyTypeEnum.Initialize:
                ServiceArgs.IsAddNew = false;
                break;
            default:
                break;
        }
        this.ServiceArgs.Count = this.Count;
        this.ServiceArgs.HasData = this.Count > 0;
        this.ServiceArgs.IsMaster = this.IsMaster;
        this.ServiceArgs.IsChild = this.IsChild;
        this.ServiceArgs.NotifyType = this._notifyType;
        this.ServiceArgs.ParentHasData = this.IsChild ? this.ParentService.Count > 0 : false;
        if (OnListChanged != null) OnListChanged(this, this.ServiceArgs);
    }
}

```


## 查询时绑定数据

```cs
private BindingSource _bindSourceView;
public virtual BindingSource BindSourceView
{
    get { return _bindSourceView ?? (_bindSourceView = new BindingSource()); }
}      
public BindingList<V> BindListView { get; set; }

public virtual void BindViewData(Stone.AQH.QueryCondition condition)
{
    try
    {
        var sort = base.GetDefualtQuerySort;
        var queryable = QWhereConditionJoin().Where(condition).OrderBy(sort);
        var listdata = queryable.Select<T, V>().ToList();
        this.BindListView = new BindingList<V>(listdata);
        this.BindSourceView.DataSource = BindListView;       
    }
    catch (Exception ex)
    {
        this.BindSourceView.DataSource = new List<V>();
    }

}

public virtual void BindViewPagedData(ref long total, int startIndex, int count, Stone.AQH.QuerySort sort, Stone.AQH.QueryCondition condition = null)
{
    try
    {
        if (!sort.HasData()) sort = base.GetDefualtQuerySort;
        var queryable = QWhereConditionJoin().Where(condition);

        var viewquery = queryable.Select<T, V>().DeferredCount().FutureValue();
        var result = queryable.OrderBy(sort).Page(startIndex, count).Select<T, V>().Future();
        total = viewquery.Value;
        var listdata = result.ToList().ReplaceUserId(this);
        this.BindListView = new BindingList<V>(listdata);
        this.BindSourceView.DataSource = BindListView;       
    }
    catch (Exception ex)
    {
        this.BindSourceView.DataSource = new List<V>();
    }

}
```


## 编辑时绑定数据

```cs
private BindingSource _bindSource;
public virtual BindingSource BindSource
{
    get { return _bindSource ?? (_bindSource = new BindingSource()); }
}

public BindingList<T> BindList { get; protected set; }

public override void BindEditData(object[] entitys)
{
    Dispose();
    var keyvals = base.GetPrimaryKeyVal(entitys);
    this.Db.Set<T>().Contains(ModelConfig.KeyName, keyvals).Load();
    InitBindData();
}
public override void BindEditData(Stone.AQH.QueryCondition condition = null)
{
    Dispose();
    if (condition != null && condition.HasData) this.Db.Set<T>().Where(condition).Load();
    InitBindData();
}
/// <summary>
/// 初始化绑定数据
/// </summary>
private void InitBindData()
{
    this.BindList = this.Db.Set<T>().Local.ToBindingList();
    this.BindList.ListChanged += BindList_ListChanged;
    this.BindSource.DataSource = this.BindList;

    if (this.ChildServices.Count > 0)
    {
        foreach (var item in this.ChildServices)
        {
            QueryCondition childCondition = null;
            if (this.CurrentEntity != null)
            {                     
                var keyval = base.ModelConfig.GetKeyVal(this.CurrentEntity);
                var foreignKeyName = item.ModelConfig.GetForeignKeyName(this.EType.Name);
                childCondition = new QueryCondition().Add(foreignKeyName, keyval.ToString(), "Equal");
            }
            item.BindEditData(childCondition);
        }

    }
    this.NotifyType = NotifyTypeEnum.Initialize;
}

public void Dispose()
{
    if (ParentService != null) return;
    Db.Dispose();
    Db = null;
}
private void BindList_ListChanged(object sender, ListChangedEventArgs e)
{
    switch (e.ListChangedType)
    {
        case ListChangedType.Reset:
            break;
        case ListChangedType.ItemAdded:
            this.NotifyType = NotifyTypeEnum.AddRow;
            break;
        case ListChangedType.ItemDeleted:
            this.NotifyType = NotifyTypeEnum.DelRow;
            break;
        case ListChangedType.ItemMoved:
            break;
        case ListChangedType.ItemChanged:
            if (RemoveFlag) break;
            var list = sender as BindingList<T>;
            OnPropertyChanged(e.PropertyDescriptor.Name, list[e.NewIndex], e.PropertyDescriptor);
            break;
        case ListChangedType.PropertyDescriptorAdded:
            break;
        case ListChangedType.PropertyDescriptorDeleted:
            break;
        case ListChangedType.PropertyDescriptorChanged:
            break;
        default:
            break;
    }

}
protected virtual void OnPropertyChanged(string name, T t, PropertyDescriptor pdesc)
{

}
        
```