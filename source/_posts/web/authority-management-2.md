---
title: 权限管理（二）
date: 2022-03-13 09:05:46
categories:
- web
tags:
- web
- 权限管理
- vue antd admin
- ruoyi
---

# 权限管理方案之ruoyi
ruoyi的权限管理方案（主要指用户、菜单、角色管理）。
## 数据库设计  
用户、菜单、角色为多对多关系，所以就应该有五张表，用户表、角色表、用户角色表、菜单表、角色菜单表。  
其中还有个问题，就是角色，不光拥有菜单权限（查看菜单），还有操作权限（增删改查），操作权限在页面上通常就是一个按钮，怎么设置操作权限？之前的处理方法是，在数据库中添加了一个操作表，把用户表和操作表相关联。这种方法的缺点就是，表太多，操作起来有些不方便。  
ruoyi的处理方法是，在菜单表里多加了权限字段和菜单类型字段。
+ 菜单类型分为M目录、C菜单和F按钮。目录和菜单作为路由菜单，返回给前端，按钮作为操作权限，在分配角色的操作权限时展示。
+ 权限字段和菜单一一对应，根据菜单的层级标记区分。举个例子，系统管理下的用户管理菜单的添加用户权限，其权限字段为 `system:user:add` 。这种层级的写法，保证了权限字段的唯一性。  

重新设计后的数据库ER图为： 
![20220313093802.png](https://s2.loli.net/2022/03/13/pyNlAYmSdVv6w8Q.png)

数据库SQL为：
``` sql

SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for menu
-- ----------------------------
DROP TABLE IF EXISTS `menu`;
CREATE TABLE `menu`  (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '菜单ID',
  `p_id` int NULL DEFAULT NULL COMMENT '父菜单ID',
  `router` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '菜单名称',
  `path` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '路由地址',
  `icon` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '菜单图标',
  `invisible` tinyint(1) NULL DEFAULT 0 COMMENT '菜单状态（0显示 1隐藏）',
  `order_num` int NULL DEFAULT NULL COMMENT '显示顺序',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `type` char(1) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '' COMMENT '菜单类型（M目录 C菜单 F按钮）',
  `is_deleted` tinyint(1) NULL DEFAULT 0 COMMENT '菜单状态（0正常 1停用）',
  `permission` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '' COMMENT '权限标识',
  `component` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL COMMENT '组件路径',
  `redirect` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '',
  `created_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `updated_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 25 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Table structure for role
-- ----------------------------
DROP TABLE IF EXISTS `role`;
CREATE TABLE `role`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `key` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `is_deleted` tinyint(1) NULL DEFAULT 0,
  `menu_check_strictly` tinyint(1) NULL DEFAULT 1 COMMENT '菜单树选择项是否关联显示',
  `created_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `updated_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '',
  `remark` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 7 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

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
) ENGINE = InnoDB AUTO_INCREMENT = 67 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

-- ----------------------------
-- Table structure for user
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `nickname` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `gender` tinyint(1) UNSIGNED ZEROFILL NULL DEFAULT NULL COMMENT '0男 ，1 女',
  `phone` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `email` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `password` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `is_deleted` tinyint(1) NOT NULL DEFAULT 0,
  `created_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `updated_by` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `remark` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT '' COMMENT '备注',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 12 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

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
) ENGINE = InnoDB AUTO_INCREMENT = 22 CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = DYNAMIC;

SET FOREIGN_KEY_CHECKS = 1;
```

## 后端
后端最主要的有两个问题：第一个，多对多关系根据某一个id拿到关联的数据；第二个，树状结构的数据处理。  
+ 之前对于多对多关系查询，感觉嵌套查询和关联查询都可以。学习了ruoyi的实现方案后决定，以后都用关联查询！
+ 对于树状结构的数据处理，之前是用的mybatis自带的查询，就是查询结果碰到嵌套的情况，mybatis会继续查sql。而ruoyi的实现方案是，首先根据多表关联查询，把所有数据一次性查出来，然后在java中，通过递归处理构建菜单树。这样处理的好处是，节省和数据库操作的IO开销。

下面根据新的思路，重写用户角色操作接口和带权限的菜单接口实现。

### 用户角色获取接口实现
用户角色接口的返回数据格式和之前一样，这次，operation应该是从菜单表里查出来的菜单权限。
``` json
[{
  "id": 1,
  "operation": [1, 2, 3]
}]
```
主要涉及的还是三张表，user,role和menu。三表之间直接用多表关联查询查出结果。具体实现如下分为两步步骤，第一，先把当前用户的角色都查出来，第二，根据查询出来的角色，查询角色所拥有的权限。
1. 实体类  
这里，数据的返回类型是Role，但又不仅仅只有Role的基本字段，还有Role的操作字段，所以需要加一个字段 operation。
``` java
public class Role extends BaseModel<Role> {

