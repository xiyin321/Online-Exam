<template>
  <div :class="{ hasTagsView: needTagsView }" class="main-content">
    <transition :enter-active-class="proxy?.animate.searchAnimate.enter"
                :leave-active-class="proxy?.animate.searchAnimate.leave">
      <div v-show="showSearch" ref="searchContainer" class="mb-[10px]">
        <el-card shadow="hover">
          <el-form ref="queryFormRef" :inline="true" :model="queryParams">
            <el-form-item label="角色名称" prop="roleName">
              <el-input v-model="queryParams.roleName" clearable placeholder="请输入角色名称" style="width: 240px"
                        @blur="handleQuery" @keyup.enter="handleQuery"/>
            </el-form-item>
            <el-form-item label="权限字符" prop="roleKey">
              <el-input v-model="queryParams.roleKey" clearable placeholder="请输入权限字符" style="width: 240px"
                        @blur="handleQuery" @keyup.enter="handleQuery"/>
            </el-form-item>
            <el-form-item label="状态" prop="status">
              <el-select v-model="queryParams.status" clearable placeholder="角色状态" style="width: 240px"
                         @change="handleQuery">
                <el-option v-for="dict in sys_normal_disable" :key="dict.value" :label="dict.label" :value="dict.value">
                  <span style="float: left">{{ dict.label }}</span>
                  <span style="float: right;color: var(--el-text-color-secondary);font-size: 13px;">
                    {{ dict.value }}
                  </span>
                </el-option>
              </el-select>
            </el-form-item>
            <el-form-item label="创建时间" style="width: 308px">
              <el-date-picker
                v-model="dateRange"
                :default-time="[new Date(2000, 1, 1, 0, 0, 0), new Date(2000, 1, 1, 23, 59, 59)]"
                end-placeholder="结束日期"
                range-separator="-"
                start-placeholder="开始日期"
                type="daterange"
                value-format="YYYY-MM-DD"
                @change="handleQuery"
              ></el-date-picker>
            </el-form-item>
          </el-form>
        </el-card>
      </div>
    </transition>

    <el-card class="card-content" shadow="hover">
      <template #header>
        <el-row :gutter="10">
          <el-col :span="1.5">
            <el-button v-hasPermi="['system:role:add']" icon="Plus" plain type="primary" @click="handleAdd()">新增
            </el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['system:role:edit']" :disabled="single" icon="Edit" plain type="success"
                       @click="handleUpdate()">修改
            </el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['system:role:delete']" :disabled="ids.length === 0" plain type="danger"
                       @click="handleDelete()">删除
            </el-button>
          </el-col>
          <el-col :span="1.5">
            <el-button v-hasPermi="['system:role:export']" icon="Download" plain type="warning" @click="handleExport">
              导出
            </el-button>
          </el-col>
          <right-toolbar v-model:showSearch="showSearch" @queryTable="getList" @resetQuery="resetQuery"></right-toolbar>
        </el-row>
      </template>

      <el-table ref="roleTableRef" v-loading="loading" :data="roleList" :height="tableHeight"
                @selection-change="handleSelectionChange">
        <el-table-column align="center" type="selection" width="55"/>
        <el-table-column v-if="false" align="center" label="角色编号" prop="roleId" width="120"/>
        <el-table-column align="center" label="角色" prop="roleId" width="180">
          <template #default="scope">
            <dict-tag :options="sys_user_role_ids" :value="scope.row.roleId"/>
          </template>
        </el-table-column>
        <el-table-column :show-overflow-tooltip="true" align="center" label="角色名称" prop="roleName" width="150"/>
        <el-table-column :show-overflow-tooltip="true" align="center" label="权限字符" prop="roleKey" width="200"/>
        <el-table-column align="center" label="显示顺序" prop="roleSort" width="100"/>
        <el-table-column align="center" label="状态" width="100">
          <template #default="scope">
            <el-switch v-model="scope.row.status" active-value="0" inactive-value="1"
                       @change="handleStatusChange(scope.row)"></el-switch>
          </template>
        </el-table-column>
        <el-table-column align="center" label="创建时间" prop="createTime">
          <template #default="scope">
            <span>{{ parseTime(scope.row.createTime) }}</span>
          </template>
        </el-table-column>

        <el-table-column align="center" fixed="right" label="操作" width="230">
          <template #default="scope">
            <el-tooltip v-if="scope.row.roleId !== 1" content="修改" placement="top">
              <el-button v-hasPermi="['system:role:edit']" circle icon="Edit" plain type="primary"
                         @click="handleUpdate(scope.row)"></el-button>
            </el-tooltip>
            <el-tooltip v-if="scope.row.roleId !== 1" content="删除" placement="top">
              <el-button v-hasPermi="['system:role:remove']" circle icon="Delete" plain type="danger"
                         @click="handleDelete(scope.row)"></el-button>
            </el-tooltip>
            <el-tooltip v-if="scope.row.roleId !== 1" content="数据权限" placement="top">
              <el-button v-hasPermi="['system:role:edit']" circle icon="CircleCheck" plain type="success"
                         @click="handleDataScope(scope.row)"></el-button>
            </el-tooltip>
            <el-tooltip v-if="scope.row.roleId !== 1" content="分配用户" placement="top">
              <el-button v-hasPermi="['system:role:edit']" circle icon="User" plain type="info"
                         @click="handleAuthUser(scope.row)"></el-button>
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
    <div class="page-content">
      <pagination
        v-show="total>0"
        v-model:limit="queryParams.pageSize"
        v-model:page="queryParams.pageNum"
        :total="total"
        @pagination="getList"
      />
    </div>
    <el-drawer v-model="dialog.visible" :title="dialog.title" append-to-body size="30%">
      <el-form ref="roleFormRef" :model="form" :rules="rules" label-width="100px">
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="form.roleName" placeholder="请输入角色名称"/>
        </el-form-item>
        <el-form-item prop="roleKey">
          <template #label>
            <span>
              <el-tooltip content="控制器中定义的权限字符，如：@SaCheckRole('admin')" placement="top">
                <el-icon><question-filled/></el-icon>
              </el-tooltip>
              权限字符
            </span>
          </template>
          <el-input v-model="form.roleKey" placeholder="请输入权限字符"/>
        </el-form-item>
        <el-form-item label="角色顺序" prop="roleSort">
          <el-input-number v-model="form.roleSort" :min="0" controls-position="right"/>
        </el-form-item>
        <el-form-item label="状态">
          <el-radio-group v-model="form.status">
            <el-radio v-for="dict in sys_normal_disable" :key="dict.value" :label="dict.value">{{
                dict.label
              }}
            </el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="菜单权限">
          <el-checkbox v-model="menuExpand" @change="handleCheckedTreeExpand($event, 'menu')">展开/折叠</el-checkbox>
          <el-checkbox v-model="menuNodeAll" @change="handleCheckedTreeNodeAll($event, 'menu')">全选/全不选
          </el-checkbox>
          <el-checkbox v-model="form.menuCheckStrictly" @change="handleCheckedTreeConnect($event, 'menu')">父子联动
          </el-checkbox>
          <el-tree
            ref="menuRef"
            :check-strictly="!form.menuCheckStrictly"
            :data="menuOptions"
            :props="{ label: 'label', children: 'children' }"
            class="tree-border"
            empty-text="加载中，请稍候"
            node-key="id"
            show-checkbox
          ></el-tree>
        </el-form-item>
        <el-form-item label="备注">
          <el-input v-model="form.remark" placeholder="请输入内容" type="textarea"></el-input>
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-drawer>

    <!-- 分配角色数据权限对话框 -->
    <el-drawer v-model="openDataScope" :title="dialog.title" append-to-body size="30%">
      <el-form ref="dataScopeRef" :model="form" label-width="25%">
        <el-form-item label="角色名称">
          <el-input v-model="form.roleName" :disabled="true"/>
        </el-form-item>
        <el-form-item label="权限字符">
          <el-input v-model="form.roleKey" :disabled="true"/>
        </el-form-item>
        <el-form-item label="权限范围">
          <el-select v-model="form.dataScope" @change="dataScopeSelectChange">
            <el-option v-for="item in dataScopeOptions" :key="item.value" :label="item.label"
                       :value="item.value"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item v-show="form.dataScope === '2'" label="数据权限">
          <el-checkbox v-model="deptExpand" @change="handleCheckedTreeExpand($event, 'dept')">展开/折叠</el-checkbox>
          <el-checkbox v-model="deptNodeAll" @change="handleCheckedTreeNodeAll($event, 'dept')">全选/全不选
          </el-checkbox>
          <el-checkbox v-model="form.deptCheckStrictly" @change="handleCheckedTreeConnect($event, 'dept')">父子联动
          </el-checkbox>
          <el-tree
            ref="deptRef"
            :check-strictly="!form.deptCheckStrictly"
            :data="deptOptions"
            :props="{ label: 'label', children: 'children' }"
            class="tree-border"
            default-expand-all
            empty-text="加载中，请稍候"
            node-key="id"
            show-checkbox
          ></el-tree>
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitDataScope">确 定</el-button>
          <el-button @click="cancelDataScope">取 消</el-button>
        </div>
      </template>
    </el-drawer>
  </div>
