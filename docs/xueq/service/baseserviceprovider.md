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


# IServiceWin实现方法

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
> [!TIP]
> 
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
> [!TIP]
> 绑定前必须调用Dispose方法

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
            if (DisablePropertyChange) break;
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

##  数据更新方法
- 通过BindingSource方法进行新增、删除、更新


```cs
public override void AddNew()
{
    BindEditData();
    if (this.IsMaster) this.AddRow();

    if (this.ChildServices.Count != 0)
    {
        foreach (var item in this.ChildServices)
        {
            item.AddNew();
        }
    }

    this.NotifyType = NotifyTypeEnum.AddNew;
}
public virtual void AddRow()
{
    this.BindSource.AddNew();
    this.BindSource.EndEdit();
    AfterAddNew(this.CurrentEntity);
}
public virtual void InsertRow()
{
    BindSource.Insert(BindSource.Position, new T());
    this.BindSource.EndEdit();
    this.BindSource.Position--;
    AfterAddNew(this.CurrentEntity);
}
protected virtual void AfterAddNew(T t)
{
    if (this.ParentService != null && this.ParentService.Current != null)
    {
        this.Db.Extensions().UpdateEntity<T>(t, this.ParentService.Current);
    }
    FillDataPoint(t);
    FillPrimaryKeyByAddNew(t);
}
public virtual void DelRow()
{
    DisablePropertyChange = true;
    this.BindSource.Remove(this.CurrentEntity);
    DisablePropertyChange = false;
}

public ExecuteResult Update(MethodType methodName)
{
    var result = XueQ.Utility.Result.OK("执行成功"); 
    this.BindSource.EndEdit();
    if (this.Count == 0 && this.IsMaster) return result;
    try
    {
        SvcBeforeUpdate(methodName, result);
        if (result.ExecuteValue) return result;
        var modifydata = this.Db.Extensions().TrackerValues;
        Db.SaveChanges();
        base.SaveUploadFile();//保存附件信息
        base.SubmitWorkFlowTask();
        base.WriteModifyRecord(methodName, modifydata);
        SvcAfterUpdate(methodName, result);
    }
    catch (DbUpdateConcurrencyException ex)
    {
        
        // ex.Entries.Single().Reload();
        result.Msg = "数据库中不存在此纪录/r/n"+ex.Message;
        result.State = ExecuteState.warning;
    }
    catch(DbUpdateException ex)
    {
        var e = XueQ.Utility.ExceptionHelper.Extract(ex);
        result.Msg = e.Message;
        if (e is SqlException)
        {
            var subex = e as SqlException;
            switch (subex.ErrorCode)
            {
                case -2146232060:
                    result.Msg = "主键重复/r/n" + e.Message;
                    break;
                default:
                    result.Msg = e.Message;
                    break;

            }
        }
        
        result.State = ExecuteState.warning;
    }
    catch (Exception ex)
    {
        result.Msg = $"未知错误,{ex.Message}";
        result.State = ExecuteState.warning;
    }
    return result;
}
public override void SvcBeforeUpdate(MethodType methodName, ExecuteResult result)
{
    this.NotifyType = methodName == MethodType.Save ? NotifyTypeEnum.BeforeSaveUpdate : NotifyTypeEnum.BeforeDeleteUpdate;
    this.BindSource.EndEdit();
    BeforeUpdate(methodName, this.BindList, result);
    if (result.ExecuteValue) return;

    if (this.ChildServices.Count > 0)
    {
        foreach (var item in this.ChildServices)
        {
            item.SvcBeforeUpdate(methodName, result);
            if (result.ExecuteValue) return;
        }
    }
    switch (methodName)
    {
        case MethodType.Save:
            this.FillPrimaryKeyBySave();
            break;
        case MethodType.Delete:
            this.Clear();
            break;
    }
}
public override void SvcAfterUpdate(MethodType methodName, ExecuteResult result)
{

    if (this.ChildServices.Count > 0)
    {
        foreach (var item in this.ChildServices)
        {
            item.SvcAfterUpdate(methodName, result);
            if (result.ExecuteValue) return;
        }
    }

    this.NotifyType = methodName == MethodType.Save ? NotifyTypeEnum.AfterSaveUpdate : NotifyTypeEnum.AfterDeleteUpdate;
    AfterUpdate(methodName, this.BindList, result);
}        
protected virtual void BeforeUpdate(MethodType method, BindingList<T> bindlist, ExecuteResult eResult)
{
    Validate(method, bindlist, eResult,this.ServiceArgs);
}
protected virtual void AfterUpdate(MethodType method, BindingList<T> bindlist, ExecuteResult eResult)
{

}
public virtual void Clear()
{
    DisablePropertyChange = true;
    this.BindList.Clear();
    DisablePropertyChange = false;
    this.NotifyType = NotifyTypeEnum.Clear;
}
public virtual void Cancel()
{
    BindEditData();
    this.NotifyType = NotifyTypeEnum.Cancel;
}
/// <summary>
/// 从数据库刷新本地数据
/// </summary>
public virtual void Refresh()
{
    var context = ((IObjectContextAdapter)Db).ObjectContext;
    var refreshableObjects = this.Db.ChangeTracker.Entries().Where(e => e.State != EntityState.Added).Select(e => e.Entity).ToList();
    context.Refresh(System.Data.Entity.Core.Objects.RefreshMode.StoreWins, refreshableObjects);
    LocalReload();
    if (this.ChildServices.Count > 0)
    {
        foreach (var item in this.ChildServices)
        {
            item.LocalReload();
        }
    }
    this.NotifyType = NotifyTypeEnum.Initialize;
}
/// <summary>
/// 从本地缓存中重新加载数据
/// </summary>
public override void LocalReload()
{
    var local = this.Db.Set<T>().Local.ToList();
    foreach (var item in this.BindList.ToList())
    {
        if (this.DBState(item).State == EntityState.Added)
        {
            this.Db.Set<T>().Local.Remove(item);
        }
    }
    var refreshableObjects = this.Db.ChangeTracker.Entries<T>().Where(e => e.State != EntityState.Added).Select(e => e.Entity).ToList();
    foreach (var item in refreshableObjects)
    {
        this.Db.Set<T>().Local.Add(item);
    }
}

```



