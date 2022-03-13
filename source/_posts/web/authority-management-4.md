---
title: 权限管理（四）
date: 2022-03-13 17:11:37
categories:
- web
tags:
- web
- 权限管理
- vue antd admin
- ruoyi
---

# 角色管理
主要功能为：查看角色列表、加载角色对应角色菜单列表树、注册角色、修改角色、删除角色
## 后端实现
### 查看角色列表
1. RoleController  
``` java
@PostMapping("/list")
public ResponseData list(@RequestBody(required = false) Role role,
                            @ApiParam("页数") @RequestParam(defaultValue = "1", value = "pageNum") Integer pageNum,
                            @ApiParam("每页的数据数量") @RequestParam(defaultValue = "10", value = "pageSize") Integer pageSize){
    PageData<Role> rolePageData = roleService.selectRoleList(new Page<>(pageNum, pageSize), role);
    return ResponseData.success(rolePageData);
}
```
2. RoleService  
这里查询数据，用的是mybatis plus。分页，也用的是mybatis plus（需要配置，开启分页）
``` java
public PageData<Role> selectRoleList(Page<Role> page, Role role) {
    QueryWrapper<Role> roleQueryWrapper = new QueryWrapper<>();
    if (role != null) {
        roleQueryWrapper.eq(role.getId() != null,"id",role.getId());
        roleQueryWrapper.eq(role.getKey() != null,"`key`",role.getKey());
        roleQueryWrapper.eq(role.getName() != null,"name",role.getName());
        roleQueryWrapper.eq(role.getIsDeleted() != null,"is_deleted",role.getIsDeleted());
    }
    Page<Role> rolePage = roleDao.selectPage(page, roleQueryWrapper);
    return new PageData<>(rolePage);
}
```

### 加载角色对应角色菜单列表树
1. MenuController   
这个接口，主要是查询当前角色拥有的菜单权限。在这里的操作有两个，第一个查询出当前角色拥有的菜单Id，第二个，查询当前角色拥有的菜单列表并转换成树格式。
``` java
@GetMapping(value = "/treeselect/{roleId}")
public ResponseData roleMenuTreeselect(@PathVariable("roleId") Long roleId) {
    Long userId = BaseController.getUserId();
    // 查询当前用户能看到的菜单列表
    List<Menu> menus = menuService.selectMenuList(userId);
    HashMap<String, Object> data = new HashMap<>();
    // 查询 当前角色拥有的 菜单Id
    Set<Long> menuIds = menuService.selectMenuListByRoleId(roleId);
    data.put("checkedKeys",menuIds);
    // 将菜单转为树结构
    List<TreeSelect> treeSelects = menuService.buildMenuTreeSelect(menus);
    data.put("menus",treeSelects);
    return ResponseData.success(data);
}
```
2. MenuService  
查询出当前角色拥有的菜单Id  
这里有一个逻辑操作，如果是超级管理员，能查到所有菜单Id，不是则根据角色菜单表查。
``` java
public Set<Long> selectMenuListByRoleId(Long roleId) {
    Set<Long> menuIds;
    Role role = roleDao.selectById(roleId);
    if (roleId.equals(1L)) {
        menuIds = menuDao.selectMenuListByRoleId(null);
    } else {
        menuIds = menuDao.selectMenuListByRoleId(roleId);
    }
    return menuIds;
}
```

查询当前角色拥有的菜单列表  
这里Menu是筛选参数，做条件查询的时候用的。
``` java
public List<Menu> selectMenuList(Menu menu, Long userId) {
    List<User> users = userDao.selectUserById(userId);
    List<Menu> menus;
    if (users.get(0).getRoleIds().contains(1L)) { // 如果是超级管理员
        QueryWrapper<Menu> menuQueryWrapper = new QueryWrapper<>();
        if (menu != null) {
            menuQueryWrapper.eq(menu.getId() != null,"id",menu.getId());
            menuQueryWrapper.eq(menu.getInvisible() != null,"invisible",menu.getInvisible());
            menuQueryWrapper.eq(menu.getName() != null,"name",menu.getName());
            menuQueryWrapper.eq(menu.getIsDeleted() != null,"is_deleted",menu.getIsDeleted());
        }
        menus = menuDao.selectList(menuQueryWrapper);
    } else {
        menus = menuDao.selectMenuListByUserId(menu, userId);
    }
    return menus;
}
```

构建菜单树  
这里的逻辑和之前差不多，看注释。
``` java
public List<Menu> buildMenuTree(List<Menu> menus) {
    // 找出 menus 中的 root Id
    List<Long> menuIds = menus.stream().map(Menu::getId).collect(Collectors.toList());
    List<Menu> menuTree = new ArrayList<>();
    for (Menu menu : menus) {
        // 如果，当前节点的父节点，不在menuIds中，说明其是根节点的第一代子节点
        if (! menuIds.contains(menu.getPId())) {
            // 给menu获取后辈
            getMenuTree(menu.getChildren(), menus, menu.getId()); // 把当前节点，加到menuTree中
            menuTree.add(menu); // 说明其是根节点的子节点
        }
    }
    // 返回Tree结构的数据
    if (menuTree.isEmpty()) {
        menuTree = menus;
    }
    return menuTree;
}
```