</template>

<script lang="ts" name="Role" setup>
import {
  addRole,
  changeRoleStatus,
  dataScope,
  delRole,
  deptTreeSelect,
  getRole,
  listRole,
  updateRole
} from "@/api/system/role";
import {roleMenuTreeselect, treeselect as menuTreeselect} from '@/api/system/menu/index';
import {DeptTreeOption, RoleForm, RoleQuery, RoleVO} from '@/api/system/role/types';
import {MenuTreeOption, RoleMenuTree} from '@/api/system/menu/types';
import useSettingsStore from "@/store/modules/settings";

const settingsStore = useSettingsStore()
const needTagsView = computed(() => settingsStore.tagsView);
const router = useRouter();
const {proxy} = getCurrentInstance() as ComponentInternalInstance;
const {sys_normal_disable, sys_user_role_ids} = toRefs<any>(proxy?.useDict('sys_normal_disable', 'sys_user_role_ids'));

const roleList = ref<RoleVO[]>();
const loading = ref(true)
const showSearch = ref(true)
const ids = ref<Array<string | number>>([])
const single = ref(true)
const multiple = ref(true)
const total = ref(0)
const dateRange = ref<[DateModelType, DateModelType]>(['', ''])
const menuOptions = ref<MenuTreeOption[]>([])
const menuExpand = ref(false)
const menuNodeAll = ref(false)
const deptExpand = ref(true)
const deptNodeAll = ref(false)
const deptOptions = ref<DeptTreeOption[]>([])
const openDataScope = ref(false)

