
## 项目结构
```text
└── XueQ
    ├── XueQ.Model                //模型层
        ├── Base                  //基类
        ├── Entity                //数据
        └── View                  //视图
    ├── XueQ.Service              //数据层
        ├── Base                  //基类
        ├── Config                //配置
        └── Context               //实现类DbContext
    ├── XueQ.Utility              //公共方法
    ├── XueQ.WebApi               //Api接口       
        ├── App_Start                //Mvc配置项
        └── Controllers            
            ├── Base              //基类
            └── Sys               //实现类
    ├── XueQ.CustControl          //自定义控件
    ├── XueQ.Win                  //Winform 程序
    ├── XueQ.AutoUpdate           //Winform 自动更新程序
    └── XueQ.WcfService           //Winform 服务端

```

## 主要模块
- Model
- Service
- Utility
- WebApi
- Win
- autoAupdate
- WcfService

## 基本功能
- 通用增删改查模块
- 通用权限管理
- 单个字段的修改记录
- OA审批流程

## 技术选型
- 花裤衩前端框架 [vue-element-admin](https://panjiachen.github.io/vue-element-admin-site/zh/)
-  DevExpress   WinForm
-  .Net MVC
-  Entity Framework  
-  AddFlow  流程图软件
-  MSSQL 


## 开发模式
- `B/S` -- 前后端分离模式
- `C/S` -- 直连模式

## 项目预览
  ***web界面***
![web](../images/web.png)

 ***winform界面***
![win](../images/win.png)
