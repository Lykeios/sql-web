<!--  -->
<template>
  <div>
    <!-- 面包屑导航 -->
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item>学生页</el-breadcrumb-item>
      <el-breadcrumb-item
        ><span style="font-weight:bolder">作答情况</span></el-breadcrumb-item
      >
    </el-breadcrumb>

    选择实验:
    <el-select v-model="value" placeholder="请选择">
      <el-option
        v-for="item in options"
        :key="item.value"
        :label="item.label.name"
        :value="item.label.id"
      >
      </el-option>
    </el-select>
      <span style="margin-left: 100px">成绩构成:</span>
      <div id="grade-pie" style="width: 100px;height:100px;display: inline-block;margin-left: 20px;"></div>

    <!-- 显示区域 -->
    <el-card
      class="box-card"
      v-if="value != ''"
      style="margin-top:10px;min-height:300px"
    >
      <div slot="header" class="clearfix">
        <span>实验名称:{{ exp_name }}</span>
      </div>
      <div v-if="show == true">
        <div>
          <el-row>
            <el-col :span="3">
              <span style="font-weight:bolder;color:#409EFF;font-size:24px"
              >成绩:</span
              >
            </el-col>
            <el-col :span="10">
              <el-progress
                      :text-inside="true"
                      :stroke-width="26"
                      :percentage="grade"
                      style="width: 50%"
              ></el-progress>
            </el-col>
            <el-col :span="11">
              <span style="font-weight:bolder;color:#409EFF;font-size:24px"
              >提交时间: {{ group[0].subTime }}</span
              >
            </el-col>
          </el-row>
          <el-divider></el-divider>

          <div v-for="(index, i) in group" :key="i" style="margin-top:20px">
            <el-input v-model="index.test" disabled
              ><template slot="prepend">问题</template></el-input
            >
            <el-input v-model="index.answer" style="margin-top:5px" disabled
              ><template slot="prepend">作答</template>
              <template slot="append">
                <i
                  class="el-icon-close"
                  title="错误"
                  v-if="index.isCorrect == 0"
                ></i>
                <i
                  class="el-icon-check"
                  v-if="index.isCorrect == 1"
                  title="正确"
                ></i> </template
            ></el-input>
            <el-input v-model="index.mark" disabled
            ><template slot="prepend">本题分值</template></el-input
            >

            <el-divider></el-divider>

            <!-- <el-input v-model="index.answer"></el-input> -->
          </div>
        </div>
      </div>
      <div v-if="noMsg == true">该实验尚未完成</div>
    </el-card>
  </div>
</template>

<script>
export default {
  data() {
    return {
      id: window.localStorage.getItem('user_id'),
      teacher_id: window.localStorage.getItem('teacher_id'),
      value: '',
      exp_name: '',
      options: [],
      noMsg: false,
      show: false,
      grade: '',
      group: [],
      mark : [],
      test : [],
      answer : [],
      isCorrect : [],
      subTime : [],
      echartsOption : {
        backgroundColor: '#ffffff',

        tooltip: {
          trigger: 'item',
          formatter: '{a} <br/>{b} : {c} ({d}%)'
        },

        visualMap: {
          show: false,
          min: 0,
          max: 100,
          inRange: {
            colorLightness: [0, 1]
          }
        },
        series: [
          {
            name: '成绩构成',
            type: 'pie',
            radius: '70%',
            center: ['50%', '50%'],
            data: [],
            // data: [
            //   {value: 335, name: '直接访问'},
            //   {value: 310, name: '邮件营销'},
            //   {value: 274, name: '联盟广告'},
            //   {value: 235, name: '视频广告'},
            //   {value: 400, name: '搜索引擎'}
            // ].sort(function (a, b) { return a.value - b.value; }),
            roseType: 'radius',
            label: {
              color: 'rgba(255, 255, 255, 0.3)'
            },
            labelLine: {
              lineStyle: {
                color: 'rgba(255, 255, 255, 0.3)'
              },
              smooth: 0.2,
              length: 10,
              length2: 20
            },
            itemStyle: {
              color: '#c23531',
              shadowBlur: 200,
              shadowColor: 'rgba(0, 0, 0, 0.1)'
            },

            animationType: 'scale',
            animationEasing: 'elasticOut',
            animationDelay: function (idx) {
              return Math.random() * 200;
            }
          },
        ]
      }
    };
  },
  methods: {},
  watch: {
    async value(val) {

      //   请求实验数据;
      this.noMsg = false;
      this.show = false;
      this.group = [];
      this.subTime = null;
      const { data: res } = await this.$http.post('student/getGrade', {
        id: this.id,
        test_id: this.value
      });
      //   状态码判断
      if (res.status == 202) {
        this.$message.error('登录状态已失效!请先登录');
        // 重定向到登录页
        this.$router.push('/login');
      } else if (res.status == 200) {
        //   渲染数据
        this.show = true;
        this.exp_name = res.data.exp_name;
        this.group = res.data.output;
        this.grade = res.data.full_mark;

        var echarts = require('echarts');
        var myChart = echarts.init(document.getElementById('grade-pie'));

        for(let i =0;i<this.group.length;i++){
          let temp = {};
          temp.value = this.group[i].mark;
          temp.name = this.group[i].test;
          this.echartsOption.series[0].data.push(temp);
        }
        myChart.setOption(this.echartsOption);
      } else if (res.status == 201) {
        // 未完成
        this.noMsg = true;
        this.$message({ type: 'warning', message: '该实验尚未完成' });
      } else if (res.status == 400) {
        this.$message.error('服务器错误');
        // 重定向到主页
        this.$router.push('/student/home');
      }
    }
  },
  async created() {

    //   获取实验列表
    if (this.id == null) {
      // 重定向到登录页
      this.$router.push('/login');
    }
    // 获取当前可见的实验列表
    const { data: res } = await this.$http.post('getVisibleExperiment', {
      student_id: this.id
    });
    // 状态码判断
    if (res.status == 202) {
      this.$message.error('登录状态已失效!请先登录');
      // 重定向到登录页
      this.$router.push('/login');
    } else if (res.status == 200) {
      this.options = [];
      for (let i = 0; i < res.data.length; i++) {
        let obj = {
          label: res.data[i],
          value: res.data[i]
        };
        this.options.push(obj);
      }
    } else {
      return this.$message.error('未知错误');
    }
  }
};
</script>
<style scoped></style>
