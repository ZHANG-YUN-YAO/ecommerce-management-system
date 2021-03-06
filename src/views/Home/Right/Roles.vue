<template>
  <div>
    <!-- 头部面包屑导航 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 内容 -->
    <el-card class="box-card">
      <!-- 添加角色 -->
      <el-row>
        <el-col>
          <el-button type="primary">添加角色</el-button>
        </el-col>
      </el-row>
      <!-- 角色列表 -->
      <el-table :data="roleList" style="width: 100%" border stripe>
        <!-- 展开部分 -->
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-row
              v-for="(item1, i1) in scope.row.children"
              :key="item1.id"
              :class="['bdbottom', 'center', i1 === 0 ? 'bdtop':'']"
            >
              <!-- 一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="removeRight(scope.row, item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 二、三级权限 -->
              <el-col :span="19">
                <el-row
                  v-for="(item2, i2) in item1.children"
                  :key="item2.id"
                  :class="[i2 === 0 ? '' : 'bdtop', 'center']"
                >
                  <el-col :span="6">
                    <el-tag
                      type="success"
                      closable
                      @close="removeRight(scope.row, item2.id)"
                    >{{item2.authName}}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="18">
                    <el-tag
                      type="warning"
                      v-for="item3 in item2.children"
                      :key="item3.id"
                      closable
                      @close="removeRight(scope.row, item3.id)"
                    >{{item3.authName}}</el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <el-table-column type="index" label="#"></el-table-column>
        <el-table-column prop="roleName" label="角色名称"></el-table-column>
        <el-table-column prop="roleDesc" label="角色描述"></el-table-column>
        <el-table-column label="操作" width="300px">
          <template slot-scope="scope">
            <el-button type="primary" icon="el-icon-edit" size="mini">编辑</el-button>
            <el-button type="danger" icon="el-icon-delete" size="mini">删除</el-button>
            <el-button
              type="warning"
              icon="el-icon-setting"
              size="mini"
              @click="showRightDialog(scope.row)"
            >分配权限</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
    <!-- 分配权限对话框 -->
    <el-dialog title="分配权限" :visible.sync="rightDialogVisible" @close="closeRightDialog">
      <el-tree
        :data="rightList"
        :props="treeProps"
        show-checkbox
        node-key="id"
        default-expand-all
        :default-checked-keys="defKeys"
        ref="treeRef"
      ></el-tree>
      <div slot="footer" class="dialog-footer">
        <el-button @click="rightDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="assignRight">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import axios from 'axios'
export default {
  created () {
    this.getRoleList()
  },
  data () {
    return {
      // 角色列表
      roleList: [],
      roleId: '',
      rightDialogVisible: false,
      // 角色所有权限列表
      rightList: [],
      // 树组件属性
      treeProps: {
        label: 'authName',
        children: 'children'
      },
      // 默认选中的树节点
      defKeys: []
    }
  },
  methods: {
    // 获取角色列表
    async getRoleList () {
      const res = await axios.get('roles')
      if (res.data.meta.status !== 200) {
        return this.$message.error('获取角色列表失败')
      }
      this.roleList = res.data.data
    },
    // 删除具体角色权限
    removeRight (role, rightId) {
      this.$confirm('此操作将永久删除该权限, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(async () => {
          const res = await axios.delete(`roles/${role.id}/rights/${rightId}`)
          if (res.data.meta.status !== 200) {
            return this.$message.error('删除角色权限失败')
          }
          role.children = res.data.data
          this.$message.success('删除成功!')
        })
        .catch(err => err)
    },
    // 显示分配权限对话框
    async showRightDialog (role) {
      const res = await axios.get('rights/tree')
      if (res.data.meta.status !== 200) { return this.$message.error('获取当前用户权限失败') }
      this.rightList = res.data.data
      this.getLeafKeys(role, this.defKeys)
      this.rightDialogVisible = true
      this.roleId = role.id
    },
    // 通过递归加载已选节点
    getLeafKeys (node, arr) {
      if (!node.children) {
        return arr.push(node.id)
      }
      for (let i = 0; i < node.children.length; i++) {
        this.getLeafKeys(node.children[i], arr)
      }
    },
    // 监听分配权限对话框的关闭
    closeRightDialog () {
      this.defKeys = []
    },
    // 确认分配权限
    async assignRight () {
      const keys = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys()
      ]
      const keyStr = keys.join(',')
      const res = await axios.post(`roles/${this.roleId}/rights`, {
        rids: keyStr
      })
      if (res.data.meta.status !== 200) {
        return this.$message.error('分配失败')
      }
      this.rightDialogVisible = false
      this.$message.success('分配成功')
      this.getRoleList()
    }
  }
}
</script>

<style lang="scss" scoped>
.el-table {
  .el-row {
    .el-tag {
      margin: 7px;
    }
  }
}
.bdtop {
  border-top: 1px solid #eee;
}
.bdbottom {
  border-bottom: 1px solid #eee;
}
.center {
  display: flex;
  align-items: center;
}
</style>
