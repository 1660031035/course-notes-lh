<template>
  <div>
    <el-card class="box-card">
      <div slot="header" class="clearfix">
        <span>文章列表</span>
      </div>
      <!-- 搜索区域 -->
      <div class="search-box">
        <el-form :inline="true" :model="q">
          <el-form-item label="文章分类">
            <el-select
              v-model="q.cate_id"
              placeholder="请选择分类"
              size="small"
            >
              <el-option
                v-for="item in getList"
                :key="item.id"
                :label="item.cate_name"
                :value="item.id"
              ></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="发布状态" style="margin-left: 15px">
            <el-select v-model="q.state" placeholder="请选择状态" size="small">
              <el-option label="已发布" value="已发布"></el-option>
              <el-option label="草稿" value="草稿"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" size="small" @click="initArtList"
              >筛选</el-button
            >
            <el-button type="info" size="small" @click="resetList"
              >重置</el-button
            >
          </el-form-item>
        </el-form>
        <!-- 发表文章的按钮 -->
        <el-button
          type="primary"
          size="small"
          class="btn-pub"
          @click="pubDialogVisible = true"
          >发表文章</el-button
        >
        <!-- 发表文章对话框start -->
        <!-- 对话框关闭后,清空内容 -->
        <el-dialog
          title="发表文章"
          :visible.sync="pubDialogVisible"
          fullscreen
          :before-close="handleClose"
          @close="onloadClosed"
        >
          <!-- 发布文章的对话框 -->
          <el-form
            :model="pubForm"
            :rules="pubFormRules"
            ref="pubFormRef"
            label-width="100px"
          >
            <!-- 文章标题 -->
            <el-form-item label="文章标题" prop="title">
              <el-input
                v-model="pubForm.title"
                placeholder="请输入标题"
              ></el-input>
            </el-form-item>
            <!-- 文章分类 -->
            <el-form-item label="文章分类" prop="cate_id">
              <el-select
                v-model="pubForm.cate_id"
                placeholder="请选择分类"
                style="width: 100%"
              >
                <el-option
                  v-for="item in getList"
                  :key="item.id"
                  :label="item.cate_name"
                  :value="item.id"
                ></el-option>
              </el-select>
            </el-form-item>
            <!-- 富文本编辑器 -->
            <el-form-item label="文章内容" prop="content">
              <!-- 使用 v-model 进行双向的数据绑定 -->
              <quill-editor v-model="pubForm.content"></quill-editor
            ></el-form-item>
            <!-- 文章封面 -->
            <el-form-item label="文章封面">
              <!-- 用来显示封面的图片 -->
              <img
                src="../../../assets/images/cover.jpg"
                class="cover-img"
                ref="imgRef"
                alt=""
              />
              <input
                type="file"
                style="display: none"
                accept="image/*"
                @change="onloadCover"
                ref="iptFile"
              />
              <div>
                <el-button @click="$refs.iptFile.click()">+选择封面</el-button>
              </div>
            </el-form-item>
          </el-form>
          <el-button type="primary" @click="pubArticle('已发布')"
            >发布</el-button
          >
          <el-button type="info" @click="pubArticle('草稿')"
            >存为草稿</el-button
          >
        </el-dialog>

        <!-- 发表文章对话框end -->
      </div>

      <!-- 文章表格区域 -->
      <el-table :data="artList" style="width: 100%" border stripe>
        <el-table-column label="文章标题" prop="title">
          <template v-slot="{ row }">
            <el-link type="primary" @click="showDatial(row.id)">
              {{ row.title }}
            </el-link>
          </template>
        </el-table-column>
        <el-table-column label="分类" prop="cate_name"></el-table-column>
        <el-table-column label="发表时间" prop="pub_date">
          <template v-slot="{ row }">
            {{ formDate(row.pub_date) }}
          </template>
        </el-table-column>
        <el-table-column label="状态" prop="state"></el-table-column>
        <el-table-column label="操作">
          <template v-slot="{ row }">
            <el-button type="danger" size="mini" @click="deleteArt(row.id)"
              >删除</el-button
            >
          </template>
        </el-table-column>
      </el-table>
      <!-- 查看文章详情的对话框 -->
      <el-dialog title="文章预览" :visible.sync="detailVisible" width="80%">
        <h1 class="title">{{ article.title }}</h1>

        <div class="info">
          <span>作者：{{ article.nickname || article.username }}</span>
          <span>发布时间：{{ formDate(article.pub_date) }}</span>
          <span>所属分类：{{ article.cate_name }}</span>
          <span>状态：{{ article.state }}</span>
        </div>

        <!-- 分割线 -->
        <el-divider></el-divider>

        <img
          :src="'http://big-event-vue-api-t.itheima.net' + article.cover_img"
          alt=""
        />

        <div v-html="article.content"></div>
      </el-dialog>
      <!-- 分页区域 -->
      <!-- 分页区域 -->
      <el-pagination
        @size-change="handleSizeChange"
        @current-change="handleCurrentChange"
        :current-page="q.pagenum"
        :page-sizes="[2, 3, 5, 10]"
        :page-size="q.pagesize"
        layout="total, sizes, prev, pager, next, jumper"
        :total="total"
      >
      </el-pagination>
    </el-card>
  </div>
