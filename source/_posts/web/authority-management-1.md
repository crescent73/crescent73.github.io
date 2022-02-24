---
title: 权限管理（一）
date: 2022-02-24 22:59:46
categories:
- web
tags:
- web
- 权限管理
---

# 权限管理解决方案之 vue antd admin
## 前言
最近一直在做web权限管理，预期目标是做一个基于用户角色的权限管理（RBAC, Role-Based Access Control）。刚开始做的时候没什么思路，只是觉得应该根据用户角色把基本的web权限给分开。  
由于现在做的是前后端分离项目，RBAC里面有个核心的要点是基本用户角色，来动态展示系统菜单。举个例子就是，假如我们系统上现在有用户管理、角色管理、日志管理这几个大模块，我们为每个模块设定角色，拥有相应角色的人才能进行管理操作（增删改）。最简单的，日志管理员登录到系统上，系统应该只显示日志管理这一大块菜单，其他菜单都不显示。当然，最简单的，可以把这些角色都预先定义好，前端根据角色隐藏菜单。但是这样，就做不了角色权限的动态更改、动态授权。  
为了实现以上功能，我基于vue antd admin这个前端框架，提出了一种解决方案。现在已经可以实现基本的授权功能。但是在做后端的时候，有些细节我感觉还是有点问题。后来我又仔细研究了一下rouyi的权限管理，感觉ruoyi的权限管理方案更好。所以我决定参考ruoyi的权限管理方案，把前后端再重写一遍。  
这个博客记录一下我基于antd的权限管理方案。
## 基于antd的权限管理
antd框架本身提供了一些权限管理方案。这里放一下官方文档：[文档](https://iczer.gitee.io/vue-antd-admin-docs/advance/async.html)  
antd的权限管理方案思路如下。
### 异步路由和菜单设置
首先，需要开启异步路由，系统的路由菜单需要根据用户角色动态生成。这里，antd已经在官方文档上提供的解决方案。步骤如下：
1. 开启异步路由  
在 `/config/config.js` 文件中设置 `asyncRoutes` 的值为 true
``` js
module.exports = {
  theme: {
    color: '#13c2c2',
    mode: 'night'
  },
  multiPage: true,
  asyncRoutes: true,       //异步加载路由，true:开启，false:不开启
  animate: {
    name: 'roll',
    direction: 'default'
  }
}
```
2. 注册路由组件  
这里需要把之前树状的路由组件，全部平铺拆分成单个并取名。基础路由组件包含路由基本配置和对应的视图组件，统一在 `/router/async/router.map.js` 文件中注册。
``` js
// 视图组件
const view = {
  tabs: () => import('@/layouts/tabs'),
  blank: () => import('@/layouts/BlankView'),
  page: () => import('@/layouts/PageView')
}

// 路由组件注册
const routerMap = {
  login: {
    authority: '*',
    path: '/login',
    component: () => import('@/pages/login')
  },
  root: {
    redirect: '/home',
    component: view.tabs // 表明这是tabs
  },
  home: {
    authority: '*',
    component: () => import('@/pages/Home')
  },
  entity: {
    authority: '*',
    component: () => view.blank()
  },
  entityList: {
    component: () => import('@/pages/entity/Entity')
  },
  entityAdd: {
    component: () => import('@/pages/entity/AddEntity')
  },
  relationAdd: {
    component: () => import('@/pages/entity/AddRelation')
  },
  entityDetail: {
    component: () => import('@/pages/entity/EntityDetail')
  },
  user: {
    component: () => view.blank()
  },
  userList: {
    component: () => import('@/pages/user/UserManage')
  },
  roleList: {
    component: () => import('@/pages/user/RoleManage')
  },
  exp403: {
    authority: '*',
    name: 'exp403',
    path: '403',
    component: () => import('@/pages/exception/403')
  },
  exp404: {
    name: 'exp404',
    path: '404',
    component: () => import('@/pages/exception/404')
  },
  exp500: {
    name: 'exp500',
    path: '500',
    component: () => import('@/pages/exception/500')
  },
  exception: {
    name: '异常页',
    icon: 'warning',
    component: view.blank
  }
}
export default routerMap
```
3. 配置基本路由  
不配置路由是无法访问页面的，所以我们需要配置一些基本路由，如login,404等。系统菜单的路由可以等登录后再动态加载。  
基本路由在 `/router/async/config.async.js` 文件中配置。
``` js
import routerMap from './router.map'
import {parseRoutes} from '@/utils/routerUtil'

// 异步路由配置
const routesConfig = [
  'login',
  {
    router: 'exp404',
    path: '*',
    name: '404'
  },
  {
    router: 'exp403',
    path: '/403',
    name: '403'
  }
]

const options = {
  routes: parseRoutes(routesConfig, routerMap)
}

export default options
```

4. 异步获取路由配置  
一般在登录完成后，我们会向后端获取路由（菜单）配置。后端返回的路由数据应该格式如下：
``` js
[{
  router: 'root',                           //匹配 router.map.js 中注册名 registerName = root 的路由
  children: [                               //root 路由的子路由配置
    {
      router: 'dashboard',                  //匹配 router.map.js 中注册名 registerName = dashboard 的路由
      children: ['workplace', 'analysis'],  //dashboard 路由的子路由配置，依次匹配 registerName 为 workplace 和 analysis 的路由
    },
    {
      router: 'form',                       //匹配 router.map.js 中注册名 registerName = form 的路由
      children: [                           //form 路由的子路由配置
        'basicForm',                        //匹配 router.map.js 中注册名 registerName = basicForm 的路由
        'stepForm',                         //匹配 router.map.js 中注册名 registerName = stepForm 的路由
        {
          router: 'advanceForm',            //匹配 router.map.js 中注册名 registerName = advanceForm 的路由
          path: 'advance'                   //重写 advanceForm 路由的 path 属性
        }
      ]   
    },
    {
      router: 'basicForm',                  //匹配 router.map.js 中注册名 registerName = basicForm 的路由
      name: '验权表单',                     //重写 basicForm 路由的 name 属性
      icon: 'file-excel',                   //重写 basicForm 路由的 icon 属性
      authority: 'form'                     //重写 basicForm 路由的 authority 属性
    }
  ]
}]
```
其中 `router` 属性 对应 `router.map.js` 中已注册的基础路由的注册名称 `registerName`，`children` 属性为路由的嵌套子路由配置。
如果想重写已注册路由的属性，可以在 `routesConfig` 配置同名属性去覆盖它。  
在具体实现的时候，我用antd自带的mock.js中的模拟接口实现后端的接口功能。我的 `routesConfig` 配置如下：
``` js
Mock.mock(`${process.env.VUE_APP_API_BASE_URL}/routes`, 'get', () => {
  let result = {}
  result.code = 0
  result.data = [{
    router: 'root',
    path: '/',
    name: '首页',
    children: [{
      router: 'home',
      name: '首页',
      icon: 'dashboard',
    },{
      router: 'entity',
      name: '实体',
      path: 'entity',
      icon: 'profile',
      authority: {
        role: [1,2,3]
      },
      children: [{
        router:'entityList',
        path: 'list',
        name: '实体列表',
        icon: 'profile',
        authority: {
          role: [1,2,3]
        },
      },{
        router:'entityAdd',
        path:'add',
        name: '添加实体',
        icon: 'profile',
        authority: {
          role: [1,3]
        },
      },{
        router:'relationAdd',
        path:'relation/add',
        name: '添加关系',
        icon: 'profile',
        authority: {
          role: [1,3]
        },
      }]
    },{
      router: 'entityDetail',
      name: '实体详情',
      icon: 'profile',
      invisible: true,
      authority: {
        role: [1,2,3]
      },
      path: 'detail/:label/:id'
    },{
      router: 'user',
      name: '用户',
      path: 'user',
      icon: 'profile',
      authority: {
        role: [1,4]
      },
      children: [{
        router:'userList',
        path: 'list',
        name: '用户列表',
        icon: 'profile',
        authority: {
          role: [1,4]
        },
      },{
        router:'roleList',
        path:'role',
        name: '角色列表',
        icon: 'profile',
        authority: {
          role: [1,4]
        },
      }]
    }]
  }]
  return result
})
```

5. 加载路由并应用  
antd提供了一个路由加载工具，我们只需调用 `/utils/routerUtil.js` 中的 `loadRoutes` 方法加载上一步获取到的 routesConfig 即可。
``` js
getRoutesConfig().then(result => {
  const routesConfig = result.data.data
  loadRoutes(routesConfig)
})
```
### 角色和权限设置
在上面的动态路由里可以看到有设置 `authority` 属性，这个就是权限设置。  
antd提供了两种授权方式，分别是 `role` 和 `permission` 。在具体使用的时候，我只使用了 `role` 授权。具体内容如下。  
1. 设置用户角色  
在登录成功后，我们就应该动态获取用户角色，antd 的 `角色/role` 包含 `id` 和 `operation` 两个属性。其中 `id` 为 `角色/role` 的 `id`，`operation` 为 `角色/role` 具有的操作权限，是一个字符串数组。
``` js
role = {
  id: 'admin',                                   //角色ID
  operation: ['add', 'delete', 'edit', 'close']  //角色的操作权限
}
```
异步请求用户角色代码如下，这个地方我根据后端返回的数据格式做了以下数据格式转换。
``` js
getUserRole(res.data.id).then(res => {
    let roles = res.data.data.map(role=>{
        return {
            id: role.id,
            operation: role.operation.map(x=> String(x))
        }
    })
    this.setRoles(roles)
})
```
2. 设置异步路由权限  
页面权限的设置上面其实已经说了，只需要给该页面对应的路由设置元数据 `authority` 即可。下面这个代码表示角色为 `1`, `2`, `3` 的用户可以访问 `entity` 页面。
```
{
    router: 'entity',
    name: '实体',
    path: 'entity',
    icon: 'profile',
    authority: {
    role: [1,2,3]
}
```
3. 设置操作权限  
页面上的一些增删改查按钮，也需要我们进行权限控制。这个时候，可以通过role下的operation来进行设置。比如上面例子里为 `admin` 的角色，拥有 `['add', 'delete', 'edit', 'close']` 的操作权限。那么对于页面里的操作按钮，我们可以通过 ```v-auth:role="`delete`"``` 指令来判断其拥有没有delete权限。
``` html
<div slot="action" slot-scope="{text, record}">
  ...
  <a v-auth:role="`delete`">
    <a-icon type="delete" />删除
  </a>
  ...
</div>
```
如果没有权限的时候想隐藏掉按钮，可以通过 ``` v-if="$auth(`delete`,`role`)"``` 来实现。


通过以上设置，就可以实现前端的权限控制功能。对于对应角色的用户，可以隐藏掉不属于其权限的页面和操作按钮。这种实现方式，完全基于antd这个框架。接下来，只需要把对应的后端代码编写好，就可以实现整套的权限控制方案了。

## 基于mybatis的后端实现
根据上面的描述，后端只需要提供两个接口，第一个接口是返回用户角色，第二个接口是返回带权限的菜单。  
上面两个功能，看起来容易，做起来其实还是有点麻烦的。麻烦的地方主要在于两点。第一，`user -- role -- menu & operation` 均为多对多关系，在设计表的时候，多对多关系需要单独设立一个外键表，在查询的时候，就会麻烦一些。第二，菜单为树状结构，关系型数据库里的表不能直接存储树状结构的数据。
### 数据库设计
数据库设计图如下，主要的表有 `user`, `role`, `menu`, `operation`。`operation` 表用来存储用户的操作权限。其他三个表为多对多关系存储产生的外键表。
![微信截图_20220225000235.png](https://s2.loli.net/2022/02/25/MdtvQsKHpir83aZ.png)

数据库sql
``` sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for menu
-- ----------------------------
DROP TABLE IF EXISTS `menu`;
CREATE TABLE `menu`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `p_id` int NULL DEFAULT NULL,
  `router` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `path` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `icon` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `invisible` tinyint(1) NULL DEFAULT 0,
  `order_num` int NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 11 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of menu
-- ----------------------------
INSERT INTO `menu` VALUES (1, 0, 'root', '首页', '/', '', 0, 1, '2021-12-19 20:13:07', '2022-02-23 16:13:49');
INSERT INTO `menu` VALUES (2, 1, 'home', '首页', 'home', 'dashboard', 0, 1, '2021-12-19 20:13:16', '2022-02-23 16:13:29');
INSERT INTO `menu` VALUES (3, 1, 'entity', '实体', 'entity', 'profile', 0, 2, '2021-12-19 20:13:26', '2022-02-23 16:13:34');
INSERT INTO `menu` VALUES (4, 1, 'entityDetail', '实体详情', 'detail/:label/:id', 'profile', 1, 4, '2021-12-19 20:13:33', '2022-02-23 16:13:45');
INSERT INTO `menu` VALUES (5, 3, 'entityAdd', '添加实体', 'add', 'profile', 0, 2, '2021-12-19 20:13:46', '2022-02-23 16:13:18');
INSERT INTO `menu` VALUES (6, 3, 'entityList', '实体列表', 'list', 'profile', 0, 1, '2021-12-19 20:13:56', '2022-02-23 16:13:14');
INSERT INTO `menu` VALUES (7, 3, 'relationAdd', '添加关系', 'relation/add', 'profile', 0, 3, '2022-02-22 17:59:47', '2022-02-23 16:13:24');
INSERT INTO `menu` VALUES (8, 1, 'user', '用户', 'user', 'profile', 0, 3, '2022-02-23 15:58:24', '2022-02-23 16:13:39');
INSERT INTO `menu` VALUES (9, 8, 'userList', '用户列表', 'list', 'profile', 0, 1, '2022-02-23 15:58:50', '2022-02-23 16:12:57');
INSERT INTO `menu` VALUES (10, 8, 'roleList', '角色列表', 'role', 'profile', 0, 2, '2022-02-23 15:59:46', '2022-02-23 16:13:01');

-- ----------------------------
-- Table structure for operation
-- ----------------------------
DROP TABLE IF EXISTS `operation`;
CREATE TABLE `operation`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  `key` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of operation
-- ----------------------------
INSERT INTO `operation` VALUES (1, '添加', 'add', '2022-02-22 18:01:19', '2022-02-22 18:01:19');
INSERT INTO `operation` VALUES (2, '删除', 'delete', '2022-02-22 18:01:27', '2022-02-22 18:01:48');
INSERT INTO `operation` VALUES (3, '编辑', 'edit', '2022-02-22 18:01:35', '2022-02-22 18:01:35');

-- ----------------------------
-- Table structure for role
-- ----------------------------
DROP TABLE IF EXISTS `role`;
CREATE TABLE `role`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `key` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `is_deleted` tinyint(1) NULL DEFAULT 0,
  `data_scope` tinyint(1) NOT NULL DEFAULT 1 COMMENT '0拥有所有权限，1为自定义权限',
  `created_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 5 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of role
-- ----------------------------
INSERT INTO `role` VALUES (1, '超级管理员', 'super-admin', 0, 0, 'admin', '2022-02-22 18:02:24', '2022-02-23 17:21:38');
INSERT INTO `role` VALUES (2, '普通用户', 'user', 0, 1, 'admin', '2022-02-22 18:02:47', '2022-02-23 17:21:40');
INSERT INTO `role` VALUES (3, '知识图谱管理员', 'kg-admin', 0, 1, 'admin', '2022-02-23 15:43:53', '2022-02-23 17:21:42');
INSERT INTO `role` VALUES (4, '用户管理员', 'user-admin', 0, 1, 'admin', '2022-02-23 15:44:13', '2022-02-23 17:21:44');

-- ----------------------------
-- Table structure for role_menu
-- ----------------------------
DROP TABLE IF EXISTS `role_menu`;
CREATE TABLE `role_menu`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `role_id` int NULL DEFAULT NULL,
  `menu_id` int NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `r_r_id`(`role_id`) USING BTREE,
  INDEX `r_rs_id`(`menu_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 19 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of role_menu
-- ----------------------------
INSERT INTO `role_menu` VALUES (5, 3, 4, '2022-02-23 15:44:40', '2022-02-23 15:44:40');
INSERT INTO `role_menu` VALUES (6, 3, 5, '2022-02-23 15:44:47', '2022-02-23 15:44:47');
INSERT INTO `role_menu` VALUES (7, 3, 6, '2022-02-23 15:44:53', '2022-02-23 15:44:53');
INSERT INTO `role_menu` VALUES (8, 3, 7, '2022-02-23 15:45:00', '2022-02-23 15:45:00');
INSERT INTO `role_menu` VALUES (10, 3, 2, '2022-02-23 15:55:36', '2022-02-23 15:55:36');
INSERT INTO `role_menu` VALUES (11, 4, 8, '2022-02-23 16:09:41', '2022-02-23 16:09:41');
INSERT INTO `role_menu` VALUES (12, 4, 9, '2022-02-23 16:09:47', '2022-02-23 16:09:47');
INSERT INTO `role_menu` VALUES (13, 4, 10, '2022-02-23 16:09:54', '2022-02-23 16:09:54');
INSERT INTO `role_menu` VALUES (14, 3, 3, '2022-02-23 16:10:20', '2022-02-23 16:10:20');
INSERT INTO `role_menu` VALUES (15, 2, 2, '2022-02-23 16:15:58', '2022-02-23 16:15:58');
INSERT INTO `role_menu` VALUES (16, 2, 3, '2022-02-23 16:16:02', '2022-02-23 16:16:02');
INSERT INTO `role_menu` VALUES (17, 2, 4, '2022-02-23 16:16:07', '2022-02-23 16:16:07');
INSERT INTO `role_menu` VALUES (18, 2, 6, '2022-02-23 16:16:14', '2022-02-23 16:16:14');

-- ----------------------------
-- Table structure for role_operation
-- ----------------------------
DROP TABLE IF EXISTS `role_operation`;
CREATE TABLE `role_operation`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `role_id` int NULL DEFAULT NULL,
  `operation_id` int NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 10 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_0900_ai_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of role_operation
-- ----------------------------
INSERT INTO `role_operation` VALUES (1, 1, 1, '2022-02-22 18:10:01', '2022-02-22 18:10:01');
INSERT INTO `role_operation` VALUES (2, 1, 2, '2022-02-22 18:10:08', '2022-02-22 18:10:08');
INSERT INTO `role_operation` VALUES (3, 1, 3, '2022-02-22 18:10:14', '2022-02-22 18:10:14');
INSERT INTO `role_operation` VALUES (4, 3, 1, '2022-02-23 16:17:13', '2022-02-23 16:17:13');
INSERT INTO `role_operation` VALUES (5, 3, 2, '2022-02-23 16:17:18', '2022-02-23 16:17:18');
INSERT INTO `role_operation` VALUES (6, 3, 3, '2022-02-23 16:17:27', '2022-02-23 16:17:27');
INSERT INTO `role_operation` VALUES (7, 4, 1, '2022-02-23 16:17:33', '2022-02-23 16:17:33');
INSERT INTO `role_operation` VALUES (8, 4, 2, '2022-02-23 16:17:39', '2022-02-23 16:17:39');
INSERT INTO `role_operation` VALUES (9, 4, 3, '2022-02-23 16:17:45', '2022-02-23 16:17:45');

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `password` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `is_deleted` tinyint(1) NOT NULL DEFAULT 0,
  `created_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 10 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of user
-- ----------------------------
INSERT INTO `user` VALUES (1, 'admin', 'bupt-admin001', 0, 'admin', '2021-11-09 20:24:59', '2022-02-23 16:12:24');
INSERT INTO `user` VALUES (2, 'kg-admin', '123456', 0, 'admin', '2021-11-09 20:24:59', '2022-02-23 16:12:26');
INSERT INTO `user` VALUES (4, 'user-admin', '123456', 0, 'admin', '2021-12-11 21:14:39', '2022-02-23 16:12:27');
INSERT INTO `user` VALUES (9, 'user', '123456', 0, 'admin', '2022-02-23 16:12:18', '2022-02-23 16:12:32');

-- ----------------------------
-- Table structure for user_role
-- ----------------------------
DROP TABLE IF EXISTS `user_role`;
CREATE TABLE `user_role`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NULL DEFAULT NULL,
  `role_id` int NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE,
  INDEX `u_r_id`(`user_id`) USING BTREE,
  INDEX `u_u_id`(`role_id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 10 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Records of user_role
-- ----------------------------
INSERT INTO `user_role` VALUES (1, 1, 1, '2022-02-22 18:03:27', '2022-02-22 18:03:27');
INSERT INTO `user_role` VALUES (2, 2, 3, '2022-02-22 18:06:12', '2022-02-23 16:11:38');
INSERT INTO `user_role` VALUES (3, 3, 4, '2022-02-23 16:12:02', '2022-02-23 16:12:02');
INSERT INTO `user_role` VALUES (4, 9, 2, '2022-02-23 16:14:19', '2022-02-23 16:14:19');
INSERT INTO `user_role` VALUES (5, 1, 2, '2022-02-23 19:53:58', '2022-02-23 19:53:58');

SET FOREIGN_KEY_CHECKS = 1;

```

### 用户角色获取接口实现
用户角色接口的返回数据格式如下：
``` json
[{
  "id": 1,
  "operation": [1, 2, 3]
}]
```
其中，主要涉及了三张表，user,role和operation。我们需要获取用户当前拥有的角色，还需要获取角色拥有的操作权限。具体实现的代码如下：
1. 编写对应的实体类  
user和operation实体类和数据库对应就行，role实体添加一个operation属性，类型为Operation类型的列表。这里封装了一个BaseModel超类，存储id，createTime等公共属性。
``` java
@EqualsAndHashCode(callSuper = true)
@Data
@AllArgsConstructor
@NoArgsConstructor
@ApiModel("角色表")
@ToString(callSuper = true)
public class Role extends BaseModel<Role> {

    @ApiModelProperty("角色名")
    private String name;

    @ApiModelProperty("角色字符")
    private String key;

    @ApiModelProperty("是否删除")
    private Boolean isDeleted;

    @ApiModelProperty("权限")
    private Boolean dataScope;

    @ApiModelProperty("创建者")
    private String createdBy;

    private Long[] operation;

}
```
2. 编写RoleDao
``` java
@Mapper
@Repository
public interface RoleDao extends BaseMapper<Role> {
    List<Role> findUserRoleById(Long id);
}
```
3. 编写roleMapper  
核心查询操作有两个。第一个是 `findUserRoleById` ，根据userId查询该用户所拥有的roleId。返回结果类型为roleMap，这个地方的roleMap包含了role的基本属性。其中 `<collection column="id" property="operation" select="findRoleOperation" />` 用来查询role拥有的operation。这个地方调用了第二个查询 `findRoleOperation` , 这里根据用户的角色id，将其权限id查出来，放到operation中去。
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.bupt.web.dao.RoleDao">

    <resultMap id="roleMap" type="Role">
        <id column="id" property="id"/>
        <result column="p_id" property="pId"/>
        <result column="name" property="name"/>
        <result column="key" property="key"/>
        <result column="is_deleted" property="isDeleted"/>
        <result column="data_scope" property="dataScope"/>
        <result column="created_by" property="createdBy"/>
        <result column="created_at" property="createdAt"/>
        <result column="updated_at" property="updatedAt"/>
        <collection column="id" property="operation" select="findRoleOperation" />
    </resultMap>

<!--    查找role的operation-->
    <select id="findRoleOperation" resultType="Long">
        select operation_id from role_operation where role_id = #{id}
    </select>

    <select id="findUserRoleById" resultMap="roleMap">
        select * from role where id in (select role_id from user_role where user_id = #{id})
    </select>
</mapper>
```
4. 编写测试类
``` java
@Test
public void testFindUserRoleDao(){
    List<Role> userRoleById = roleDao.findUserRoleById(1L);
    System.out.println(userRoleById);
}
```

### 带权限的菜单接口实现
这个接口主要两个难点，第一个，返回一个树状的菜单界面；第二个，返回的菜单信息里需要包含权限信息。  
在实际编写代码的时候，我先分别实现了返回树状的菜单和返回带权限的菜单列表（不是树状）的功能。然后想最后一起整合。但是在整个的过程中，又研究了一下ruoyi框架的实现方案，觉得ruoyi的实现方案更好，所以，我觉得不写了hhh。这里记一下已经实现的。

1. Menu实体类  
注意两个属性，roles和children。
``` java
public class Menu extends BaseModel<Menu> {

    @ApiModelProperty("pid")
    private Integer pId;

    @ApiModelProperty("菜单名")
    private String name;

    @ApiModelProperty("路由名")
    private String router;

    @ApiModelProperty("路径")
    private String path;

    @ApiModelProperty("icon")
    private String icon;

    @ApiModelProperty("是否可见")
    private Boolean invisible;

    @ApiModelProperty("顺序")
    private Integer orderNum;

    @ApiModelProperty("权限")
    private Long[] roles;

    @ApiModelProperty("子菜单")
    private List<Menu> children;
}
```
2. MenuDao  
``` java
@Mapper
@Repository
public interface MenuDao extends BaseMapper<Menu> {
    List<Menu> getSourceTree(Integer pId);
    List<Menu> getAllMenu();
}
```
3. MemuMapper  
树级菜单的实现方式是嵌套查询，核心代码是 ` <collection property="children" ofType="Menu" column="id" select="findTreeByPid"/>` 。childern的查询结果是另外一个查询。  
``` xml
<mapper namespace="com.bupt.web.dao.MenuDao">
    <resultMap id="treeVoMap" type="com.bupt.web.model.pojo.Menu">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="p_id" property="pId" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="router" property="router" jdbcType="VARCHAR"/>
        <result column="path" property="path" jdbcType="VARCHAR"/>
        <result column="icon" property="icon" jdbcType="VARCHAR"/>
        <result column="invisible" property="invisible" jdbcType="TINYINT"/>
        <result column="order_num" property="orderNum" jdbcType="INTEGER"/>
        <collection property="children" ofType="com.bupt.web.model.pojo.Menu" column="id" select="findTreeByPid"/>
    </resultMap>

    <select id="getSourceTree" resultMap="treeVoMap">
        select * from menu where p_id = #{pId} order by order_num
    </select>

    <select id="findTreeByPid" resultMap="treeVoMap">
        select * from menu where p_id = #{id} order by order_num
    </select>
</mapper>
```
权限查询实现方式差不多，核心代码是 `<collection column="id" property="roles" ofType="Long"  select="findMenuRole"/>`
``` xml
<mapper namespace="com.bupt.web.dao.MenuDao">
    <resultMap id="menuAuthMap" type="com.bupt.web.model.pojo.Menu">
        <id column="id" property="id" jdbcType="INTEGER"/>
        <result column="p_id" property="pId" jdbcType="INTEGER"/>
        <result column="name" property="name" jdbcType="VARCHAR"/>
        <result column="router" property="router" jdbcType="VARCHAR"/>
        <result column="path" property="path" jdbcType="VARCHAR"/>
        <result column="icon" property="icon" jdbcType="VARCHAR"/>
        <result column="invisible" property="invisible" jdbcType="TINYINT"/>
        <result column="order_num" property="orderNum" jdbcType="INTEGER"/>
        <collection column="id" property="roles" ofType="Long"  select="findMenuRole"/>
    </resultMap>

    <select id="findMenuRole" resultType="Long">
        select role_id from role_menu where menu_id = #{id}
    </select>

    <select id="getAllMenu" resultMap="menuAuthMap">
        select * from menu order by order_num
    </select>
</mapper>
```

其实整合一下就能实现最终效果了，但我总觉得这样嵌套查询效率不太高。然后又看了下ruoyi的实现方案准备转若依的方案了。  