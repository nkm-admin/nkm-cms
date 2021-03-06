<template>
  <div class="container">
    <el-button
      icon="el-icon-upload2"
      type="primary"
      class="m-r-15px"
      :loading="uploadLoading"
      @click="_uploadFile"
    >
      上传文件
    </el-button>

    <el-popover
      v-model="showDirectoryInp"
      width="200"
    >
      <p class="m-b-5px">目录名</p>
      <el-input v-model="directoryName"></el-input>
      <div class="align-right m-t-15px">
        <el-button size="mini" type="text" @click="showDirectoryInp = false">取消</el-button>
        <el-button :loading="createDirLoading" type="primary" size="mini" @click="_createDirectory">确定</el-button>
      </div>
      <el-button slot="reference">新建文件夹</el-button>
    </el-popover>

    <el-row class="m-t-20px">
      <el-col :span="18">
        <div class="breadcrumb">
          <template>
            <span v-if="isRoot">全部文件</span>
            <template v-else>
              <el-link type="primary" @click="_back">返回上一级</el-link>
              <el-divider direction="vertical"></el-divider>
            </template>
          </template>
          <el-breadcrumb separator-class="el-icon-arrow-right">
            <el-breadcrumb-item v-for="(item, index) in breadcrumb" :key="index">
              <el-link v-if="index !== breadcrumb.length - 1" type="primary" @click="_breadcrumbHandler(item, index)">{{ item }}</el-link>
              <span v-else class="color-sub-text">{{ item }}</span>
            </el-breadcrumb-item>
          </el-breadcrumb>
        </div>
      </el-col>
      <el-col :span="6" class="align-right">
        <span class="color-sub-text">已加载全部，共{{ list.length }}个</span>
      </el-col>
    </el-row>

    <el-divider />

    <ul class="list-wrapper">
      <li
        v-for="(item, index) in list"
        :key="index"
        class="item align-center"
        :title="item.filename"
        @click="_enterDirectory(item)"
      >
        <el-image :src="item.preview" class="item--image" fit="contain">
          <template #placeholder>
            <div class="h-100 flex f-a-center f-j-justify">
              <x-svg-icon icon="loading" />
            </div>
          </template>
        </el-image>
        <p class="w-100 m-t-5px item--title">{{ item.filename }}</p>
      </li>
    </ul>

    <file-detail :visible.sync="showFileDetail" :data.sync="currentFile" />
  </div>
</template>

<script>
import { isEmpty } from '@/utils'
import { mapState, mapActions } from 'vuex'
import FileDetail from './components/FileDetail'
import uploadFile, { selectFileHandler } from '@/utils/upload'
import API from '@/api'
import { matchDirectory } from '@/utils/regexp'

export default {
  name: 'Media',

  components: {
    FileDetail
  },

  data() {
    return {
      uploadLoading: false,

      showDirectoryInp: false,
      createDirLoading: false,
      directoryName: '',

      currentFile: {},
      showFileDetail: false,
    }
  },

  computed: {
    ...mapState('media', ['list']),

    breadcrumb() {
      return (this.$route.query.path || '').split('/').filter(item => item)
    },

    isRoot() {
      return isEmpty(this.breadcrumb)
    }
  },

  created() {
    this._getDirectoryList()
  },

  methods: {
    ...mapActions('media', ['readDirectory']),

    async _getDirectoryList() {
      try {
        window.common.showLoading('加载中...')
        await this.readDirectory({
          path: this.$route.query.path
        })
        window.common.hideLoading()
      } catch(err) {
        window.common.hideLoading()
      }
    },

    // 进入下一级目录
    _enterDirectory(data) {
      const { isDirectory, filename } = data
      if (isDirectory) {
        this.$router.replace({
          name: 'Media',
          query: {
            path: `${this.$route.query.path || ''}/${filename}`
          }
        })
        this._getDirectoryList()
      } else {
        this.showFileDetail = true
        this.currentFile = data
      }
    },

    // 面包屑导航目录切换
    _breadcrumbHandler(dir, index) {
      this.$router.replace({
        name: 'Media',
        query: {
          path: `${this._findPath(index).join('/')}/${dir}`
        }
      })
      this._getDirectoryList()
    },

    _findPath(index) {
      const path = []
      for (let i = 0; i < index; i++) {
        path.push(this.breadcrumb[i])
      }
      return path
    },

    _back() {
      this.$router.replace({
        name: 'Media',
        query: {
          path: `${this._findPath(this.breadcrumb.length - 1).join('/')}`
        }
      })
      this._getDirectoryList()
    },

    async _uploadFile() {
      try {
        const file = await selectFileHandler()
        this.uploadLoading = true
        await uploadFile(file, {
          type: 'media',
          path: this.breadcrumb.join('/') || '/'
        })
        this.uploadLoading = false
        this._getDirectoryList()
        window.common.showMessage({
          type: 'success',
          message: '上传成功'
        })
      } catch (err) {
        this.uploadLoading = false
      }
    },

    async _createDirectory() {
      if (!matchDirectory(this.directoryName)) {
        window.common.showMessage({
          type: 'error',
          message: '目录名只能输入中文、英文、数字、短横线（-）、下划线（_）'
        })
        return
      }
      try {
        this.createDirLoading = true
        await API.media.createDirectory({
          path: this.breadcrumb.join('/') || '/',
          name: this.directoryName
        })
        this.createDirLoading = false
        this.directoryName = ''
        this._getDirectoryList()
      } catch (err) {
        this.createDirLoading = false
      }
    }
  }
}
</script>

<style lang="scss" scoped>
.breadcrumb {
  display: flex;
  align-items: center;

  /deep/ a {
    font-weight: initial;
  }

  /deep/ .el-breadcrumb__separator,
  /deep/ .el-breadcrumb__inner {
    vertical-align: middle;
  }
}
.list-wrapper {
  display: flex;
  flex-wrap: wrap;

  .item {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    width: 130px;
    height: 130px;
    padding: 15px;
    box-sizing: border-box;
    border-radius: 5px;
    cursor: pointer;

    &:hover {
      background: #f1f5fa;
    }
  }

  .item--image {
    height: calc(120px - 50px);
  }

  .item--title {
    @extend %ellipsis;
    font-size: 12px;
    line-height: 1.5;
  }
}
</style>