/** 数据范围选项*/
const dataScopeOptions = ref([
  {value: "1", label: "全部数据权限"},
  {value: "2", label: "自定数据权限"},
  {value: "3", label: "本部门数据权限"},
  {value: "4", label: "本部门及以下数据权限"},
  {value: "5", label: "仅本人数据权限"}
])

const queryFormRef = ref<ElFormInstance>();
const roleFormRef = ref<ElFormInstance>();
const dataScopeRef = ref<ElFormInstance>();
const menuRef = ref<ElTreeInstance>();
const deptRef = ref<ElTreeInstance>();
/*用于计算搜索栏高度*/
const searchContainer = ref<HTMLElement | null>(null)
const initForm: RoleForm = {
  roleId: undefined,
  roleSort: 1,
  status: '0',
  roleName: '',
  roleKey: '',
  menuCheckStrictly: true,
  deptCheckStrictly: true,
  remark: '',
  dataScope: '1',
  menuIds: [],
  deptIds: [],
}

const data = reactive<PageData<RoleForm, RoleQuery>>({
  form: {...initForm},
  queryParams: {
    pageNum: 1,
    pageSize: 20,
    roleName: '',
    roleKey: '',
    status: '',
  },
  rules: {
    roleName: [{required: true, message: "角色名称不能为空", trigger: "blur"}],
    roleKey: [{required: true, message: "权限字符不能为空", trigger: "blur"}],
    roleSort: [{required: true, message: "角色顺序不能为空", trigger: "blur"}]
  }
})
const {form, queryParams, rules} = toRefs(data)

const dialog = reactive<DialogOption>({
  visible: false,
  title: ''
});

