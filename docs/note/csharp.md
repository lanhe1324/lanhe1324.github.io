## 反射按地址传递变量的方法
 - Type.GetType("System.Int64&")

 ```csharp
private void PageControl_OnPageChange(object sender, int pageSize, int pageIndex,ref long recordCount)
{
    if (ShowChildView && BSvc.ChildService != null)
    {       
        var m = BSvc.ChildService.GetType().GetMethod("SelectPagedList", new Type[] { Type.GetType("System.Int64&"), typeof(int), typeof(int), typeof(QuerySort), typeof(QueryCondition) });
        object[] objs = new object[] { recordCount, pageIndex, pageSize, this.Sort, this.Condition };
        m.Invoke(BSvc.ChildService, objs);
        recordCount =(long)objs[0];
        return;
    }
    BSvc.SelectPagedList(ref recordCount, pageIndex, pageSize, this.Sort, this.Condition);           
}   
 ```