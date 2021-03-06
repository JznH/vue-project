<template>
  <div>
    <el-row :gutter="10">
      <el-col :span="4">
        <tree-table
          :title="'区域导航跟节点'"
          :height="tableHeight"
          :tree-list="treeList"
          :loading="treeLoading"
          @refreshList="onRefresTable"
          @refreshTree="onRefresTree"
        />
      </el-col>
      <el-col :span="20">
        <main-table :title="'应用列表'">
          <div slot="all-area">
            <el-form ref="formInline" :inline="true" :model="formInline" class="form-inline" size="mini">
              <el-form-item label="选择时间" prop="time">
                <el-date-picker
                  v-model="formInline.time"
                  type="daterange"
                  style="width:250px"
                  unlink-panels
                  range-separator="至"
                  start-placeholder="开始日期"
                  end-placeholder="结束日期"
                  value-format="yyyy-MM-dd"
                  :picker-options="pickerOptions"
                />
              </el-form-item>
              <el-form-item prop="region">
                <el-select v-model="formInline.region" placeholder="类型选择">
                  <el-option label="工程名称" value="1" />
                  <el-option label="批文号" value="2" />
                  <el-option label="施工单位" value="3" />
                  <el-option label="监理单位" value="4" />
                </el-select>
              </el-form-item>
              <el-form-item prop="code">
                <el-input v-model="formInline.code" placeholder="输入查询关键字" />
              </el-form-item>
              <el-form-item label="批文名称" prop="name">
                <el-input v-model="formInline.name" placeholder="批文名称" />
              </el-form-item>
              <el-form-item label="异常情况" prop="abnormal">
                <el-select v-model="formInline.abnormal" placeholder="状态选择">
                  <el-option label="全部" value="1" />
                  <el-option label="无异常" value="2" />
                  <el-option label="有异常" value="3" />
                </el-select>
              </el-form-item>
              <el-form-item label="开工状态" prop="state">
                <el-select v-model="formInline.state" placeholder="状态选择" style="width:120px;">
                  <el-option label="全部" value="1" />
                  <el-option label="计划内施工" value="2" />
                  <el-option label="无计划施工" value="3" />
                </el-select>
              </el-form-item>
            </el-form>
          </div>
          <div slot="right-area">
            <el-button type="primary" icon="el-icon-search" size="mini" @click="formSearch()">查询</el-button>
            <el-button type="primary" icon="el-icon-refresh" size="mini" @click="resetForm('formInline')">重置</el-button>
          </div>
          <div slot="left-area">
            <el-button-group>
              <el-button type="primary" size="mini" @click="handleAdd">新增</el-button>
              <el-button type="primary" size="mini" @click="changeEdit">编辑</el-button>
              <el-button type="primary" size="mini" @click="changeDel">删除</el-button>
              <el-button type="primary" size="mini" :disabled="multipleSelection.length===0">批量删除</el-button>
              <el-button type="primary" size="mini">导入</el-button>
              <el-button type="primary" size="mini">导出</el-button>
              <el-button type="primary" size="mini">移库</el-button>
              <el-button type="primary" size="mini">撤回</el-button>
            </el-button-group>
          </div>
          <div slot="table">
            <el-table
              ref="multipleTable"
              v-loading="listLoading"
              :data="list"
              element-loading-text="加载中"
              element-loading-spinner="el-icon-loading"
              highlight-current-row
              :height="tableHeight"
              :default-sort="{prop: 'date', order: 'descending'}"
              stripe
              :row-class-name="tableRowClassName"
              @selection-change="handleSelectionChange"
            >
              <el-table-column type="selection" width="35" />
              <el-table-column v-if="show" prop="transFileno" label="id" />
              <el-table-column prop="transFileno" label="批文文号" show-overflow-tooltip fixed width="150" />
              <el-table-column prop="transFilename" label="批文名称" fixed width="150" />
              <el-table-column
                prop="transType"
                label="批文类型"
                width="120"
                :filters="[{ text: '农网工程', value: '农网工程' }, { text: '城网工程', value: '城网工程' }, { text: '机井通电工程', value: '机井通电工程' }]"
                :filter-method="filterHandler"
              />
              <el-table-column label="批次年度" width="100">
                <template slot-scope="scope">
                  <el-tooltip content="点击查看详情" placement="right">
                    <router-link
                      class="underline"
                      :to="{
                        path: '/example/table',
                        name: '/modules/table-demo',
                        query: {
                          id: scope.row.transFileno
                        },
                        params: {
                          title: scope.row.transFilename+' - '+scope.row.transYear,
                          name: scope.row.createName
                        }
                      }"
                    >{{ scope.row.transYear }}</router-link>
                  </el-tooltip>
                </template>
              </el-table-column>
              <el-table-column prop="tranDate" label="批次下达时间" width="130" sortable />
              <el-table-column prop="engCount" label="工程数量" width="100" />
              <el-table-column prop="createName" label="创建者" width="100" />
              <el-table-column prop="createDate" label="创建时间" width="100" />
              <el-table-column prop="remark" label="备注" show-overflow-tooltip />
              <el-table-column label="操作" width="140">
                <template slot-scope="scope">
                  <el-button size="mini" @click="handleEdit(scope.$index, scope.row)">编辑</el-button>
                  <el-button type="danger" size="mini" @click="handleDel(scope.$index, scope.row)">删除</el-button>
                </template>
              </el-table-column>
            </el-table>
            <pagi-nation :total="total" @pageSizeChange="getRows" @currentPage="getPage" @paginationRfresh="getRfresh" />
          </div>
        </main-table>
        <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogFormVisible">
          <el-form ref="editObj" :inline="true" size="small" :model="editObj" :rules="editObjRules" label-width="80px">
            <el-form-item label="批文文号" prop="transFileno">
              <el-input v-model="editObj.transFileno" auto-complete="off" />
            </el-form-item>
            <el-form-item label="批文名称" prop="transFilename">
              <el-input v-model="editObj.transFilename" auto-complete="off" />
            </el-form-item>
            <el-form-item label="批文类型" prop="transType">
              <el-select v-model="editObj.transType" placeholder="类型选择">
                <el-option label="农网工程" value="1" />
                <el-option label="城网工程" value="2" />
                <el-option label="机井通电工程" value="3" />
              </el-select>
            </el-form-item>
            <el-form-item label="批次年度" prop="transYear">
              <el-input v-model="editObj.transYear" auto-complete="off" />
            </el-form-item>
            <el-form-item label="下达时间" prop="tranDate">
              <el-date-picker
                v-model="editObj.tranDate"
                style="width:199px;"
                type="date"
                placeholder="选择日期"
              />
            </el-form-item>
            <el-form-item label="工程数量" prop="engCount">
              <el-input v-model="editObj.engCount" auto-complete="off" />
            </el-form-item>
            <el-form-item label="创建时间" prop="createDate">
              <el-date-picker
                v-model="editObj.createDate"
                style="width:199px;"
                type="date"
                :disabled="true"
                placeholder="选择日期"
              />
            </el-form-item>
            <el-form-item label="备注" style="width:100%;" prop="remark">
              <el-input v-model="editObj.remark" type="textarea" :rows="3" style="width:400px" />
            </el-form-item>
          </el-form>
          <div slot="footer" class="dialog-footer">
            <el-button size="small" @click="dialogFormVisible = false">取消</el-button>
            <el-button size="small" type="primary" @click="dialogStatus==='create'?createData():updateData()">确定</el-button>
          </div>
        </el-dialog>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import treeTable from '@/components/TreeMenu/index'
