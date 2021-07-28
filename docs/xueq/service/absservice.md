## 描述
> [!Tip|label:主要功能]
> - 定义基本的方法和属性;
> - 主从表操作时便于使用`子表`方法和属性;
## 代码
```cs
 public abstract class AbsService
    {
        public abstract XueQDbContext Db { get; set; }
        public abstract bool IsDefualtConString { get; }
        public abstract List<AbsService> ChildServices { get; set; }
        public abstract AbsService ParentService { get; set; }
        public  XueQ.Model.View.SysUserView User { get; set; }
        public abstract Type EType { get; }
        public abstract Type VType { get; }  
        public abstract DbModelConfig ModelConfig { get; }
        public abstract List<XueQ.Model.View.SysEntityFieldConfigView> EntityFieldConfig { get; }
        public abstract object Current { get; }
        public abstract int Count { get; }     
        public abstract bool IsMaster { get; }
        public abstract bool HasChild { get; }
       
        public abstract void AddNew();      
        public abstract void SvcBeforeUpdate(MethodType methodName, ExecuteResult result);
        public abstract void SvcAfterUpdate(MethodType methodName, ExecuteResult result);     
        public abstract void LocalReload();      
        public abstract void BindEditData(Stone.AQH.QueryCondition condition = null);
        public abstract void BindEditData(object[] entitys);       
        public abstract object SelectViewData(QueryCondition condition, QuerySort sort);
 
    }

```
     