</template>

<script>
// 导入时间组件
import dayjs from 'dayjs'
// 导入默认的封面图片
import defaultImg from '@/assets/images/cover.jpg'
export default {
  name: 'ArtList',
  data () {
    return {
      // 文章详情
      article: [],
      // 查询参数对象
      q: {
        pagenum: 1,
        pagesize: 2,
        cate_id: '',
        state: ''
      },
      pubForm: {
        title: '',
        cate_id: '',
        // 文章的内容
        content: '',
        cover_img: null,
        // 定义state属性,用来存储文章的发布状态
        state: ''
      },
      pubFormRules: {
        title: [{ required: true, message: '请输入文章标题', trigger: 'blur' },
        { min: 1, max: 30, message: '文章标题的长度为1-30个字符', trigger: 'blur' }],
        cate_id: [{ required: true, message: '请选择文章分类', trigger: 'change' }],
        content: [
          { required: true, message: '请输入文章内容', trigger: 'blur' }
        ]


      },
      // 控制发表文章的显示与隐藏
      pubDialogVisible: false,
      // 文章分类
      getList: [],
      // 文章列表数据
      artList: [],
      // 总数据条数
      total: 0,
      // 文章详情弹框
      detailVisible: false
    }
  },
  methods: {
    // 重置按钮
    resetList () {
      this.q = {
        pagenum: 1,
        pagesize: 2,
        cate_id: '',
        state: ''
      }
      this.initArtList()
    },
    // 关闭前询问对话框
    handleClose (done) {
      this.$confirm('此操作将永久删除该文件, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(_ => {
          done();
        })
        .catch(_ => { });
    },
    // 初始化文章分类的列表数据
    async getCateList () {
      const { data: res } = await this.$http.get('/my/cate/list')
      if (res.code === 0) {
        this.getList = res.data

      }
    },
    // 获取文章列表数据
    async initArtList () {
      const { data: res } = await this.$http.get('/my/article/list', {
        params: this.q

      })
      if (res.code !== 0) return this.$message.error('获取文章列表数据失败！')
      this.artList = res.data
      this.total = res.total
    },
    onloadCover (e) {
      // 获取用户选择的文件
      const files = e.target.files
      if (files.length === 0) {
        // 没有选择封面
        this.pubForm.cover_img = null
        // 设置封面的默认地址
        this.$refs.imgRef.setAttribute('src', defaultImg)

      } else {
        this.pubForm.cover_img = files[0]
        const url = URL.createObjectURL(files[0])
        this.$refs.imgRef.setAttribute('src', url)
        // element.setAttribute(name, value) 将value新值赋值给name,
        // 清空value
        e.target.value = ''
      }
    },
    onloadClosed () {
      // 清空关键数据
      this.$refs.pubFormRef.resetFields()
      // 清空文章内容
      this.pubForm.content = ''
      // 清空封面
      this.pubForm.cover_img = null
      // 清空状态
      this.pubForm.state = ''
      // 封面恢复默认
      this.$refs.imgRef.setAttribute('src', defaultImg)
    },
    pubArticle (state) {
      // 设置发布状态
      this.pubForm.state = state
      // 表单预校验
      this.$refs.pubFormRef.validate(async valid => {
        if (!valid) return this.$message.error('请完善信息')
        // 判断文章封面
        if (!this.pubForm.cover_img) return this.$message.error('请添加封面')
        // 发布文章
        // console.log(this.pubForm)
        // 发送请求
        // 创建 FormData 对象
        const fd = new FormData()
        // 向 FormData 中追加数据
        Object.keys(this.pubForm).forEach(key => {
          fd.append(key, this.pubForm[key])
        })
        // 发送请求
        const { data: res } = await this.$http.post('/my/article/add', fd)
        if (res.code !== 0) return this.$message.error(res.message)
        this.$message.success(res.message)
        // 重新获取列表
        this.initArtList()
        // 关闭对话框
        this.pubDialogVisible = false
      })
    },

    // 切换当前页时触发
    handleCurrentChange (newPage) {
      // 设置q对象中pagenum页数
      this.q.pagenum = newPage
      // 重新获取文章数据
      this.initArtList()
    },
    // 声明 handleSizeChange 函数 切换每条页数时触发
    handleSizeChange (newSize) {
      // 为pagesize赋值
      this.q.pagenum = 1
      this.q.pagesize = newSize
      // 重新获取文章数据
      this.initArtList()
    },
    //定义时间函数 需要return
    formDate (time) {
      return dayjs(time).format('YYYY-MM-DD HH:mm:ss')
    },
    // 文章详情 注册点击事件 + 发送请求 + data函数中定义数据项
    async showDatial (id) {
      const { data: res } = await this.$http.get('/my/article/info', {
        params: { id }
      })
      if (res.code === 0) {
        this.article = res.data
        console.log(this.article)
        this.detailVisible = true
      }
    },
    // 删除文章 询问对话框 + 发送请求  + 刷新列表
    deleteArt (id) {
      // 询问是否删除
      this.$confirm('此操作将永久删除该文件, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(
        async () => {
          // 发送请求
          const { data: res } = await this.$http.delete('/my/article/info', {
            params: { id }
          })
          if (res.code !== 0) return this.$message.error(res.message)
          this.$message.success(res.message)
          // 如果刷新之前 当前页的数据只有1条且当前页码大于1则q.pagenum要减1
          if (this.artList.length === 1 && this.q.pagenum > 1) {
            this.q.pagenum--
          }
          this.initArtList()
        }).catch(() => {
          // 删除失败
        })
    }
  },

  created () {
    this.getCateList()
    this.initArtList()
  }

}
</script>

<style lang="less" scoped>
.search-box {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  .btn-pub {
    margin-top: 5px;
  }
}
// 设置富文本编辑器的默认最小高度
/deep/ .ql-editor {
  min-height: 300px;
}
// 设置图片封面的宽高
.cover-img {
  width: 400px;
  height: 280px;
  object-fit: cover;
  //object-fit: cover; 把图片等比例裁剪,并且居中
}
.el-pagination {
  margin-top: 15px;
}
// 文章详情弹框
.title {
  font-size: 24px;
  text-align: center;
  font-weight: normal;
  color: #000;
  margin: 0 0 10px 0;
}

.info {
  font-size: 12px;
  span {
    margin-right: 20px;
  }
}
</style>