## 填充主键和DataPoint方法
> [!TIP]
> 填充主键有两种情况:1.新增时填充，2.保存时填充 
```cs
/// <summary>
/// 新增时填充主键
/// </summary>
/// <param name="type"></param>
public virtual void FillPrimaryKeyByAddNew(T t)
{
    if (base.IsFillPrimaryKeyOfAddNew)
    {
        Common.SetKeyVal<T>(t, base.EntityFieldAttInfo.FirstOrDefault(n => n.IsAutoKey));
    }
}
/// <summary>
/// 保存时填充主键
/// </summary>
/// <param name="type"></param>
public virtual void FillPrimaryKeyBySave()
{
    if (base.IsFillPrimaryKeyOfAddNew) return;
    var addlist = this.BindList.Where(n => this.DBState(n).State.Equals(EntityState.Added)).ToList();
    if (addlist.Count > 0) Common.SetKeyVal<T>(addlist, base.EntityFieldAttInfo.FirstOrDefault(n => n.IsAutoKey));
}

/// <summary>
/// 填充数据类型
/// </summary>
/// <param name="t"></param>
protected virtual void FillDataPoint(T t)
{
    Dictionary<string, object> dic = new Dictionary<string, object>();
    dic.Add("CreateBy", base.User?.Id);
    dic.Add("CreateDate", DateTime.Now);
    dic.Add("fldCreateBy", base.User?.Id);
    dic.Add("fldCreateDate", DateTime.Now);
    var point = base.EntityFieldAttInfo.FirstOrDefault(n => n.IsDataPoint);
    if (point != null)
    {
        dic.Add(point.Name, point.DataPointFirstValues);
    }
    t.Extensions<T>().SetValue(dic);
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


# IServiceWeb实现方法

## Web新增方法
> [!TIP]
> 前端需要提交Json格式的数据，可以同时保存一对多关系的数据
> 
```cs
public virtual ExecuteResult GetAddNewData()
{
    var t = new T();
    AfterAddNewData(t);
    return XueQ.Utility.Result.OK(t.CreateMap<T,V>().Map()); 
}
protected virtual void AfterAddNewData(T t)
{
    FillDataPoint(t);
}
/// <summary>
/// Json数据类型增加
/// </summary>
/// <param name="value"></param>
/// <returns></returns>
public virtual ExecuteResult AddNew(string value)
{
    GetUploadAndWorkFlow(value);      
    
    this.WebBindData();

    var child = this.ChildServices?.Select(n => n.EType).ToArray();
    var t = this.Db.Extensions().AddNewEntity<T>(value, child);      
    
    FillDataPoint(t);

    this.NotifyType = NotifyTypeEnum.AddNew;
    return Update(Model.XEnum.MethodType.Save);
}   