/**
 * 查询角色列表
 */
const getList = () => {
  loading.value = true
  listRole(proxy?.addDateRange(queryParams.value, dateRange.value)).then(res => {
    roleList.value = res.rows
    total.value = res.total
    loading.value = false
  })
}

/**
 * 搜索按钮操作
 */
const handleQuery = () => {
  queryParams.value.pageNum = 1;
  getList();
}

/** 重置 */
const resetQuery = () => {
  dateRange.value = ['', '']
  queryFormRef.value?.resetFields();
  handleQuery();
}
/**删除按钮操作 */
const handleDelete = async (row?: RoleVO) => {
  const roleids = row?.roleId || ids.value;
  await proxy?.$modal.confirm('是否确认删除角色编号为' + roleids + '数据项目');
  await delRole(roleids);
  getList();
  proxy?.$modal.msgSuccess('删除成功');
}

/** 导出按钮操作 */
const handleExport = () => {
  proxy?.download("system/role/export", {
    ...queryParams.value,
  }, `role_${new Date().getTime()}.xlsx`)
}
/** 多选框选中数据 */
const handleSelectionChange = (selection: RoleVO[]) => {
  ids.value = selection.map((item: RoleVO) => item.roleId);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}

/** 角色状态修改 */
const handleStatusChange = async (row: RoleVO) => {
  let text = row.status === "0" ? "启用" : "停用";
  try {
    await proxy?.$modal.confirm('确认要"' + text + '""' + row.roleName + '"角色吗?');
    await changeRoleStatus(row.roleId, row.status);
    proxy?.$modal.msgSuccess(text + "成功");
  } catch {
    row.status = row.status === "0" ? "1" : "0";
  }
}

/** 分配用户 */
const handleAuthUser = (row: RoleVO) => {
  router.push("/system/role-auth/user/" + row.roleId);
}

/** 查询菜单树结构 */
const getMenuTreeselect = async () => {
  const res = await menuTreeselect();
  menuOptions.value = res.data;
}
/** 所有部门节点数据 */
const getDeptAllCheckedKeys = (): any => {
  // 目前被选中的部门节点
  let checkedKeys = deptRef.value?.getCheckedKeys();
  // 半选中的部门节点
  let halfCheckedKeys = deptRef.value?.getHalfCheckedKeys();
  if (halfCheckedKeys) {
    checkedKeys?.unshift.apply(checkedKeys, halfCheckedKeys);
  }
  return checkedKeys
}
/** 重置新增的表单以及其他数据  */
const reset = () => {
  menuRef.value?.setCheckedKeys([]);
  menuExpand.value = false
  menuNodeAll.value = false
  deptExpand.value = true
  deptNodeAll.value = false
  form.value = {...initForm};
  roleFormRef.value?.resetFields();

}