3. MenuMapper   
查询出当前角色拥有的菜单Id
``` xml
 <select id="selectMenuListByRoleId" resultType="Long">
    select distinct m.id
    from menu m
        left join role_menu rm on m.id = rm.menu_id
        <where>
            <if test="roleId != null">
                rm.role_id = #{roleId}
            </if>
        </where>
</select>
```

查询当前角色拥有的菜单列表  
``` xml
<select id="selectMenuListByUserId" resultType="Menu">
    select distinct m.id, m.p_id, m.`name`, m.path, m.component, m.invisible, m.is_deleted, ifnull(m.permission,'') as permission, m.type, m.icon, m.order_num, m.created_at, m.redirect
    from menu m
    left join role_menu rm on m.id = rm.menu_id
    left join user_role ur on rm.role_id = ur.role_id
    left join role ro on ur.role_id = ro.id
    where ur.user_id = #{userId}
    <if test="menu.name != null and menu.name != ''">
        AND m.name like concat('%', #{menu.name}, '%')
    </if>
    <if test="menu.invisible != null and menu.invisible != ''">
        AND m.invisible = #{menu.invisible}
    </if>
    <if test="menu.isDeleted != null and menu.isDeleted != ''">
        AND m.is_deleted = #{menu.isDeleted}
    </if>
    order by m.p_id, m.order_num
</select>
```
### 添加角色
1. RoleController  
``` java
@PostMapping
public ResponseData add(@Validated @RequestBody Role role) {
    if (role.getName() == null || role.getKey() == null){
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    if (!roleService.checkRoleNameUnique(role.getName())) {
        return ResponseData.setResult(ResultCodeEnum.ROLE_NAME_HAS_EXIST);
    }
    if (!roleService.checkRoleKeyUnique(role.getKey())) {
        return ResponseData.setResult(ResultCodeEnum.ROLE_KEY_HAS_EXIST);
    }
    // 设置被创建者
    role.setCreatedBy(BaseController.getUserName());
    int result = roleService.insertRole(role);
    if (result > 0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_ADD_FAILURE);
    }
}
```
2. RoleService  
有一个特别的，就是添加角色的时候，也会添加菜单。
``` java
public int insertRole(Role role) {
    // 添加角色
    int insert = roleDao.insert(role);
    // 添加菜单
    insertRoleMenu(role);
    return insert;
}

public int insertRoleMenu(Role role) {
    // 新增用户与角色管理
    if (role.getMenuIds() != null && role.getId() != null) {
        role.getMenuIds().forEach(menuId -> {
            RoleMenu roleMenu = new RoleMenu();
            roleMenu.setMenuId(menuId);
            roleMenu.setRoleId(role.getId());
            roleMenuDao.insert(roleMenu);
        });
    }
    return role.getMenuIds().size();
}
```

### 修改角色
1. RoleController  
``` java
@PutMapping()
public ResponseData update(@RequestBody Role role) {
    if (role.getId() == null) {
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    // 修改角色
    role.setUpdatedBy(BaseController.getUserName());
    int result = roleService.updateRole(role);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_UPDATE_ERROR);
    }
}
```

2. RoleService  
这里，修改角色时，如果传了菜单Id，则先删除角色关联的菜单Id，再新添加
``` java
public int updateRole(Role role) {
    // 修改角色信息
    int insert = 0;
    if (role.getName() != null || role.getKey() != null || role.getIsDeleted() != null) {
        insert = roleDao.updateById(role);
    }
    if (role.getMenuIds() != null) {
        // 删除角色与菜单关联
        insert = deleteRoleMenuByRoleId(role.getId());
        // 重新添加角色菜单权限
        insert = insertRoleMenu(role);
    }
    return insert;
}

public int deleteRoleMenuByRoleId(Long roleId) {
    QueryWrapper<RoleMenu> roleMenuQueryWrapper = new QueryWrapper<>();
    roleMenuQueryWrapper.eq("role_id",roleId);
    return roleMenuDao.delete(roleMenuQueryWrapper);
}
```

### 删除角色
1. RoleController  
``` java
@DeleteMapping("/{roleId}")
public ResponseData delete(@PathVariable("roleId") Long roleId) {
    if (roleId == null) {
        return ResponseData.setResult(ResultCodeEnum.PARA_FORMAT_ERROR);
    }
    int result = roleService.deleteRoleById(roleId);
    if (result>0) {
        return ResponseData.success();
    } else {
        return ResponseData.setResult(ResultCodeEnum.DB_DELETE_FAILURE);
    }

}
```

2. RoleService  
删除角色，会删除相关联的菜单Id
``` java
public int deleteRoleById(Long roleId) {
    // 删除角色相关菜单
    int i = deleteRoleMenuByRoleId(roleId);
    // 删除角色
    return roleDao.deleteRoleById(roleId);
}
```

## 前端
![20220313172824.png](https://s2.loli.net/2022/03/13/dhlzUOaMANFV6ev.png)
![20220313172919.png](https://s2.loli.net/2022/03/13/rRzmyeGYcpJIFTB.png)