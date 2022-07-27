<template>
  <div class="hello">
    <countdown :time="5 * 60 * 1000" :transform="transform">
      <template #default="{ hours, minutes, seconds }">
        {{ `${hours}:${minutes}:${seconds}` }}
      </template>
    </countdown>
    <el-row type="flex" align="middle" justify="center">
      <el-col :xs="24" :sm="12">
        <el-tabs v-model="activeName" type="card" style="text-align:left">
          <el-tab-pane label="随机答题" name="2">
            <template v-if="activeName === '2'">
              <div class="opt-btn" style="margin: 15px 0">
                <el-button type="primary" @click="handleRandom">随机答题</el-button>
                <el-button type="primary" @click="handleRandom">开始答题</el-button>
                <el-button type="primary" @click="handleRandom">重新答题</el-button>
                <div style="float: right; color: red; font-weight: bold;">
                  <span>{{ examineStatusMap[examineStatus].label }}</span>
                  <countdown ref="countdownRef" :time="countdownTime" :transform="transform">
                    <template #default="{ hours, minutes, seconds }">
                      {{ `${hours}:${minutes}:${seconds}` }}
                    </template>
                  </countdown>
                  <!-- <span>{{  }}</span> -->
                  <el-button type="primary" size="mini" @click="handleRandom">暂停时间</el-button>
                </div>
              </div>
              <el-collapse v-model="examineItemActiveName">
                <el-collapse-item v-for="(item, index) in examineList" :key="item.id" :name="item.id" :title="item.q">
                  <template v-slot:title>
                    <span style="margin-right: 5px; font-size: 12px; font-weight: bold;">{{ index + 1 }}. {{ item.q }}</span>
                    <div class="btn-icon-box" style="width: 90px">
                      <i class="el-icon-edit-outline" style="color: #409EFF; margin: 0 3px 0 0;" @click="handleEdit(item)"></i>
                      <i class="el-icon-delete" style="color: #f56c6c;margin: 0 3px;" @click="handleDelete(index)"></i>
                    </div>
                  </template>
                  <div v-html="item.a"></div>
                </el-collapse-item>
              </el-collapse>
            </template>
          </el-tab-pane>
          <el-tab-pane label="题目列表" name="1">
            <template v-if="activeName === '1'">
              <div class="opt-btn" style="margin: 15px 0">
                <el-button type="primary" size="small" @click="handleAdd">增加题目</el-button>
              </div>
              <el-collapse v-model="itemActiveName">
                <el-collapse-item v-for="(item, index) in list" :key="item.id" :name="item.id" :title="item.q">
                  <template v-slot:title>
                    <span style="margin-right: 5px; font-size: 12px; font-weight: bold;">{{ index + 1 }}. {{ item.q }}</span>
                    <div class="btn-icon-box" style="width: 90px">
                      <i class="el-icon-edit-outline" style="color: #409EFF; margin: 0 3px 0 0;" @click="handleEdit(item)"></i>
                      <i class="el-icon-delete" style="color: #f56c6c;margin: 0 3px;" @click="handleDelete(index)"></i>
                    </div>
                  </template>
                  <div v-html="item.a"></div>
                </el-collapse-item>
              </el-collapse>
            </template>
          </el-tab-pane>
          <el-tab-pane label="新增/编辑题目" name="3">
            <template v-if="activeName === '3'">
              <el-form :model="form" label-position="top">
                <el-form-item label="题目">
                  <el-input v-model="form.q" placeholder="请输入题目"></el-input>
                </el-form-item>
                <el-form-item label="答案">
                  <!-- <el-input v-model="form.a" type="textarea" :rows="10" placeholder=""></el-input> -->
                  <editor v-model="form.a" :init="initOptions"></editor>
                </el-form-item>
                <el-form-item label="" style="text-align: right">
                  <el-button type="default" @click="handleCancel">取消</el-button>
                  <el-button type="primary" @click="handleSave">保存</el-button>
                </el-form-item>
              </el-form>
            </template>
          </el-tab-pane>
        </el-tabs>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import VueCountdown from '@chenfengyuan/vue-countdown'
