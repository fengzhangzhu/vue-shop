<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>

    <!-- 卡片视图 -->
    <el-card>
      <!-- 添加角色按钮区域 -->
      <el-row >
        <el-col >
          <el-button type="primary" @click="addDialogVisible=true">添加角色</el-button>
        </el-col>
      </el-row>

      <!-- 角色列表区域 -->
      <el-table :data="rolelist" border stripe>
        <!-- 展开列 -->
        <el-table-column type="expand" >
          <template slot-scope="scope">
            <!-- 栅格布局  -->
            <el-row :class="['bdbottom',i1===0 ? 'bdtop':'','vcenter']" v-for="(item1,i1) in scope.row.children" :key="item1.id">
              <!-- 渲染一级权限 -->
              <el-col :span="5">
                <el-tag closable
                     @close="removeRightById(scope.row,item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 渲染二级和三级权限 -->
              <el-col :span="19">
                <!-- 通过for循环 嵌套渲染二级权限 -->
                <el-row :class="[i2===0 ? '':'bdtop','vcenter']"  v-for="(item2,i2) in item1.children" :key="item2.id">
                  <el-col :span="6">
                    <el-tag type="success" closable
                     @close="removeRightById(scope.row,item2.id)">{{item2.authName}}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                   <!-- 通过for循环 嵌套渲染三级权限 -->
                  <el-col :span="18">
                     <el-tag type="warning" v-for="(item3,i3) in item2.children" :key="item3.id" closable
                     @close="removeRightById(scope.row,item3.id)">{{item3.authName}}</el-tag>  
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <!-- 索引列 -->
        <el-table-column type="index" label="#"></el-table-column>
        <el-table-column label="角色名称" prop="roleName"></el-table-column>
        <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
        <!-- 通过作用域插槽绑定数据 -->
        <el-table-column label="操作" >
          <template slot-scope="scope" width="300px">
           <el-button size="mini" type="primary" icon="el-icon-edit" @click="showEditDialog(scope.row.id)">编辑</el-button>
           <el-button size="mini" type="danger" icon="el-icon-delete"  @click="removeRoleById(scope.row.id)">删除</el-button>
           <el-button size="mini" type="warning" icon="el-icon-setting" @click="showSetRightDialog(scope.row)">分配权限</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <!-- 添加角色的对话框 -->
    <el-dialog title="添加角色" :visible.sync="addDialogVisible" width="50%" @close="addDialogClosed">
      <!-- 内容主体区域  model:数据绑定对象，rules：数据验证规则，ref:引用对象-->
      <el-form :model="addRoleForm" :rules="addRoleFormRules" ref="addRoleFormRef" label-width="70px">
        <!-- label:显示的文本，prop:具体的校验规则，校验规则一定要在addFormRules做相关定义 -->
        <el-form-item label="角色名称" prop="roleName"  label-width="100px">
          <el-input v-model="addRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc" label-width="100px" >
          <el-input v-model="addRoleForm.roleDesc" ></el-input>
        </el-form-item>
      </el-form>
        <!-- 底部区域 -->
      <span slot="footer" class="dialog-footer">
        <el-button @click="addDialogVisible = false">取 消</el-button>
        <el-button type="primary"  @click="addRole">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 修改角色的对话框 -->
    <el-dialog title="修改角色" :visible.sync="editDialogVisible" width="50%" @close="editDialogClosed">
      <el-form :model="editRoleForm" :rules="editRoleFormRules" ref="editRoleFormRef" label-width="70px">
        <el-form-item label="角色名称"  prop="roleName" label-width="100px">
          <el-input v-model="editRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop="roleDesc" label-width="100px"> 
          <el-input v-model="editRoleForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="editDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editRoleInfo">确 定</el-button>
      </span>
    </el-dialog>

    <!-- 分配权限的对话框 -->
    <el-dialog title="分配权限" :visible.sync="setRightDialogVisible" width="50%" @close="setRightDialogClosed">
      <!-- 树形控件 -->
      <el-tree :data="rightslist" :props="treeProps" show-checkbox node-key="id" default-expand-all 
      :default-checked-keys="defKeys" ref="treeRef"></el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="allotRights">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      // 获取角色列表的参数对象
      queryInfo:{
        query:'',   //查询参数,可以为空
      },
      // 所有角色列表数据
      rolelist:[],
      // 控制添加角色对话框的显示与隐藏，默认是隐藏
      addDialogVisible: false,
      // 添加角色的表单数据
      addRoleForm:{
        roleName:'',  //角色名称不能为空
        roleDesc:'', //角色描述可以为空
      },
      // 添加表单的验证规则对象
      addRoleFormRules:{
        roleName:[
          { required: true, message: '请输入角色名', trigger: 'blur' },
          { min: 1, max: 10, message: '角色名长度在 1~10个字符之间', trigger: 'blur' }         
        ],
        roleDesc:[
          { required: true, message: '请输入角色描述', trigger: 'blur' },
          { min: 1, max: 50, message: '角色描述长度在 1~50个字符之间', trigger: 'blur' }         
        ],
      },
      // 控制修改角色对话框的显示与隐藏
      editDialogVisible: false,
      // 查询到的角色信息对象
      editRoleForm:{},
      // 修改表单的验证规则对象
      editRoleFormRules:{
        roleName:[
          { required: true, message: '请输入角色名', trigger: 'blur' },
          { min: 1, max: 10, message: '角色名长度在 1~10个字符之间', trigger: 'blur' }         
        ],
        roleDesc:[
          { required: true, message: '请输入角色描述', trigger: 'blur' },
          { min: 1, max: 50, message: '角色描述长度在 1~50个字符之间', trigger: 'blur' }         
        ],
      },
      // 控制分配权限对话框的显示与隐藏
      setRightDialogVisible: false,
      // 所有权限的数据
      rightslist:[],
      // 树形控件的属性绑定对象
      treeProps:{
        label:'authName',
        children:'children'
      },
      // 默认选中的节点Id值数组
      defKeys: [],
      // 当前即将分配权限的角色id
      roleId:''
    }
  },
  created() {
    this.getRoleList()
  },
  methods: {
    // 获取所有角色的列表
    async getRoleList(){
      const {data: res} = await this.$http.get('roles')

      if(res.meta.status !== 200){
        return this.$message.error('获取角色列表失败！')
      }

      this.rolelist = res.data
      console.log(this.rolelist)
    },
    // 监听添加用户对话框的关闭事件
    addDialogClosed(){
      this.$refs.addRoleFormRef.resetFields()
    },
    // 点击按钮，添加新角色
    addRole(){
      this.$refs.addRoleFormRef.validate(async valid => {
        // console.log(valid)
        if (!valid) return
        // 可以发起添加用户的网络请求
        const {data:res} =  await this.$http.post('roles',this.addRoleForm)

        if(res.meta.status !== 201){
          this.$message.error('添加角色失败！')
        }

        this.$message.success('添加角色成功！')
      //  隐藏添加用户的对话框
      this.addDialogVisible = false
      // 重新获取角色列表数据
      this.getRoleList()
      })
    },
    // 展示编辑角色的对话框
    async showEditDialog(id){
      // console.log(id)
      const {data:res} = await this.$http.get('roles/' + id)

      if(res.meta.status !== 200){
        return this.$message.error('查询角色信息失败！')
      }
      
      this.editRoleForm = res.data
      this.editDialogVisible = true
   },
   // 监听修改角色对话框的关闭事件
    editDialogClosed(){
      this.$refs.editRoleFormRef.resetFields()
    },
    // 修改角色信息并提交
    editRoleInfo(){
      this.$refs.editRoleFormRef.validate(async valid => {
        // console.log(valid)
        if(!valid) return
        // 发起修改角色信息的数据请求
        const {data:res} = await this.$http.put('roles/' + this.editRoleForm.roleId,{
          roleName: this.editRoleForm.roleName,
          roleDesc: this.editRoleForm.roleDesc,
        })

        if(res.meta.status !== 200){
          return this.$message.error('更新角色信息失败！')
        }

        // 关闭对话框
        this.editDialogVisible = false
        // 刷新数据列表
        this.getRoleList()
        // 提示修改成功
        this.$message.success('更新角色信息成功！')
      })
    },
     // 根据ID删除对应的角色信息
    async removeRoleById(id){
      // 弹框询问用户是否删除数据
      const confirmResult =  await this.$confirm('此操作将永久删除该用户, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).catch(err => err)  //catch捕获错误消息

        //如果用户确认删除，则返回值为字符串 confirm

        // 如果用户取消了删除，则返回值为字符串 cancel
        // console.log(confirmResult)
        if (confirmResult !== 'confirm'){
          return this.$message.info('已取消删除')
        }

        const {data:res} = await this.$http.delete('roles/' + id)

        if(res.meta.status !== 200){
          return this.$message.error('删除角色失败！')
        }

        this.$message.success('删除角色成功！')
        this.getRoleList()
    },
    // 根据id删除对应的权限
    async removeRightById(role,rightId){
      // 弹框提示用户是否要删除
      const confirmResult = await this.$confirm('此操作将永久删除该文件, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }
      ).catch(err => err)
      
      if(confirmResult !== 'confirm'){
        return this.$message.info('取消了删除！')
      }

      const {data:res} = await this.$http.delete(`roles/${role.id}/rights/${rightId}`)

      if (res.meta.status !== 200){
        return this.$message.error('删除权限失败！')
      }
      // 防止整个列表刷新影响客户体验
      // this.getRoleList()
      role.children = res.data
    },
    // 展示分配权限的对话框
    async showSetRightDialog(role){
      // 保存角色ID
      this.roleId = role.id
      // 获取所有权限的数据
      const {data:res} = await this.$http.get('rights/tree')

      if(res.meta.status !== 200){
        return this.$message.error('获取权限数据失败！')
      }
      // 获取到的权限数据保存到data中
      this.rightslist =res.data
      console.log(this.rightslist)

      // 递归获取三级节点的Id
      this.getLeafKeys(role,this.defKeys)
      this.setRightDialogVisible = true
    },
    // 通过递归的形式，获取角色下所有三级权限的id,并保存到defKeys数组中
    getLeafKeys(node,arr){
      //如果当前node节点不包含children属性，则是三级节点
      if(!node.children){
        return arr.push(node.id)
      }

      node.children.forEach(item => this.getLeafKeys(item,arr))
    },
    // 监听分配权限对话框的关闭事件
    setRightDialogClosed(){
      this.defKeys= []
    },
    // 点击为角色分配权限
     async allotRights(){
      //  拿到所有全选及半选的ID
      const keys = [
        ...this.$refs.treeRef.getCheckedKeys(),
        ...this.$refs.treeRef.getHalfCheckedKeys()
      ]

      const idStr = keys.join(',')

      const {data: res} = await this.$http.post(`roles/${this.roleId}/rights`,{rids:idStr})

      if (res.meta.status !== 200){
        return this.$message.error('分配权限失败！')
      }

      this.$message.success('分配权限成功！')
      // 重新刷新角色列表
      this.getRoleList()
      // 关闭整个对话框
      this.setRightDialogVisible=false
    }

  }
}
</script>


<style lang="less" scoped>
  .el-tag{
    margin: 7px;
  }

  .bdtop{
    border-top: 1px solid #eee;
  }

  .bdbottom{
    border-bottom: 1px solid #eee;
  }
  
  .vcenter{
    display: flex;
    align-items: center;
  }
</style>