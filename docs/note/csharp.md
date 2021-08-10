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


 ## 查询每月的双休日

 ```cs
public void Weekday()
{
    int Y = 2011;                                      // 记录年份，可以改动，也可以通过控件输入   
    int M = 6;                                         // 记录月份，可以改动，也可以通过控件输入   

    List<int> Weekend_list = new List<int>();          // 用来记录月份中双休日的编号（从1开始）   

    int Month_number = DateTime.DaysInMonth(Y, M);     // 用来记录一个月中的天数   

    for (int i = 1; i <= Month_number; i++)
    {
        if (Whether_Weekend(Y, M, i))
        {
            Weekend_list.Add(i);
        }
    }
    Console.WriteLine("{0}年{1}月共有{2}个双休日，分布如下：/r", Y, M, Weekend_list.Count);
    foreach (int item in Weekend_list)
    {
        Console.WriteLine("{0}号/r", item);
    }
    Console.ReadLine();

}
public static bool Whether_Weekend(int y, int m, int d)
{
    if (m == 1 || m == 2)
    {
        m += 12;
        y--;
    }
    int week = (d + 2 * m + 3 * (m + 1) / 5 + y + y / 4 - y / 100 + y / 400) % 7;   // 基姆拉尔森公式   
    if (week == 5 || week == 6)
    {
        return true;
    }
    else
    {
        return false;
    }
}
 ```