import Editor from '@tinymce/tinymce-vue'
import { cloneDeep, differenceBy } from 'lodash'
import { v4 as uuidv4 } from 'uuid'
const listCache = window.localStorage.getItem('list') || ''
const havedListCache = window.localStorage.getItem('havedList') || ''

export default {
  name: 'HelloWorld',
  components: {
    Editor,
    Countdown: VueCountdown
  },
  props: {
    msg: String
  },
  data() {
    return {
      initOptions: {
        language:'zh_CN',
        min_height: 400
      },
      activeName: '2',
      list: listCache ? JSON.parse(listCache) : [],
      havedList: havedListCache ? JSON.parse(havedListCache) : [],
      examineList: [],
      curItem: {},
      form: this.initForm(),
      itemActiveName: '',
      examineItemActiveName: '',
      countdownTime: 0,
      examineStatus: 0, //答题状态 0：未开始 1:准备，2：过渡阶段 3：答题 ；4：结束
      examineStatusMap: {
        0: { value: 0, label: '未开始', time: 0 },
        1: { value: 1, label: '备考倒计时：', time: 5 * 60 * 1000 },
        2: { value: 2, label: '准备答题倒计时：', time: 1 * 60 * 1000 },
        3: { value: 3, label: '答题倒计时：', time: 10 * 60 * 1000 },
        4: { value: 4, label: '结束答题', time: 0 * 60 * 1000 }
      }
    }
  },
  watch:{
    list: {
      deep: true,
      handler(value) {
        window.localStorage.setItem('list', JSON.stringify(value))
      }
    },
    havedList: {
      deep: true,
      handler(value) {
        window.localStorage.setItem('havedList', JSON.stringify(value))
      }
    },
    examineStatus(value) {
      this.countdownTime = this.examineStatusMap[value].time
    }
  },
  computed: {
    // 可用考试题
    validList() {
      return differenceBy(this.list, this.havedList, 'id')
    }
  },
  methods: {
    initForm() {
      return {
        id: '',
        q: '',
        a: ''
      }
    },
    transform(props) {
      Object.keys(props).forEach(key => {
        const value = props[key]
        props[key] = value < 10 ? `0${value}` : value
      })
      return props
    },
    handleAdd() {
      this.form = this.initForm()
      this.curItem = {}
      this.activeName = '3'
    },
    handleCancel() {
      this.form = this.initForm()
      this.curItem = {}
      this.activeName = '1'
    },
    handleSave() {
      const cloneForm = {
        ...this.form
      }
      if (cloneForm.id) {
        Object.assign(this.curItem, cloneForm)
      } else {
        cloneForm.id = uuidv4()
        this.list.push(cloneForm)
      }
      this.form = this.initForm()
      this.curItem = {}
      // this.activeName = '1'
    },
    handleEdit(item) {
      this.curItem = item
      this.form = { ...item }
      this.activeName = '3'
    },
    handleDelete(index) {
      this.list.splice(index, 1)
    },
    handleRandom() {
      const cloneValidList = cloneDeep(this.validList)
      if (cloneValidList.length < 3) {
        this.havedList = []
        this.handleRandom()
      } else {
        this.$nextTick(() => {
          const result = []
          while (result.length < 3) {
            const { random, floor } = Math
            const randomIndex = floor(random() * cloneValidList.length)
            const item = cloneValidList.splice(randomIndex, 1)
            result.push(...item)
          }
          this.examineList = result
          this.havedList.push(...result)
        })
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
::v-deep .el-collapse-item__header {
  line-height: normal;
  min-height: 48px;
  padding: 5px 0;
}

</style>
<style>
.tox-notifications-container {
  display: none !important;
}
</style>