    @ApiModelProperty("角色名")
    private String name;

    @ApiModelProperty("角色字符")
    @TableField("`key`")
    private String key;

    @ApiModelProperty(hidden = true)
    private Boolean isDeleted;

    @ApiModelProperty("备注")
    private String remark;

    @ApiModelProperty(hidden = true)
    @TableField(exist = false)
    private Set<String> operation;

    /** 菜单组 */
    @TableField(exist = false)
    @ApiModelProperty(hidden = true)
    private List<Long> menuIds;
}
```
2. Dao层接口  
Dao层定义接口，具体实现在mapper中  
Role Dao
``` java
public interface RoleDao extends BaseMapper<Role> {
    List<Role> selectRolePermissionByUserId(Long userId);
}
```
Menu Dao
``` java
/**
* 根据角色ID查询权限
*
* @param roleId 用户ID
* @return 权限列表
*/
Set<String> selectMenuPermsByRoleId(Long roleId);
```
3. Mapper 实现  
重头戏，核心要点在于多表关联查询，然后根据id字段筛选结果。  
RoleMapper
``` xml
<mapper namespace="com.bupt.web.dao.RoleDao">
    <sql id="selectRoleVo">
        select distinct r.*
        from role r
             left join user_role ur on ur.role_id = r.id
             left join user u on u.id = ur.user_id
    </sql>

    <select id="selectRolePermissionByUserId" parameterType="Long" resultType="Role">
        <include refid="selectRoleVo"/>
        where r.is_deleted = 0 and ur.user_id = #{userId}
    </select>
</mapper>
```
MenuMaper
``` xml
<select id="selectMenuPermsByRoleId" parameterType="Long" resultType="String">
    select distinct m.permission as operations
    from menu m
        left join role_menu rm on m.id = rm.menu_id
        left join role ro on rm.role_id = ro.id
    where m.is_deleted = 0 and ro.is_deleted = 0 and ro.id = #{roleId} and m.permission != ''
</select>
```
4. Service实现  
RoleService  
这里的基本逻辑也很简单，首先，根据UserId查询出来用户所拥有的角色。然后对角色列表进行遍历，挨个查询角色所拥有的菜单权限。最后对数据进行格式转换，返回前端需要的数据格式。
``` java
public List<RoleVo> getRolePermissionByUserId(Long userId) {
    // 查询用户角色
    List<Role> roleList = roleDao.selectRolePermissionByUserId(userId);
    // 给用户角色添加权限
    roleList.forEach(role -> role.setOperation(menuService.selectMenuPermsByRoleId(role.getId())));
    // 数据类型转换
    return roleList.stream().map(RoleVo::new).collect(Collectors.toList());
}
```
这里再单独说一下数据转换，这里用了java 8里的流处理操作（stream和map）。map这里是通过构造方法转换的，大概思想就是创建RoleVo对象的时候，接收一个Role对象？？大家意会一下，具体的自己百度吧。放一下RoleVo的定义。
``` java
@Data 
public class RoleVo {
    private Long id;
    private String name;
    private String key;
    private Set<String> operation;