import { getTableList } from '@/api/table'
import { getTreeList } from '@/api/table'

export default {
  components: {
    treeTable
  },
  data() {
    return {
      treeList: [],
      tableHeight: null,
      treeLoading: true,
      formInline: {
        time: '',
        region: '',
        code: '',
        name: '',
        state: '',
        abnormal: ''
      }, // 查询项
      pickerOptions: {
        disabledDate(time) {
          return time.getTime() < Date.now() - 8.64e7
        }
      },
      show: false,
      dialogStatus: '', // 弹出框标题
      textMap: {
        edit: '编辑',
        create: '新增'
      },
      dialogFormVisible: false,
      listLoading: true, // 默认加载
      list: [], // 表格数据
      multipleSelection: [], // 选中项数据
      total: 0, // 分页总数
      editObj: {
        transFileno: '',
        transFilename: '',
        transType: '',
        transYear: '',
        tranDate: '',
        engCount: '',
        createDate: new Date()
      },
      editObjRules: {
        transFileno: [{ required: true, message: '请输入批文文号', trigger: 'blur' }]
      }
    }
  },
  created() {
    this.fetchData()
    this.treeData()
  },
  mounted() {
    this.tableHeight = this.$tableHeigh()
  },
  methods: {
    formSearch() {
      console.log(this.formInline)
    },
    treeData() {
      this.treeLoading = true
      getTreeList(this.listQuery).then(response => {
        this.treeList = response.date.items
        this.treeLoading = false
      })
    },
    onRefresTable(data) {
      console.log(data.id)
      this.fetchData()
    },
    onRefresTree() {
      this.treeData()
    },
    // 获取数据
    fetchData() {
      this.listLoading = true
      getTableList(this.listQuery).then(response => {
        this.list = response.date.items
        this.total = response.total
        this.listLoading = false
      })
    },
    // 把每一行的索引放进row
    tableRowClassName({ row, rowIndex }) {
      row.index = rowIndex
    },
    // 多选选中获取信息
    handleSelectionChange(val) {
      console.log(val)
      this.multipleSelection = val
    },
    // 筛选功能
    filterHandler(value, row, column) {
      const property = column['property']
      return row[property] === value
    },
    // 分页
    getRows(val) {
      this.rows = val
      this.fetchData()
    },
    getPage(val) {
      this.page = val
      this.fetchData()
    },
    // 刷新
    getRfresh() {
      this.fetchData()
    },
    // 关闭弹窗
    editDialogClose() {
      this.dialogFormVisible = false
      this.$refs['editObj'].clearValidate()
    },
    // 重置
    resetForm(formName) {
      this.$refs[formName].resetFields()
    },
    // 新增
    createData() {
      this.$refs.editObj.validate(valid => {
        if (valid) {
          this.$confirm('确认提交吗？', '提示', {})
            .then(() => {
              this.editObj.id = (parseInt(Math.random() * 100)).toString() // mock a id
              const para = Object.assign({}, this.editObj)
              getTableList(para).then(res => {
                this.$message({
                  message: '新增成功',
                  type: 'success'
                })
                this.$refs['editObj'].clearValidate()
                this.dialogFormVisible = false
                this.fetchData()
              })
            })
            .catch(e => {
              // 打印一下错误
              console.log(e)
            })
        }
      })
    },
    // 修改
    updateData() {
      console.log(this.editObj.transFileno)
    },
    // 删除
    handleDel(index, row) {
      console.log(row)
      this.$confirm('确认删除该记录吗?', '提示', {
        type: 'warning'
      })
        .then(() => {
          const para = { id: row.id }
          getTableList(para).then(response => {
            this.$message({
              message: '删除成功',
              type: 'success'
            })
            this.fetchData()
          })
        })
        .catch(e => {
          // 打印一下错误
          this.$message({
            message: e,
            type: 'warning'
          })
        })
    },
    // 行外删除
    changeDel() {
      const len = this.multipleSelection.length
      if (len === 0) {
        this.$message.error('请选择一条数据')
      } else if (len > 1) {
        this.$message.error('只能选择一条数据')
      } else {
        this.multipleSelection.map((selectedItem) => {
          this.editObj = Object.assign({}, selectedItem)
        })
        this.handleDel(this.editObj)
      }
    },
    // 行内显示编辑页面
    handleEdit(index, row) {
      this.editObj = Object.assign({}, row) // copy obj
      this.dialogStatus = 'edit'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['editObj'].clearValidate()
      })
    },
    // 行外显示编辑页面
    changeEdit(row) {
      this.dialogStatus = 'edit'
      const len = this.multipleSelection.length
      if (len === 0) {
        this.$message.error('请选择一条数据')
      } else if (len > 1) {
        this.$message.error('只能选择一条数据')
      } else {
        this.dialogFormVisible = true
        this.multipleSelection.map((selectedItem) => {
          this.editObj = Object.assign({}, selectedItem)
        })
      }
      /* this.multipleSelection.forEach(selectedItem => {
        删除请求
        axios({
          url: '/xxxxxxx/delete/' + selectedItem.id,
          method: 'get',
          dataType: 'json'
        })
      }) */
    },
    // 显示新增界面
    resetTemp() {
      this.editObj = {
        transFileno: '',
        transFilename: '',
        transType: '',
        transYear: '',
        tranDate: '',
        engCount: '',
        createDate: new Date()
      }
    },
    handleAdd() {
      this.resetTemp()
      this.dialogStatus = 'create'
      this.dialogFormVisible = true
      this.$nextTick(() => {
        this.$refs['editObj'].clearValidate()
      })
    }
  }
}
</script>

<style scoped>
.underline {
  text-decoration: underline;
  color: #409eff;
}
</style>