/** 添加角色 */
const handleAdd = () => {
  reset();
  getMenuTreeselect();
  dialog.visible = true;
  dialog.title = "添加角色";
}
/** 修改角色 */
const handleUpdate = async (row?: RoleVO) => {
  reset();
  const roleId = row?.roleId || ids.value[0]
  const {data} = await getRole(roleId);
  Object.assign(form.value, data);
  form.value.roleSort = Number(form.value.roleSort);
  const res = await getRoleMenuTreeselect(roleId);
  dialog.title = "修改角色";
  dialog.visible = true;
  res.checkedKeys.forEach((v) => {
    nextTick(() => {
      menuRef.value?.setChecked(v, true, false);
    })
  })

}
/** 根据角色ID查询菜单树结构 */
const getRoleMenuTreeselect = (roleId: string | number) => {
  return roleMenuTreeselect(roleId).then((res): RoleMenuTree => {
    menuOptions.value = res.data.menus;
    return res.data;
  })
}
/** 根据角色ID查询部门树结构 */
const getRoleDeptTreeSelect = async (roleId: string | number) => {
  const res = await deptTreeSelect(roleId);
  deptOptions.value = res.data.depts;
  return res.data;
}
/** 树权限（展开/折叠）*/
const handleCheckedTreeExpand = (value: boolean, type: string) => {
  if (type == "menu") {
    let treeList = menuOptions.value;
    for (let i = 0; i < treeList.length; i++) {
      if (menuRef.value) {
        menuRef.value.store.nodesMap[treeList[i].id].expanded = value;
      }
    }
  } else if (type == "dept") {
    let treeList = deptOptions.value;
    for (let i = 0; i < treeList.length; i++) {
      if (deptRef.value) {
        deptRef.value.store.nodesMap[treeList[i].id].expanded = value;
      }
    }
  }
}
/** 树权限（全选/全不选） */
const handleCheckedTreeNodeAll = (value: any, type: string) => {
  if (type == "menu") {
    menuRef.value?.setCheckedNodes(value ? menuOptions.value as any : []);
  } else if (type == "dept") {
    deptRef.value?.setCheckedNodes(value ? deptOptions.value as any : []);
  }
}
/** 树权限（父子联动） */
const handleCheckedTreeConnect = (value: any, type: string) => {
  if (type == "menu") {
    form.value.menuCheckStrictly = value;
  } else if (type == "dept") {
    form.value.deptCheckStrictly = value;
  }
}
/** 所有菜单节点数据 */
const getMenuAllCheckedKeys = (): any => {
  // 目前被选中的菜单节点
  let checkedKeys = menuRef.value?.getCheckedKeys();
  // 半选中的菜单节点
  let halfCheckedKeys = menuRef.value?.getHalfCheckedKeys();
  if (halfCheckedKeys) {
    checkedKeys?.unshift.apply(checkedKeys, halfCheckedKeys);
  }
  return checkedKeys;
}
/** 提交按钮 */
const submitForm = () => {
  roleFormRef.value?.validate(async (valid: boolean) => {
    if (valid) {
      form.value.menuIds = getMenuAllCheckedKeys()
      form.value.roleId ? await updateRole(form.value) : await addRole(form.value);
      proxy?.$modal.msgSuccess("操作成功")
      dialog.visible = false
      getList()
    }
  })
}
/** 取消按钮 */
const cancel = () => {
  reset()
  dialog.visible = false;
}
/** 选择角色权限范围触发 */
const dataScopeSelectChange = (value: string) => {
  if (value !== "2") {
    deptRef.value?.setCheckedKeys([])
  }
}
/** 分配数据权限操作 */
const handleDataScope = async (row: RoleVO) => {
  const response = await getRole(row.roleId);
  Object.assign(form.value, response.data);
  const res = await getRoleDeptTreeSelect(row.roleId);
  openDataScope.value = true;
  dialog.title = "分配数据权限";
  await nextTick(() => {
    deptRef.value?.setCheckedKeys(res.checkedKeys);
  })
}
/** 提交按钮（数据权限） */
const submitDataScope = async () => {
  if (form.value.roleId) {
    form.value.deptIds = getDeptAllCheckedKeys();
    await dataScope(form.value);
    proxy?.$modal.msgSuccess("修改成功");
    openDataScope.value = false;
    getList();
  }
}
/** 取消按钮（数据权限）*/
const cancelDataScope = () => {
  dataScopeRef.value?.resetFields();
  form.value = {...initForm};
  openDataScope.value = false;
}

/*定义窗口高度*/
const screenHeight = ref(window.innerHeight)
const tableHeight = ref<number | string>(0)
const height = ref(0);

function handleResize() {
  screenHeight.value = window.innerHeight;
}

const calculateTableHeight = () => {
  const searchHeight = showSearch.value ? ref(height.value) : ref(-12); // 搜索栏的高度
  const tagsViewHeight = needTagsView.value ? ref(34) : ref(0);
  tableHeight.value = screenHeight.value - 50 - tagsViewHeight.value - 26 - searchHeight.value - 170
}
watch(showSearch, () => {
  calculateTableHeight()
});
/*响应式监听*/
watchEffect(() => {
  calculateTableHeight()
});
/*移除事件监听器*/
onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

onMounted(() => {
  /*自适应表格高度*/
  if (searchContainer.value) {
    height.value = searchContainer.value.offsetHeight;
  }
  window.addEventListener('resize', handleResize)
  calculateTableHeight();
  getList();
});
</script>
<style lang="scss" scoped>

</style>