    public RoleVo(Role role) {
        this.id = role.getId();
        this.name = role.getName();
        this.key = role.getKey();
        this.operation = role.getOperation();
    }
}
```

MenuService
MenuService这里其实是有个逻辑操作的。就是超级管理员拥有所有权限，所以如果是超级管理员（默认的id为1），直接返回 `*:*:*` 字符，不需要查询。具体实现如下：
``` java
public Set<String> selectMenuPermsByRoleId(Long roleId) {
    Set<String> perms = new HashSet<>();
    if (roleId.equals(1L)) {
        perms.add("*:*:*"); // 管理员拥有所有权限
    } else {
        perms.addAll(menuDao.selectMenuPermsByRoleId(roleId));
    }
    return perms;
}
```

5. Controller实现  
很简单，没什么多说的，ResponseData是自己封装的返回数据格式。
``` java
@GetMapping("/user")
public ResponseData getRolePermission(){
    Long userId = BaseController.getUserId();
    if (userId == null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    List<RoleVo> roleList = roleService.getRolePermissionByUserId(userId);
    return ResponseData.success(roleList);
}
```

### 带权限的菜单接口实现
这个接口的功能之前应该说过，就是给定用户Id，返回当前用户所拥有的菜单权限。之前antd的解决方案里，其实返回的是所有的路由菜单，然后在路由菜单中标记了拥有这个菜单权限的角色。而ruoyi是在后端根据用户的角色去对菜单进行查询，返回的就是这个角色所拥有的菜单，不是所有菜单。  
具体实现的时候，核心要点还是两个，第一个是查询出这个用户拥有的菜单权限，第二个是将查询出来的菜单列表，转换成树状结构。第一个要点的解决方法是内联表，第二个要点的解决方法是递归。下面说下具体实现：
1. Menu实体类  
由于Menu是树状的嵌套结构，所以需要添加一个children属性。
``` java
public class Menu extends BaseModel<Menu> {

    @ApiModelProperty("pId")
    @JsonProperty("pId")
    private Long pId;

    @ApiModelProperty("菜单名")
    private String name;

    @ApiModelProperty("路径")
    private String path;

    @ApiModelProperty("icon")
    private String icon;
    @ApiModelProperty("重定向")
    private String redirect;
    @ApiModelProperty("是否可见")
    private Boolean invisible;
    @ApiModelProperty("顺序")
    private Integer orderNum;
    @ApiModelProperty("菜单类型")
    private String type;
    @ApiModelProperty("是否启用")
    private Boolean isDeleted;
    @ApiModelProperty("菜单权限")
    private String permission;
    @ApiModelProperty("菜单组件")
    private String component;

    @TableField(exist = false)
    @ApiModelProperty(hidden = true)
    private List<Menu> children = new ArrayList<>();
}
```

2. MenuDao  
Dao层的接口定义。这里定义了两个接口，第一个是查询所有的菜单接口（超级管理员拥有所有的菜单权限），第二个是根据用户Id查询其拥有的菜单接口。
``` java
public interface MenuDao extends BaseMapper<Menu> {
    /**
     * 查询所有菜单列表
     * @return 菜单列表
     */
    List<Menu> selectMenuTreeAll();

    /**
     * 根据用户ID查询菜单
     * @param userId 用户ID
     * @return 菜单列表
     */
    List<Menu> selectMenuTreeByUserId(Long userId);
}
```

3. MenuMapper  
不在Mapper里做嵌套查询，只做关联查询，查出相关的菜单。注：这个时候查出来的路由菜单，限制了是目录和菜单。
``` xml
<mapper namespace="com.bupt.web.dao.MenuDao">

    <select id="selectMenuTreeAll" resultType="Menu">
        select distinct m.id, m.p_id, m.`name`, m.router, m.path, m.component, m.invisible, m.is_deleted, ifnull(m.permission,'') as permission, m.type, m.icon, m.order_num, m.created_at, m.redirect
        from menu m where m.type in ('M', 'C') and m.is_deleted = 0
        order by m.p_id, m.order_num
    </select>


    <select id="selectMenuTreeByUserId" parameterType="Long" resultType="Menu">
        select distinct m.id, m.p_id, m.`name`, m.router,m.path, m.component, m.invisible, m.is_deleted, ifnull(m.permission,'') as permission, m.type, m.icon, m.order_num, m.created_at, m.redirect
        from menu m
            left join role_menu rm on m.id = rm.menu_id
            left join user_role ur on rm.role_id = ur.role_id
            left join role ro on ur.role_id = ro.id
            left join `user` u on ur.user_id = u.id
        where u.id = #{userId} and  m.type in ('M', 'C') and m.is_deleted = 0 and ro.is_deleted = 0
        order by m.p_id, m.order_num
    </select>
</mapper>
```

4. MenuService  
第一段代码，查询菜单列表。逻辑比较简单，先根据用户Id查一下当前用户拥有的角色。如果拥有超级管理员的角色，则拥有所有菜单权限，如果不是超级管理员，就对着表查。  

``` java
public List<MenuVo> selectMenuTreeByUserId(Long userId) {
//         首先判断用户是不是超级管理员，如果是超级管理员，则显示所有的菜单，如果不是超级管理员，则根据用户id筛选菜单
    List<User> users = userDao.selectUserById(userId);
    List<Menu> menus;
    if (!users.isEmpty() && users.get(0).getRoleIds().contains(1L)) { // 如果是超级管理员
        menus = menuDao.selectMenuTreeAll();
    } else {
        menus = menuDao.selectMenuTreeByUserId(userId);
    }
    // 返回Tree结构的数据
    List<Menu> menuTree = new ArrayList<>();
    getMenuTree(menuTree, menus, 0L);
    return menuTree.stream().map(MenuVo::new).collect(Collectors.toList());
}
```

返回树状结构的代码，核心函数是 `getMenuTree` 。它是一个递归函数，其中包括了三个参数，第一个参数 `menuTree` 用来存放最终构建的菜单树，第二个参数 `menuList` 用来存放所有的菜单列表，第三个参数 `pId` 用来存放当前这轮递归的菜单父节点。  
基本逻辑是：对菜单列表中的所有菜单那进行遍历，筛选出父节点是Pid的菜单（也就是当前父节点的子节点），代码是 `menuList.stream().filter(menu -> menu.getPId().equals(pId))`。然后遍历这群字节点，然后把子节点挂在在当前节点数上，继续寻找这批子节点的子节点。   
下面放一下这个函数：
``` java
/**
    * 递归遍历菜单，构建菜单树
    * @param menuTree 构建的菜单树
    * @param menuList 菜单列表
    * @param pId 父id
    */
public void getMenuTree(List<Menu> menuTree, List<Menu> menuList, Long pId) {
    // 获取当前pid的子菜单
    menuList.stream().filter(menu -> menu.getPId().equals(pId))
        .forEach(menu -> { // 构建结果树 & 递归遍历子菜单
            menuTree.add(menu);
            getMenuTree(menu.getChildren(), menuList, menu.getId());
        });
}
```
5. MenuController  
其中 `BaseController.getUserId();` 封装了获取当前用户Id的方法。userId可以由参数传入。
``` java
@GetMapping("/router")
public ResponseData getMenuByUserId(){
    Long userId = BaseController.getUserId();
    List<MenuVo> menus = menuService.selectMenuTreeByUserId(userId);
    return ResponseData.success(menus);
}
```

## 前端
### 异步路由设置
之前说了，antd的异步路由，需要预先在 `router.map.js` 中把组件都注册好，这点感觉，不太方便。ruoyi就不需要预先注册，所以就去ruoyi那里看了下源代码，把antd的代码修改了以下。在获取路由菜单的接口里获取组件路径，并注册。主要修改的文件为 `/utils/routerUtils.js` 。  
修改的函数是 `parseRoutes` 。其中，有一行，获取路由菜单，是直接从routerMap里获取的，这里要修改下逻辑，修改成，如果routerMap里拿不到，就自己注册路由。代码如下（主要看10-13行）：
``` js
function parseRoutes(routesConfig, routerMap) {
  let routes = []
  routesConfig.forEach(item => {
    // 获取注册在 routerMap 中的 router，初始化 routeCfg
    let router = undefined, routeCfg = {}
    if (typeof item === 'string') {
      router = routerMap[item]
      routeCfg = {path: (router && router.path) || item, router: item}
    } else if (typeof item === 'object') {
      router = routerMap[item.router] || {
        name: item.name,
        component: loadView(item.component)
      }
      routeCfg = item
    }
    if (!router) {
      console.warn(`can't find register for router ${routeCfg.router}, please register it in advance.`)
      router = typeof item === 'string' ? {path: item, name: item} : item
    }
    // 从 router 和 routeCfg 解析路由
    const meta = {
      authority: router.authority,
      icon: router.icon,
      page: router.page,
      link: router.link,
      params: router.params,
      query: router.query,
      ...router.meta
    }
    const cfgMeta = {
      authority: routeCfg.authority,
      icon: routeCfg.icon,
      page: routeCfg.page,
      link: routeCfg.link,
      params: routeCfg.params,
      query: routeCfg.query,
      ...routeCfg.meta
    }
    Object.keys(cfgMeta).forEach(key => {
      if (cfgMeta[key] === undefined || cfgMeta[key] === null || cfgMeta[key] === '') {
        delete cfgMeta[key]
      }
    })
    Object.assign(meta, cfgMeta)
    const route = {
      path: routeCfg.path || router.path || routeCfg.router,
      name: routeCfg.name || router.name,
      component: router.component,
      redirect: routeCfg.redirect || router.redirect,
      meta: {...meta, authority: meta.authority || '*'}
    }
    if (routeCfg.invisible || router.invisible) {
      route.meta.invisible = true
    }
    if (routeCfg.children && routeCfg.children.length > 0) {
      route.children = parseRoutes(routeCfg.children, routerMap)
    }
    routes.push(route)
  })
  return routes
}
```
`loadView` 方法如下：  
``` js
export const loadView = (view) => {
  if (process.env.NODE_ENV === 'development') {
    return (resolve) => require([`@/${view}`], resolve)
  } else {
    // 使用 import 实现生产环境的路由懒加载
    return () => import(`@/${view}`)
  }
}
```
这里，有一个小坑，就是这里import后面要加字符串（ `@/` ），不能直接加路径（ `@` 报错），要不然会报错。

其他异步设置和之前一样，就写一个异步请求，然后loadRouters就行。
``` js
getUserMenu().then(res => {
    let routesConfig = [{path: '/', redirect: '/home', name: '首页', children: res.data.data, component: 'layouts/tabs' }]
    loadRoutes(routesConfig)
    this.$router.push('/')
    this.$message.success("欢迎回来！", 3)
}).catch(res=>{
    console.log(res)
})
```

### 角色和权限设置
角色和权限设置这里也有小区别。第一个，之前路由里定义了角色权限，现在路由里不包含角色权限了。第二个，之前判断权限的指令是 `v-auth:role` 。这个指令是根据路由里的角色判断是否拥有权限的，我们这里就要修改，根据用户的角色判断是否拥有权限。这里修改的文件是 `/plugins/authority-plugin.js` 。  
进行权限校验的时候，先调 `$auth(check, type)` 这个方法，这个方法拿到了页面和用户的权限，再进 `auth.apply(this, [{check, type}, permission, role, permissions, roles])` 进行进一步判断。进 `auth.apply` 里继续看，执行的是下面这段代码，因为指定是role的，执行的是第8行。这里，判断一下 `role` 和 `roles` 这两个参数。`role` 是当前页面拥有的权限，`roles` 是当前用户拥有的权限，根据我们的需求，这里应该穿的是 `roles` (改之前是 `role` )。
``` js
const auth = function(authConfig, permission, role, permissions, roles) {
  const {check, type} = authConfig
  if (check && typeof check === 'function') {
    return check.apply(this, [permission, role, permissions, roles])
  }
  if (type === 'permission') {
    return checkFromPermission(check, permission)
  } else if (type === 'role') {
    return checkFromRoles(check, roles)
  } else {
    return checkFromPermission(check, permission) || checkFromRoles(check, role)
  }
}
```
进 `checkFromRoles(check, roles)` 这个方法继续看，这里，我们拿到拥有的角色和操作权限，需要进行的操作就是，把当前传入的字段和角色拥有的权限字段进行对比，如果相同则拥有这个权限。这里有一个特例，就是超级管理员拥有的操作权限是 `*:*:*`。 如果拥有这个字符，也拥有权限。代码入下：
``` js
/**
 * 检查 roles 是否有操作权限
 * @param check 需要检查的操作权限
 * @param roles 角色数组
 * @returns {boolean}
 */
const checkFromRoles = function(check, roles) {
  if (!roles) {
    return false
  }
  for (let role of roles) {
    const {operation} = role
    if (operation && (operation.indexOf("*:*:*") !== -1 ||  operation.indexOf(check) !== -1)) {
      return true
    }
  }
  return false
}
```

## 结语
通过以上方法，重新实现了权限管理操作。在数据库里修改数据，就可以实现用户的权限设置。接下来做用户、角色、菜单的CURD操作（增删改查）。