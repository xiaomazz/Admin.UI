<template>
  <section style="padding:10px;">
    <!--查询-->
    <el-form :inline="true" :model="filters" @submit.native.prevent>
      <el-form-item>
        <el-input v-model="filters.label" placeholder="视图名或地址" clearable @keyup.enter.native="onGetList">
          <template #prefix>
            <i class="el-input__icon el-icon-search" />
          </template>
        </el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="onGetList">查询</el-button>
      </el-form-item>
      <el-form-item v-if="checkPermission(['api:admin:view:add'])">
        <el-button type="primary" @click="onAdd">新增</el-button>
      </el-form-item>
      <el-form-item v-if="checkPermission(['api:admin:view:sync'])">
        <my-confirm-button
          :icon="'el-icon-refresh'"
          :placement="'bottom-end'"
          :loading="syncLoading"
          style="margin:0px;"
          @click="onSync"
        >
          <template #content>
            <p>确定要同步视图吗？</p>
          </template>
          同步视图
        </my-confirm-button>
      </el-form-item>
      <el-form-item v-if="checkPermission(['api:admin:view:batchsoftdelete'])">
        <my-confirm-button
          :disabled="sels.length === 0"
          :type="'delete'"
          :placement="'bottom-end'"
          :loading="deleteLoading"
          style="margin-left: 0px;"
          @click="onBatchDelete"
        >
          <template #content>
            <p>确定要批量删除吗？</p>
          </template>
          批量删除
        </my-confirm-button>
      </el-form-item>
    </el-form>

    <!--列表-->
    <el-table
      ref="multipleTable"
      v-loading="listLoading"
      row-key="id"
      :data="viewTree"
      :default-expand-all="false"
      :expand-row-keys="expandRowKeys"
      :tree-props="{ children: 'children', hasChildren: 'hasChildren' }"
      highlight-current-row
      style="width: 100%;"
      @select-all="onSelectAll"
      @select="onSelect"
    >
      <el-table-column type="selection" width="50" />
      <el-table-column prop="label" label="视图名称" width="180" />
      <el-table-column prop="name" label="视图命名" width="180" />
      <el-table-column prop="path" label="视图地址" width />
      <el-table-column prop="description" label="视图描述" width />
      <el-table-column prop="enabled" label="状态" width="100">
        <template #default="{row}">
          <el-tag :type="row.enabled ? 'success' : 'info'" disable-transitions>
            {{ row.enabled ? '正常' : '禁用' }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column v-if="checkPermission(['api:admin:view:update','api:admin:view:softdelete'])" label="操作" width="180">
        <template #default="{ $index, row }">
          <el-button v-if="checkPermission(['api:admin:view:update'])" @click="onEdit($index, row)">编辑</el-button>
          <my-confirm-button v-if="checkPermission(['api:admin:view:softdelete'])" type="delete" :loading="row._loading" @click="onDelete($index, row)" />
        </template>
      </el-table-column>
    </el-table>

    <!--新增窗口-->
    <my-window
      v-if="checkPermission(['api:admin:view:add'])"
      title="新增"
      :visible.sync="addFormVisible"
      @close="onCloseAddForm"
    >
      <el-form ref="addForm" :model="addForm" label-width="80px" :rules="addFormRules">
        <el-form-item prop="parentIds" label="上级视图">
          <el-cascader
            :key="addFormKey"
            v-model="addForm.parentIds"
            placeholder="请选择，支持搜索功能"
            :options="modules"
            :props="{ checkStrictly: true, value: 'id' }"
            filterable
            style="width:100%;"
          />
        </el-form-item>
        <el-form-item label="视图名称" prop="label">
          <el-input v-model="addForm.label" auto-complete="off" />
        </el-form-item>
        <el-form-item label="视图命名" prop="name">
          <el-input v-model="addForm.name" auto-complete="off" />
        </el-form-item>
        <el-form-item label="视图地址" prop="path">
          <el-input v-model="addForm.path" auto-complete="off" />
        </el-form-item>
        <el-form-item label="启用" prop="enabled">
          <el-switch v-model="addForm.enabled" />
        </el-form-item>
        <el-form-item label="说明" prop="description">
          <el-input v-model="addForm.description" type="textarea" rows="2" auto-complete="off" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click.native="addFormVisible = false">取消</el-button>
        <my-confirm-button type="submit" :validate="addFormValidate" :loading="addLoading" @click="onAddSubmit" />
      </template>
    </my-window>

    <!--编辑窗口-->
    <my-window
      v-if="checkPermission(['api:admin:view:update'])"
      title="编辑"
      :visible.sync="editFormVisible"
      @close="onCloseEditForm"
    >
      <el-form ref="editForm" :model="editForm" :rules="editFormRules" label-width="80px">
        <el-form-item prop="parentIds" label="上级视图">
          <el-cascader
            :key="editFormKey"
            v-model="editForm.parentIds"
            placeholder="请选择，支持搜索功能"
            :options="modules"
            :props="{ checkStrictly: true, value: 'id' }"
            filterable
            style="width:100%;"
          />
        </el-form-item>
        <el-form-item label="视图名称" prop="label">
          <el-input v-model="editForm.label" auto-complete="off" />
        </el-form-item>
        <el-form-item label="视图命名" prop="name">
          <el-input v-model="editForm.name" auto-complete="off" />
        </el-form-item>
        <el-form-item label="视图地址" prop="path">
          <el-input v-model="editForm.path" auto-complete="off" />
        </el-form-item>
        <el-form-item label="启用" prop="enabled">
          <el-switch v-model="editForm.enabled" />
        </el-form-item>
        <el-form-item label="说明" prop="description">
          <el-input v-model="editForm.description" type="textarea" rows="2" auto-complete="off" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click.native="editFormVisible = false">取消</el-button>
        <my-confirm-button type="submit" :validate="editFormValidate" :loading="editLoading" @click="onEditSubmit" />
      </template>
    </my-window>
  </section>
</template>

<script>
import { formatTime, treeToList, listToTree, getTreeParents } from '@/utils'
import {
  removeView,
  editView,
  addView,
  syncView,
  getViewList,
  batchRemoveView,
  getView
} from '@/api/admin/view'
import MyWindow from '@/components/my-window'
import MyConfirmButton from '@/components/my-confirm-button'

export default {
  name: 'AdminView',
  components: {
    MyWindow,
    MyConfirmButton
  },
  data() {
    return {
      filters: {
        label: ''
      },
      viewTree: [],
      expandRowKeys: [],
      listLoading: false,
      sels: [], // 列表选中列

      addDialogFormVisible: false,
      editFormVisible: false, // 编辑界面是否显示
      editLoading: false,
      editFormRules: {
        parentIds: [{ required: true, message: '请选择上级视图', trigger: 'change' }],
        label: [{ required: true, message: '请输入视图名', trigger: 'blur' }]
      },
      // 编辑界面数据
      editForm: {
        id: 0,
        parentIds: [],
        label: '',
        name: '',
        path: '',
        enabled: false,
        description: ''
      },
      editFormKey: 1,

      addFormVisible: false, // 新增界面是否显示
      addLoading: false,
      addFormRules: {
        parentIds: [{ required: true, message: '请选择上级视图', trigger: 'change' }],
        label: [{ required: true, message: '请输入视图名', trigger: 'blur' }]
      },
      // 新增界面数据
      addForm: {
        parentIds: [],
        label: '',
        name: '',
        path: '',
        enabled: true,
        description: ''
      },
      addFormKey: 1,
      modules: [],
      syncLoading: false,
      deleteLoading: false
    }
  },
  mounted() {
    this.onGetList()
  },
  methods: {
    formatCreatedTime: function(row, column, time) {
      return formatTime(time, 'YYYY-MM-DD HH:mm')
    },
    // 获取列表
    async onGetList() {
      const para = {
        key: this.filters.label
      }
      this.listLoading = true
      const res = await getViewList(para)
      this.listLoading = false

      if (!res?.success) {
        return
      }

      const list = _.cloneDeep(res.data)

      const keys = list.filter(l => l.parentId === 0).map(l => l.id + '')
      this.expandRowKeys = keys

      this.modules = listToTree(_.cloneDeep(list), {
        id: 0,
        parentId: 0,
        label: '顶级'
      })

      list.forEach(l => {
        l._loading = false
      })

      const tree = listToTree(list)
      this.sels = []
      this.viewTree = tree
    },
    // 显示编辑界面
    async onEdit(index, row) {
      const loading = this.$loading()
      const res = await getView({ id: row.id })
      loading.close()
      if (res && res.success) {
        const parents = getTreeParents(this.viewTree, row.id)
        const parentIds = parents.map(p => p.id)
        parentIds.unshift(0)

        const data = res.data
        data.parentIds = parentIds
        this.editForm = data
        this.editFormVisible = true
        ++this.editFormKey
      }
    },
    onCloseEditForm() {
      this.$refs.editForm.resetFields()
      ++this.editFormKey
    },
    // 显示新增界面
    onAdd() {
      this.addFormVisible = true
    },
    onCloseAddForm() {
      this.$refs.addForm.resetFields()
      ++this.addFormKey
    },
    // 编辑
    editFormValidate: function() {
      let isValid = false
      this.$refs.editForm.validate(valid => {
        isValid = valid
      })
      return isValid
    },
    async onEditSubmit() {
      this.editLoading = true
      const para = _.cloneDeep(this.editForm)
      para.parentId = para.parentIds.pop()
      if (para.id === para.parentId) {
        this.$message({
          message: '上级视图不能是自己！',
          type: 'error'
        })
        this.editLoading = false
        return
      }

      const res = await editView(para)
      this.editLoading = false

      if (!res?.success) {
        return
      }

      this.$message({
        message: this.$t('admin.updateOk'),
        type: 'success'
      })
      this.$refs['editForm'].resetFields()
      this.editFormVisible = false
      this.onGetList()
    },
    // 新增
    addFormValidate: function() {
      let isValid = false
      this.$refs.addForm.validate(valid => {
        isValid = valid
      })
      return isValid
    },
    async onAddSubmit() {
      this.addLoading = true
      const para = _.cloneDeep(this.addForm)
      para.parentId = para.parentIds.pop()

      const res = await addView(para)
      this.addLoading = false

      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.addOk'),
        type: 'success'
      })
      this.$refs['addForm'].resetFields()
      this.addFormVisible = false
      this.onGetList()
    },
    // 删除
    async onDelete(index, row) {
      row._loading = true
      const para = { id: row.id }
      const res = await removeView(para)

      row._loading = false

      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.deleteOk'),
        type: 'success'
      })
      this.onGetList()
    },
    // 批量删除
    async onBatchDelete() {
      const para = { ids: [] }
      para.ids = this.sels.map(s => {
        return s.id
      })

      this.deleteLoading = true
      const res = await batchRemoveView(para.ids)
      this.deleteLoading = false

      if (!res?.success) {
        return
      }

      this.$message({
        message: this.$t('admin.batchDeleteOk'),
        type: 'success'
      })

      this.onGetList()
    },
    // 同步view
    async onSync() {
      const viewFiles = require.context('@/views', true, /\.vue$/)
      const views = viewFiles.keys().reduce((views, modulePath) => {
        const path = modulePath.replace(/^\.\/(.*)\.\w+$/, '$1')
        const view = viewFiles(modulePath)
        const name = view.default.name
        const syncInfo = view.default._sync
        if (name === 'DictionaryType') {
          console.log(view)
        }
        const excludeNames = ['Login', 'LoginCallback', 'RefreshToken', 'Error404']
        if (!excludeNames.includes(name) && syncInfo?.disabled !== true) {
          views[views.length] = {
            path: path,
            name: name,
            label: syncInfo?.title,
            description: syncInfo?.desc,
            cache: syncInfo?.cache !== false
          }
        }
        return views
      }, [])

      this.syncLoading = true
      const syncRes = await syncView({ views })
      this.syncLoading = false

      if (!syncRes?.success) {
        return
      }

      this.$message({
        message: this.$t('view.sync'),
        type: 'success'
      })
      this.onGetList()
    },
    // 生成前端view
    onGenerate() {
      // let bb = this.sels
    },
    onSelectAll: function(selection) {
      const selections = treeToList(selection)
      const rows = treeToList(this.viewTree)
      const checked = selections.length === rows.length
      rows.forEach(row => {
        this.$refs.multipleTable.toggleRowSelection(row, checked)
      })

      this.sels = this.$refs.multipleTable.selection
    },
    onSelect: function(selection, row) {
      const checked = selection.some(s => s.id === row.id)
      if (row.children && row.children.length > 0) {
        const rows = treeToList(row.children)
        rows.forEach(row => {
          this.$refs.multipleTable.toggleRowSelection(row, checked)
        })
      }

      this.sels = this.$refs.multipleTable.selection
    }
  }
}
</script>
