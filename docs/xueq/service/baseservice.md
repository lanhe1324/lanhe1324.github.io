## 基本描述
  > - 泛型基类,继承自`AbsService`重写部分方法和属性;    
  > - 实现最基本的数据操作;
  > - T类型参数:对应数据实体`Entity`
  > - V类型参数:对应数据视图`View`
```cs
public abstract class BaseService<T,V>: AbsService,IService where T : class, new() where V : class, new()
{
     public BaseService()
      {            
          this.DefualtQueryCondition = new QueryCondition();
      }
}
```

## ConStringName
> 默认值为是Default，可以在实现类的构造函数里修改连接名称
```cs
protected ConStringName ConStringName = ConStringName.Default;
```

## DefualtQuerySort
> 默认值为主键降序,可以在实现类的构造函数里添加排序字段
```cs
protected QuerySort DefualtQuerySort { get; set; }
protected QuerySort GetDefualtQuerySort 
{ 
    get 
    { 
        return DefualtQuerySort??(DefualtQuerySort= new QuerySort().Add(ModelConfig.KeyName, SortEnum.Desc)); 
    } 
}
public SysMenuService()
{
    base.ConStringName = XueQ.Model.XEnum.ConStringName.Default;
    base.DefualtQuerySort = new Stone.AQH.QuerySort().Add(nameof(SysMenuEntity.FatherId), Stone.AQH.SortEnum.Aesc);  
}

```

## DefualtQueryCondition
> 默认查询条件，可以在实现类的构造函数添加具体条件

```cs
protected QueryCondition DefualtQueryCondition { get; set; }
public SysMenuService()
{       
    base.DefualtQueryCondition.Add(nameof(SysMenuEntity.IsDevelop), false);    
}
```

## ModelConfig
> 通过提取EFConfig，获取实体配置,取得`主键名`、`外键名`等等信息
```cs
public override DbModelConfig ModelConfig { get { return Db.Extensions().ModelConfig<T>(); } }
```

## EntityFieldConfig
> 实体字段配置,通配置可以设置用户对字段的权限`只读`、`可见`、`必填`等等
```cs
private List<XueQ.Model.View.SysEntityFieldConfigView> _entityFieldConfig;
public override List<XueQ.Model.View.SysEntityFieldConfigView> EntityFieldConfig
{
    get { return _entityFieldConfig ?? (_entityFieldConfig = GetEntityFieldConfig()); }
}
protected virtual List<SysEntityFieldConfigView> GetEntityFieldConfig()
{
    var result = new SysEntityFieldConfigService().SelectList(n => n.MName.Equals(this.EType.Name));
    this.User?.SetEntityPermission(result);
    return result;
}
```

## GetThisPrimaryKeyVal
> 取主键值
```cs
public object GetThisPrimaryKeyVal()
{
    if (this.Current==null)
    {
        return null;
    }            
    return this.Current.Extensions().GetValue(ModelConfig.KeyName);
}      
public object GetThisPrimaryKeyVal(object entity)
{           
    return entity.Extensions().GetValue(ModelConfig.KeyName);
}
```

## 子类公共方法
> 一些对数据操作最基本的方法，只能在子类里使用
```cs
protected int Add(T t)
{
    Db.Set<T>().Add(t);
    return Db.SaveChanges();
}
protected T GetOne(params object[] keys)
{
    return Db.Set<T>().Find(keys);
}
protected int Remove(T t)
{
    Db.Set<T>().Remove(t);
    return Db.SaveChanges();
}
protected virtual int Remove(object Id)
{
    var t = GetOne(Id);
    Db.Set<T>().Remove(t);
    return Db.SaveChanges();
}
protected IQueryable<T> QWhere(Expression<Func<T, bool>> where = null)
{
    if (where == null)
    {
        return Db.Set<T>();
    }
    return Db.Set<T>().Where(where).AsQueryable();
}
protected IQueryable<T> QWhereCondition(Expression<Func<T, bool>> where = null)
{
    var queryable = QWhere(where);
    if (this.User != null)
    {
        //获取数据权限
        var dataPoint = EntityFieldAttInfo.FirstOrDefault(n => n.IsDataPoint);
        if (dataPoint != null)
        {
            var values = dataPoint?.DataPointValues;
            queryable = queryable.Contains(dataPoint.Name, values);//添加数据权限，此处为包含查询in(1,2,3)
        }
        this.ResetViewOwn();//重设私有数据的权限
        //IsViewOwn=true时只能查询私有数据,默认筛选条件为CreateBy
        if (this.IsViewOwn && this.User.IsSuperAdmin == false)
        {
            string field = "CreateBy";
            var viewOwnName = EntityFieldAttInfo.FirstOrDefault(n => n.IsViewOwnName);
            if (viewOwnName != null) field = viewOwnName.Name;           
            this.DefualtQueryCondition.Add(field, this.User.Id);                  
        }               
    }
    queryable = queryable.Where(this.DefualtQueryCondition);
    return queryable;
}
protected IQueryable<T> QWhereConditionJoin(Expression<Func<T, bool>> where = null)
{
    var result = this.QWhereCondition(where);

    return result.Join<T>();//添加关联查询
}

```

## Delete方法

```cs


```

