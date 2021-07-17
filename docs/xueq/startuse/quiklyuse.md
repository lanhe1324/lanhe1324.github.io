## 开始使用
列如添加一个`SysUser`表到框架中，只需要按以下步骤进行操作即可实现最基本的功能。

## 1. 新建实体
> 在Model项目里新建Entity和View实体，并且要继承Base基类。
- Entity对应数据中实体表，所以必须与数据库中的字段一样，否则运行时会报错。
- View就好比是数据库里的视图，在这里我们只需要定义在框架中，而不需要在数据库里定义视图了。
1. 新建`SysUserEntity`，包含3个外键表`SysRole`|`SysDept`|`SysPosition`
   > 外键实体可以根据业务的需要添加`Virtual`关键字
```csharp
public class SysUserEntity : Base.BaseEntity
    {
        public string Id { get; set; }
        public string Name { get; set; }
        public string Password { get; set; }
        public int DeptId { get; set; }
        public int PositionId { get; set; }
        public int RoleGroupId { get; set; }
        public bool IsDeptHead { get; set; }
        public string Email { get; set; }
        public string Description { get; set; }
        public string LastLoginIp { get; set; }
        public DateTime? LastLoginDate { get; set; }

        public string CreateBy { get; set; }
        public DateTime CreateDate { get; set; }
        public bool IsDisabled { get; set; }
        public bool IsDel { get; set; }       
        public  SysRoleGroupEntity SysRoleGroup { get; set; }
        public  SysDeptEntity SysDept { get; set; }
        public virtual SysPositionEntity SysPosition { get; set; }      
    }
```
1. 新建`SysUserView`,视图应是`扁平化`的，只定义需要的外键表字段即可： 
    - 定义`SysDeptEntity`外键实体`Name`字段，框架会根据定义自动映射：
      1. 外键实体名_字段名(SysDept_Name) 
      2. 通过添加标记(`MappedName`("SysDept.Name"))
```csharp
 public class SysUserView : Base.BaseView
    {       
        public string Id { get; set; }
        public string Name { get; set; }
        [Stone.AQH.Attribute.IgnoreMapped]
        [JsonIgnore]
        public string Password { get; set; }
        public int? DeptId { get; set; }
        public int? PositionId { get; set; }
        public int? RoleGroupId { get; set; }
        public bool IsDeptHead { get; set; }
        public string Email { get; set; }
        public string Description { get; set; }
        public string LastLoginIp { get; set; }
        public DateTime? LastLoginDate { get; set; }
        public string CreateBy { get; set; }        
        public DateTime CreateDate { get; set; }
        //[JsonConverter(typeof(BoolConvert))]
        public bool IsDisabled { get; set; }
       
        public bool IsDel { get; set; }

        public bool IsSuperAdmin { get; set; }
        [JsonIgnore]
        public bool IsSuperLeader { get; set; }     

        [MappedName("SysDept.Name")]
        public string DeptName { get; set; }
        [MappedName("SysPosition.Name")]
        public string PositionName { get; set; }
        public int SysPosition_Grade { get; set; }
        [MappedName("SysRoleGroup.Name")]
        public string RoleGroupName { get; set; }
       
    }
```

## 2. 添加EF配置
> 在Service项目里添加实体的`SysUserConfig`配置，继承`EntityTypeConfiguration`基类
- 传入泛型参数`SysUserEntity`

```csharp
public class SysUserConfig : EntityTypeConfiguration<SysUserEntity>
    {
        public SysUserConfig()
        {
            this.ToTable("SysUser");
            this.HasKey<string>(k => k.Id);
            //SysUserEntity必须有SysRoleGroup，一个SysRoleGroup有很多的SysUserEntity，他们使用RoleGroupId做外键
            this.HasRequired(c => c.SysRoleGroup).WithMany().HasForeignKey(e => e.RoleGroupId).WillCascadeOnDelete(false);
            this.HasRequired(c => c.SysDept).WithMany().HasForeignKey(e => e.DeptId).WillCascadeOnDelete(false);
            this.HasRequired(c => c.SysPosition).WithMany().HasForeignKey(e => e.PositionId).WillCascadeOnDelete(false);
        }
    }
```

## 3. 添加DbContext
> 在Service项目里添加`SysUserService`,并继承`BaseServiceVirtual`基类;
> - 通过继承，子类拥有父类的一切属性和行为
- 传入泛型参数`SysUserEntity`和`SysUserView`
- 在构造函数中设置连接名称

```csharp
 public class SysUserService : Base.BaseServiceVirtual<SysUserEntity, SysUserView>
    {
        public SysUserService()
        {
            base.ConStringName = XueQ.Model.XEnum.ConStringName.Web;
        }
    }
```

## 4. 添加Controller
> 在MVC项目里添加`SysUserController`,继承`BaseController`。
> - 由于之前的项目用的是.Net Mvc的框架，就不在新建新的Api项目，直接修改了原有的框架，添加相应Api接口操作数据。
- 添加SysUserService字段,运行时框架会自动实例化
```csharp
 public class SysUserController : Base.BaseController
    {
        public Service.Context.SysUserService Svc;      
        public SysUserController()
        {
           
        }
    }    
```

## 5. Api操作
> 经过以上几个步骤后我们就拥有了Base基类的所有功能了，如果你不需要进行复杂的业务逻辑，那么通过以下Api接口就可以对表进行`增删改查`操作了。

| 接口名 | 描述 |
|:----|---|
|  ApiList   |获取对象集合|
|  ApiAddNew   |新增|
|  ApiEdit   |编辑|
|  ApiDelete   |删除|
|  ApiSelectComboBox   |获取下拉集合数据|
|  ApiColConfig   |获取字段配置|
|  ApiWriteToolbar   |写入工具栏|
|  ApiExportExcel   |导出Excel|


```js
import request from '@/utils/request'
export function getCol(svcname, params) {
  return request({
    url: `/${svcname}/ApiColConfig`,
    method: 'get',
    params
  })
}
export function getList(svcname, params) {
  return request({
    url: `/${svcname}/apilist`,
    method: 'get',
    params
  })
}

export function postList(svcname, data) {
  return request({
    url: `/${svcname}/apilist`,
    method: 'post',
    data
  })
}
export function getAddNew(svcname, params) {
  return request({
    url: `/${svcname}/ApiAddNew`,
    method: 'get',
    params
  })
}
export function postAddNew(svcname, data) {
  return request({
    url: `/${svcname}/ApiAddNew`,
    method: 'post',
    data
  })
}
export function postEdit(svcname, data) {
  return request({
    url: `/${svcname}/ApiEdit`,
    method: 'post',
    data
  })
}
export function getEdit(svcname, id) {
  const params = { id: id }
  return request({
    url: `/${svcname}/ApiEdit`,
    method: 'get',
    params
  })
}
export function postDelete(svcname, data) {
  return request({
    url: `/${svcname}/ApiDelete`,
    method: 'post',
    data
  })
}
export function postWriteToolbar(svcname, data) {
  return request({
    url: `/${svcname}/apiWriteToolbar`,
    method: 'post',
    data
  })
}
export function postExportExcel(svcname, data) {
  return request({
    url: `/${svcname}/ApiExportExcel`,
    method: 'post',
    data,
    responseType: 'blob'
  })
}
export function getComboBox(svcname, id,name) {
  const params = { id: id,name:name }
  return request({
    url: `/${svcname}/ApiSelectComboBox`,
    method: 'get',
    params
  })
}
```