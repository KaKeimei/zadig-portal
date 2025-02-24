<template>
  <div class="integration-other-container">
    <el-dialog title="其他外部系统" :close-on-click-modal="false" :visible.sync="dialogExternalVisible">
      <el-form :model="externalEdit" @submit.native.prevent :rules="externalRules" ref="externalForm" label-position="left" label-width="100px">
        <el-form-item label="系统名称" prop="name">
          <el-input v-model="externalEdit.name" placeholder="输入系统名称" size="small"></el-input>
        </el-form-item>
        <el-form-item label="访问地址" prop="server">
          <el-input v-model.trim="externalEdit.server" placeholder="输入系统访问地址" size="small"></el-input>
        </el-form-item>
        <el-form-item label="API Token" prop="api_token">
          <el-input v-model="externalEdit.api_token" v-if="dialogExternalVisible" type="password" placeholder="输入 API Token" size="small"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" native-type="submit" size="small" @click="updateExternalConfig" class="start-create">{{$t(`global.confirm`)}}</el-button>
        <el-button plain native-type="submit" size="small" @click="dialogExternalVisible = false">{{$t(`global.cancel`)}}</el-button>
      </div>
    </el-dialog>

    <div class="tab-container">
      <template>
        <el-alert type="info" :closable="false">
          <template>
            支持集成其他外部系统，配置后工作流可以调用外部系统 API，详情可参考
            <el-link
              style="font-size: 14px; vertical-align: baseline;"
              type="primary"
              :href="`https://docs.koderover.com/zadig/settings/others/`"
              :underline="false"
              target="_blank"
            >{{$t(`global.helpDoc`)}}</el-link>。
          </template>
        </el-alert>
      </template>
      <div class="sync-container">
        <el-button size="small" type="primary" plain @click="dialogExternalVisible = true">{{$t(`global.add`)}}</el-button>
      </div>
      <el-table :data="external" style="width: 100%;">
        <el-table-column label="系统名称" prop="name"></el-table-column>
        <el-table-column label="访问地址" prop="server"></el-table-column>
        <el-table-column label="API Token">
          <template>**********</template>
        </el-table-column>
        <el-table-column :label="$t(`global.operation`)" width="160">
          <template slot-scope="{ row }">
            <el-button type="primary" size="mini" plain @click="handleExternalEdit(row)">{{$t(`global.edit`)}}</el-button>
            <el-button type="danger" size="mini" @click="handleExternalDelete(row.id)" plain>{{$t(`global.delete`)}}</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
  </div>
</template>
<script>
import bus from '@utils/eventBus'
import {
  getExternalSystemsAPI,
  getExternalSystemByIdAPI,
  updateExternalSystemAPI,
  deleteExternalSystemAPI,
  createExternalSystemAPI
} from '@api'
import { cloneDeep } from 'lodash'
const externalInfo = {
  id: '',
  name: '',
  server: '',
  api_token: ''
}

const validateURL = (rule, value, callback) => {
  if (value === '') {
    callback(new Error('请输入系统访问地址，包含协议'))
  } else {
    if (value.endsWith('/')) {
      callback(new Error('URL 末尾不能包含 /'))
    } else {
      callback()
    }
  }
}

export default {
  data () {
    this.externalRules = {
      name: {
        required: true,
        message: '请输入系统名称',
        trigger: ['blur', 'change']
      },
      server: [
        {
          required: true,
          trigger: ['blur', 'change'],
          validator: validateURL
        }
      ]
    }

    return {
      external: [],
      externalEdit: cloneDeep(externalInfo),
      dialogExternalVisible: false
    }
  },
  watch: {
    dialogExternalVisible (val) {
      if (!val) {
        this.handleExternalCancel()
      }
    }
  },
  methods: {
    getExternalConfig () {
      const key = this.$utils.rsaEncrypt()
      getExternalSystemsAPI(key).then(res => {
        res.external_system.forEach(item => {
          item.api_token = this.$utils.aesDecrypt(item.api_token)
        })
        this.external = res.external_system
      })
    },
    handleExternalEdit (row) {
      this.dialogExternalVisible = true
      this.externalEdit = cloneDeep(row)
      getExternalSystemByIdAPI(row.id).then(res => {
        if (this.externalEdit.id === res.id) {
          this.externalEdit = res
        }
      })
    },
    handleExternalDelete (id) {
      this.$confirm(`确定要删除这个外部系统配置吗？`, '确认', {
        confirmButtonText: this.$t(`global.confirm`),
        cancelButtonText: this.$t(`global.cancel`),
        type: 'warning'
      }).then(() => {
        deleteExternalSystemAPI(id).then(res => {
          this.getExternalConfig()
          this.$message({
            message: '外部系统配置删除成功',
            type: 'success'
          })
        })
      })
    },
    updateExternalConfig () {
      this.$refs.externalForm.validate(valid => {
        if (valid) {
          const id = this.externalEdit.id
          const payload = {
            name: this.externalEdit.name,
            server: this.externalEdit.server,
            api_token: this.externalEdit.api_token
          }
          const fn = id
            ? updateExternalSystemAPI(id, payload)
            : createExternalSystemAPI(payload)
          fn.then(res => {
            this.getExternalConfig()
            this.dialogExternalVisible = false
            this.$message({
              message: `外部系统配置${id ? '修改' : '创建'}成功`,
              type: 'success'
            })
          })
        }
      })
    },
    handleExternalCancel () {
      this.externalEdit = cloneDeep(externalInfo)
      this.$nextTick(() => {
        this.$refs.externalForm.clearValidate()
      })
    }
  },
  mounted () {
    this.getExternalConfig()
    bus.$emit('set-topbar-title', { title: '', breadcrumb: [{ title: this.$t(`sidebarMenu.systemIntegration`), url: '/v1/system/integration' }, { title: this.$t(`sysSetting.integration.externalSystemTab`), url: '' }] })
  }
}
</script>

<style lang="less" scoped>
.integration-other-container {
  position: relative;
  flex: 1;
  padding: 15px 30px;
  overflow: auto;
  font-size: 13px;

  .tab-container {
    .sync-container {
      padding-top: 15px;
      padding-bottom: 15px;
    }
  }

  /deep/.el-dialog {
    width: 550px;

    .el-dialog__header {
      padding: 15px;
      text-align: center;
      border-bottom: 1px solid #e4e4e4;

      .el-dialog__close {
        font-size: 10px;
      }
    }

    .el-dialog__body {
      color: #606266;
      font-size: 14px;

      .el-form-item {
        margin-bottom: 15px;
      }
    }
  }
}
</style>
