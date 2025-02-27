<!--
 * @Description:
 * @Author: gumingchen
 * @Email: 1240235512@qq.com
 * @Date: 2021-04-21 22:52:19
 * @LastEditors: gumingchen
 * @LastEditTime: 2021-05-28 22:36:10
-->
<template>
  <div class="g-container">
    <el-form ref="refForm" :inline="true" @keyup.enter="reacquireHandle()">
      <el-form-item>
        <el-input v-model="form.bean_name" placeholder="Bean名称" clearable />
      </el-form-item>
      <el-form-item>
        <el-button v-repeat @click="reacquireHandle()">查询</el-button>
        <el-button v-repeat @click="clearJson(form), reacquireHandle()">重置</el-button>
        <el-button v-permission="'quartz:schedule:task:create'" type="primary" @click="addEditHandle()">新增</el-button>
        <el-button
          v-permission="'quartz:schedule:task:run'"
          type="danger"
          @click="runHandle()"
          :disabled="selection.length <= 0">批量立即执行</el-button>
        <el-button
          v-permission="'quartz:schedule:task:resume'"
          type="danger"
          @click="resumeHandle()"
          :disabled="selection.length <= 0">批量恢复</el-button>
        <el-button
          v-permission="'quartz:schedule:task:pause'"
          type="danger"
          @click="pauseHandle()"
          :disabled="selection.length <= 0">批量暂停</el-button>
        <el-button
          v-permission="'quartz:schedule:task:delete'"
          type="danger"
          @click="delHandle()"
          :disabled="selection.length <= 0">批量删除</el-button>
      </el-form-item>
    </el-form>
    <el-table
      border
      class="g-table"
      element-loading-spinner="el-icon-loading"
      v-loading="loading"
      :data="list"
      @selection-change="selectionHandle">
      <el-table-column align="center" type="selection" width="50" />
      <el-table-column
        align="center"
        label="ID"
        prop="id"
        width="80" />
      <el-table-column
        align="center"
        label="Bean名称"
        prop="bean_name"
        min-width="120" />
      <el-table-column
        align="center"
        label="Cron表达式"
        prop="cron_expression"
        width="160px" />
      <el-table-column
        align="center"
        label="参数"
        prop="params"
        :show-overflow-tooltip="true" />
      <el-table-column
        align="center"
        label="状态"
        prop="status"
        width="80px">
        <template v-slot="{ row }">
          <el-tag :type="row.status === 1 ? 'success' : 'info'">
            {{ dictionaryMap[row.status] }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column
        align="center"
        label="备注"
        prop="remark"
        min-width="200px" />
      <el-table-column
        align="center"
        label="操作"
        width="240"
        fixed="right">
        <template v-slot="{ row }">
          <el-button
            v-permission="'quartz:schedule:task:update'"
            type="text"
            @click="addEditHandle(row.id)">编辑</el-button>
          <el-button
            v-permission="'quartz:schedule:task:run'"
            type="text"
            @click="runHandle(row.id)">立即执行</el-button>
          <el-button
            v-if="row.status === 0"
            v-permission="'quartz:schedule:task:resume'"
            type="text"
            @click="resumeHandle(row.id)">恢复</el-button>
          <el-button
            v-if="row.status === 1"
            v-permission="'quartz:schedule:task:pause'"
            type="text"
            @click="pauseHandle(row.id)">暂停</el-button>
          <el-button
            v-permission="'quartz:schedule:task:delete'"
            type="text"
            @click="delHandle(row.id)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <g-page :page="page" @change="pageChangeHandle" />
    <add-edit ref="refAddEdit" v-if="visible" @refresh="getList" />
  </div>
</template>

<script>
import { defineComponent, nextTick, onBeforeMount, reactive, ref, toRefs } from 'vue'

import { ElMessage, ElMessageBox } from 'element-plus'
import AddEdit from './components/add-edit.vue'

import usePage from '@/mixins/page'
import useDictionary from '@/mixins/dictionary'
import { clearJson } from '@/utils'

import { delApi, pageApi, pauseApi, resumeApi, runApi } from '@/api/develop/task'

export default defineComponent({
  components: { AddEdit },
  setup() {
    const refForm = ref()
    const refAddEdit = ref()

    const { page } = usePage()
    const { dictionaryMap, getDictionaryMap } = useDictionary()
    const data = reactive({
      loading: false,
      visible: false,
      form: {
        bean_name: ''
      },
      list: [],
      selection: []
    })

    const getList = () => {
      const params = {
        ...data.form,
        current: page.current,
        size: page.size
      }
      data.loading = true
      pageApi(params).then(r => {
        if (r) {
          data.list = r.data.list
          page.total = r.data.total
        }
        nextTick(() => {
          data.loading = false
        })
      })
    }

    /**
     * @description: 重新获取、重置 数据
     * @param {*}
     * @return {*}
     * @author: gumingchen
     */
    const reacquireHandle = () => {
      page.current = 1
      getList()
    }

    /**
     * @description: 新增/编辑弹窗
     * @param {*}
     * @return {*}
     * @author: gumingchen
     */
    const addEditHandle = id => {
      data.visible = true
      nextTick(() => {
        refAddEdit.value.init(id)
      })
    }

    /**
     * @description: 删除
     * @param {number} id
     * @return {*}
     * @author: gumingchen
     */
    const delHandle = id => {
      let params
      if (id) {
        params = [id]
      } else {
        params = data.selection.map(item => item.id)
      }
      ElMessageBox.confirm(`确定对[id=${ params.join(',') }]进行[${ id ? '删除' : '批量删除' }]操作?`, '提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        delApi(params).then(r => {
          if (r) {
            ElMessage({
              message: '操作成功!',
              type: 'success'
            })
            getList()
          }
        })
      }).catch(() => {
        // to do something on canceled
      })
    }

    /**
     * @description: 立即执行
     * @param {number} id
     * @return {*}
     * @author: gumingchen
     */
    const runHandle = id => {
      let params
      if (id) {
        params = [id]
      } else {
        params = data.selection.map(item => item.id)
      }
      ElMessageBox.confirm(`确定对[id=${ params.join(',') }]进行[${ id ? '立即执行' : '批量立即执行' }]操作?`, '提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        runApi(params).then(r => {
          if (r) {
            ElMessage({
              message: '操作成功!',
              type: 'success'
            })
            getList()
          }
        })
      }).catch(() => {
        // to do something on canceled
      })
    }

    /**
     * @description: 恢复
     * @param {number} id
     * @return {*}
     * @author: gumingchen
     */
    const resumeHandle = id => {
      let params
      if (id) {
        params = [id]
      } else {
        params = data.selection.map(item => item.id)
      }
      ElMessageBox.confirm(`确定对[id=${ params.join(',') }]进行[${ id ? '恢复' : '批量恢复' }]操作?`, '提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        resumeApi(params).then(r => {
          if (r) {
            ElMessage({
              message: '操作成功!',
              type: 'success'
            })
            getList()
          }
        })
      }).catch(() => {
        // to do something on canceled
      })
    }

    /**
     * @description: 暂停
     * @param {number} id
     * @return {*}
     * @author: gumingchen
     */
    const pauseHandle = id => {
      let params
      if (id) {
        params = [id]
      } else {
        params = data.selection.map(item => item.id)
      }
      ElMessageBox.confirm(`确定对[id=${ params.join(',') }]进行[${ id ? '暂停' : '批量暂停' }]操作?`, '提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        pauseApi(params).then(r => {
          if (r) {
            ElMessage({
              message: '操作成功!',
              type: 'success'
            })
            getList()
          }
        })
      }).catch(() => {
        // to do something on canceled
      })
    }

    /**
     * @description: table多选事件
     * @param {*} val
     * @return {*}
     * @author: gumingchen
     */
    const selectionHandle = val => {
      data.selection = val
    }

    /**
     * @description: 分页变化事件
     * @param {*}
     * @return {*}
     * @author: gumingchen
     */
    const pageChangeHandle = argPage => {
      page.current = argPage.current
      page.size = argPage.size
      getList()
    }

    onBeforeMount(() => {
      getList()
      getDictionaryMap('status')
    })

    return {
      refForm,
      refAddEdit,
      page,
      dictionaryMap,
      ...toRefs(data),
      getList,
      reacquireHandle,
      addEditHandle,
      delHandle,
      runHandle,
      resumeHandle,
      pauseHandle,
      selectionHandle,
      pageChangeHandle,
      clearJson
    }
  }
})
</script>

<style lang="scss" scoped>
</style>