```

## web编辑方法

```cs
/// <summary>
/// Json数据修改
/// </summary>
/// <param name="value"></param>
/// <returns></returns>
public virtual ExecuteResult Edit(string value)
{
    GetUploadAndWorkFlow(value);
    var keyVal = XueQ.Utility.JsonFormat.DeserializeObjectByKey(value, ModelConfig.KeyName.ToCamelCase());           
 
    if (keyVal.IsNullOrEmpty()) return XueQ.Utility.Result.Warning("主键不能为空");

    var condition = new QueryCondition().Add(ModelConfig.KeyName, keyVal);       
    this.WebBindData(condition, keyVal);
    if (CurrentEntity == null) return XueQ.Utility.Result.Warning("当前记录不存在");

    var child = this.ChildServices?.Select(n => n.EType).ToArray();
    this.Db.Extensions().UpdateEntity<T>(value, child);

    return Update(Model.XEnum.MethodType.Save);
} 

```


## web删除方法

```cs
public virtual ExecuteResult Delete(params object[] keys)
{
    QueryCondition condition = new QueryCondition();
    condition.AddInQuery(ModelConfig.KeyName, keys);
    BindEditData(condition);
    return Update(Model.XEnum.MethodType.Delete);
}
```


## web绑定数据

```cs
protected virtual void WebBindData(QueryCondition condition = null, object keyval = null)
{
    this.Dispose();              
    if (condition != null && condition.HasData) this.Db.Set<T>().Where(condition).Load();
    this.BindList = this.Db.Set<T>().Local.ToBindingList();
    this.BindSource.DataSource = this.BindList;
    if (this.HasChild)
    {
        QueryCondition childCondition = null;
        foreach (var item in this.ChildServices)
        {
            if (keyval != null)
            {
                var foreignKeyName = item.ModelConfig.GetForeignKeyName(this.EType.Name);
                childCondition = new QueryCondition().Add(foreignKeyName, keyval, "Equal");
            }
            item.BindEditData(childCondition);
        }
    }
}
```


## web查询数据方法

```cs
/// <summary>
/// 查询编辑数据
/// </summary>
/// <param name="id"></param>
/// <returns></returns>
public virtual ExecuteResult SelectEditViewData(object id)
{
    var result =base.QWhereConditionJoin().Where(ModelConfig.KeyName, id.ToString());
    object eval = result.Select<T, V>().FirstOrDefault().ReplaceUserId(this);            
    if (this.HasChild  && eval != null)
    {
        foreach (var item in this.ChildServices)
        {
            var foreignKeyName = item.ModelConfig.GetForeignKeyName(this.EType.Name);
            var childCondition = new QueryCondition().Add(foreignKeyName, id.ToString(), "Equal");
            var childsort = new QuerySort().Add(item.ModelConfig.KeyName, SortEnum.Aesc);
            var childval = item.SelectViewData(childCondition, childsort);
            eval = eval.Extensions().AddProperty(item.EType.Name, childval);                   
        }               
    }
    //添加附件数据
    var filelist= new SysUploadFileService().SelectList(n => n.DocService == this.Name && n.DocNum == id.ToString()&& n.IsDelete==false,false);
    if (filelist.Count > 0)
    {
        eval = eval.Extensions().AddProperty("SysUploadFile", filelist);
    }
    return XueQ.Utility.Result.OK(eval);
}
/// <summary>
/// 查询所有数据
/// </summary>
/// <param name="condition"></param>
/// <param name="sort"></param>
/// <returns></returns>
public override object SelectViewData(QueryCondition condition, Stone.AQH.QuerySort sort)
{
    return SelectList(condition, sort);
}
/// <summary>
/// 查询分页数据
/// </summary>
/// <param name="total"></param>
/// <param name="startIndex"></param>
/// <param name="count"></param>
/// <param name="condition"></param>
/// <param name="sort"></param>
/// <returns></returns>
public virtual object SelectPagedViewData(ref long total, int startIndex, int count, QueryCondition condition, QuerySort sort)
{
    return SelectPagedList(ref total, startIndex, count, condition, sort);
}

```