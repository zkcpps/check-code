<template>
  <div>
    <el-drawer
      append-to-body
      direction="rtl"
      :title="title"
      :visible.sync="visible"
      :before-close="handleClose"
      destroy-on-close
      size="960px"
    >
      <div class="add-class-content">
        <el-form ref="formValue" :model="formValue" label-width="106px">
          <el-form-item
            label="坐席小组名称"
            prop="name"
            :rules="[
              {
                required: true,
                validator: validNameInput,
              },
            ]"
          >
            <div class="my-input">
              <el-input
                placeholder="请输入"
                v-model="formValue.name"
                maxlength="8"
                show-word-limit
              ></el-input>
            </div>
          </el-form-item>
          <el-form-item
            label="接待上限"
            prop="receptionMaxLimit"
            :rules="[{ required: true, message: '接待上限不能为空' }]"
          >
            <div class="my-input-number">
              <el-input-number
                v-model="formValue.receptionMaxLimit"
                :min="1"
                :step="1"
                :precision="0"
              ></el-input-number>
              <div class="help">坐席小组每个接待人员的接待上限</div>
            </div>
          </el-form-item>
          <el-form-item
            label="排班规则"
            prop="workingScheduleType"
            :rules="[{ required: true, message: '请选择排班规则' }]"
          >
            <div class="my-radio">
              <el-radio-group
                :disabled="!!form.id"
                v-model="formValue.workingScheduleType"
                @change="handleClickWorkingScheduleType"
              >
                <el-radio :label="2">非固定排班规则</el-radio>
                <el-radio :label="1">固定排班规则</el-radio>
              </el-radio-group>
            </div>
          </el-form-item>
        </el-form>
        <!-- 排班规则 固定排班规则 -->
        <div v-if="formValue.workingScheduleType === 1" class="class-rule">
          <el-form label-width="93px" ref="gdForm" :model="gdForm">
            <el-form-item
              label="接待日"
              prop="fixed_week"
              :rules="[{ required: true, message: '接待日不能为空' }]"
            >
              <div class="my-checkbox">
                <el-checkbox-group v-model="gdForm.fixed_week">
                  <el-checkbox
                    v-for="item in receipt_day"
                    :key="item"
                    :label="item"
                    >{{ item }}</el-checkbox
                  >
                </el-checkbox-group>
              </div>
            </el-form-item>
            <el-form-item
              label="接待时间"
              prop="date"
              :rules="[{ required: true, validator: validTime }]"
            >
              <div class="my-time-picker">
                <el-time-picker
                  style="width: 253px; border-radius: 2px; height: 32px"
                  prefix-icon="el-icon-timer"
                  format="HH:mm"
                  is-range
                  v-model="gdForm.date"
                  range-separator="至"
                  start-placeholder="开始时间"
                  end-placeholder="结束时间"
                >
                </el-time-picker>
              </div>
            </el-form-item>
            <el-form-item
              label="接待人员"
              prop="reception_person"
              :rules="[{ required: true, validator: validPerson }]"
            >
              <div class="my-add-user">
                <add-user v-model="gdForm.reception_person"></add-user>
              </div>
            </el-form-item>
          </el-form>
        </div>
        <!-- 排班规则 非固定排班规则 -->
        <div
          v-if="formValue.workingScheduleType === 2"
          class="class-rule-table"
        >
          <div class="table-header">
            <week-picker
              :value="currentWeek"
              :changeValue="handleChangeWeekValue"
            ></week-picker>
            <div class="table-header-opetetor">
              <el-button type="text" size="mini" @click="handleBatchClassRule"
                >批量添加</el-button
              >
            </div>
          </div>
          <class-table
            :loading="tableLoading"
            :date="getCurrentWeekDay"
            :clickEdit="handleAddClassRule"
            :dataSource="currentWeekDataSource"
          ></class-table>
        </div>
      </div>
      <div class="drawer-footer">
        <el-button style="font-size: 14px; padding: 9px 16px" @click="close"
          >取 消</el-button
        >
        <el-button
          style="font-size: 14px; padding: 9px 16px"
          type="primary"
          @click="handleAddSeat('formValue')"
          :loading="sumbitLoadding"
          >确 定</el-button
        >
      </div>
      <add-classrule-dialog
        title="添加排班规则"
        :visible="addClassRuleVisible"
        :close="handleClassRuleDialogClose"
        :form="addClassRuleInfoForm"
        :submit="handleClassRuleDialogSumbit"
        :classList="classList"
      ></add-classrule-dialog>
      <batch-classrule-dialog
        title="添加排班规则"
        :visible="batchClassRuleVisible"
        :close="handleBatchClassRuleDialogClose"
        :form="batchClassRuleInfoForm"
        :submit="handleBatchClassRuleDialogSumbit"
        :classList="classList"
      ></batch-classrule-dialog>

      <!-- 提醒弹框 -->
      <tip-dialog :visible="tipDialogVisible" :content="tipContent">
        <template slot="footer">
          <el-button
            style="font-size: 14px; padding: 9px 16px"
            @click="handleCloseTip"
            >取 消</el-button
          >
          <el-button
            style="font-size: 14px; padding: 9px 16px"
            type="primary"
            @click="goToAddClass"
            >去添加</el-button
          >
        </template>
      </tip-dialog>
    </el-drawer>
  </div>
</template>

<script>
import { CheckboxGroup } from "element-ui";
import AddUser from "../add-user.vue";
import WeekPicker from "./week-picker.vue";
import ClassTable from "./class-table.vue";
import AddClassruleDialog from "./add-classrule-dialog.vue";
import BatchClassruleDialog from "./batch-classrule-dialog.vue";
import TipDialog from "../tip-dialog.vue";
import * as seatApi from "@/api/weixin-customer/seat-manage.js";
import { isSpecialCharacters } from "@/util/input";

import moment from "moment";

export default {
  name: "add-seat-drawer",
  props: {
    title: {
      type: String,
    },
    visible: {
      type: Boolean,
      default: false,
    },
    close: {
      type: Function,
      default: () => {},
    },
    form: {
      type: Object,
      default: () => {},
    },
    submit: {
      type: Function,
    },
    openExtral: {
      type: Function,
    },
    classData: {
      type: Array,
    },
    sumbitLoadding: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      handleDateDatas: [], // 操作过的天数（非固定排班用）
      formValue: {},
      gdForm: {
        reception_person: [],
        fixed_week: ["周一", "周二", "周三", "周四", "周五"],
        date: [
          moment(moment().format("YYYY-MM-DD ") + "09:00"),
          moment(moment().format("YYYY-MM-DD ") + "18:30"),
        ],
      },
      receipt_day: ["周一", "周二", "周三", "周四", "周五", "周六", "周日"],
      currentWeek: moment().add(moment().day() ? 8 - moment().day() : 1, "day"), // 获取下周一日期
      pickerOptions: {
        firstDayOfWeek: 1,
      },
      // 排班列表数据
      classTableList: {},
      // 排班列表
      classList: [],
      // 当前选中周列表数据
      currentWeekDataSource: [],
      tableLoading: false,

      // 排班规则
      addClassRuleVisible: false,
      addClassRuleInfoForm: {},
      // 批量排班规则
      batchClassRuleVisible: false,
      batchClassRuleInfoForm: {},

      // 弹窗提醒
      tipDialogVisible: false,
      tipContent: "",

      // 缓存批量排班参数
      batchFormValues: {},
      validNameInput: (rele, value, callback) => {
        if (value && isSpecialCharacters(value)) {
          callback(new Error("坐席小组名称不能包含特殊符号"));
        }
        if (value && value.length > 8) {
          callback(new Error("坐席小组名称长度不能大于8位"));
        }
        if (!value) {
          callback(new Error("坐席小组名称长度不能为空"));
        }
        callback();
      },
      validTime: (rele, value1, callback) => {
        const value = this.gdForm.date;
        if (!value) {
          callback(new Error("接待时间不能为空"));
        }
        const ymd = moment().format("YYYY/MM/DD ");
        if (
          moment(ymd + moment(value[1]).format("HH:mm:ss")).isBefore(
            ymd + moment(value[0]).format("HH:mm:ss")
          )
        ) {
          callback(new Error("开始时间不能晚于结束时间"));
        }
        callback();
      },
      validPerson: (rele, value, callback) => {
        const person = this.gdForm.reception_person;
        if (person.length === 0) {
          callback(new Error("接待人员不能为空"));
        }
        callback();
      },
    };
  },
  created() {},
  mounted() {},
  watch: {
    currentWeek(v) {
      console.log("改变currentWeek---", moment(v).format("YYYY-MM-DD"));
      if (!this.visible) return;
      this.createClassUserList(v);
      // 修改
      if (this.form.id) {
        this.getSeatDetail();
      }
      // 新增
      if (!this.form.id) {
        this.getTableData();
      }
    },
    visible(v) {
      console.log(v ? "显示-----" : "隐藏-----");
      this.handleDateDatas = [];
      if (!this.formValue.name && this.$refs["formValue"]) {
        this.$refs["formValue"].resetFields();
        // this.currentWeekDataSource = [];
        // this.classTableList = {};
      }
      if (!v) {
        this.$refs["formValue"] && this.$refs["formValue"].resetFields();
        this.$refs["gdForm"] && this.$refs["gdForm"].resetFields();
        this.currentWeekDataSource = [];
        this.classTableList = {};
        this.currentWeek = moment().add(
          moment().day() ? 8 - moment().day() : 1,
          "day"
        );
      }
      if (v) {
        // this.currentWeek = moment().add(7, "day").day()
        //   ? moment().add(moment().add(7, "day").day(), "day")
        //   : moment().add(moment().add(7, "day").day(), "day");
        if (this.form.id) {
          // this.currentWeekDataSource = [];
          // this.classTableList = {};
          this.getSeatDetail();
        } else {
          this.$refs["formValue"] && this.$refs["formValue"].resetFields();
          this.$refs["gdForm"] && this.$refs["gdForm"].resetFields();
          // this.currentWeekDataSource = [];
          // this.classTableList = {};
        }
      }
      console.log(
        v ? "显示" : "隐藏",
        "------classTableList",
        this.classTableList
      );
      console.log(
        v ? "显示" : "隐藏",
        "------currentWeekDataSource",
        this.currentWeekDataSource
      );
    },
    form(v) {
      this.formValue = {
        ...v,
      };
      // if (!this.gdForm["reception_person"])
      //   this.gdForm["reception_person"] = [];
      // if (!this.gdForm["fixed_week"]) this.gdForm["fixed_week"] = [];
      // console.log("this.formValue", this.formValue);
    },
    tipDialogVisible(v) {
      if (!v) this.tipContent = "";
    },
    classData(v) {
      this.classList = v;
    },
    gdForm: {
      deep: true,
      handler(newVal, oldVal) {
        // if (newVal.reception_person.length) {
        if (
          !(
            newVal.reception_person.length == 0 &&
            oldVal.reception_person.length == 0
          )
        ) {
          if (this.$refs["gdForm"]) this.$refs["gdForm"].validate();
        }
        // }
      },
    },
  },
  components: {
    AddUser,
    CheckboxGroup,
    WeekPicker,
    ClassTable,
    AddClassruleDialog,
    BatchClassruleDialog,
    TipDialog,
  },
  // created() {
  //   this.createClassUserList(this.currentWeek);
  // },
  methods: {
    // 点击排班规则
    handleClickWorkingScheduleType(e) {
      this.formValue.workingScheduleType = e;
      if (e == "1") {
        // this.gdForm = {
        //   reception_person: [],
        //   fixed_week: ["周一", "周二", "周三", "周四", "周五"],
        //   date: [
        //     moment(moment().format("YYYY-MM-DD ") + "09:00"),
        //     moment(moment().format("YYYY-MM-DD ") + "18:30"),
        //   ],
        // };
      }
      if (e == "2") {
        this.getTableData();
      }
      // }
    },
    // 根据日期初始化
    createClassUserList(current) {
      const startDay = moment(current).format("YYYY-MM-DD");
      const keyValue = this.classTableList;
      if (!keyValue[startDay]) {
        for (let i = 0; i < 7; i++) {
          const key = moment(startDay).add(i, "day").format("YYYY-MM-DD");
          if (!keyValue[key] && key !== undefined) {
            keyValue[key] = this.classList.map((item) => {
              return { ...item, userList: [] };
            });
          }
        }
        this.classTableList = keyValue;
        console.log(
          "根据日期生成createClassUserList---------",
          this.classTableList
        );
        // 新增需要生成
        if (!this.form.id) {
          this.getTableData();
        }
      }
    },

    handleClose() {
      this.$refs["formValue"].resetFields();
      this.close();
      this.gdForm = {
        reception_person: [],
        fixed_week: ["周一", "周二", "周三", "周四", "周五"],
        date: [
          moment(moment().format("YYYY-MM-DD ") + "09:00"),
          moment(moment().format("YYYY-MM-DD ") + "18:30"),
        ],
      };
    },
    handleAddSeat(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          if (this.formValue.workingScheduleType === 1) {
            this.$refs["gdForm"].validate();
            if (this.gdForm.reception_person.length === 0) {
              return false;
            }
            //校验时间
            if (!this.gdForm.date) {
              return false;
            }
            const ymd = moment().format("YYYY/MM/DD ");
            if (
              moment(
                ymd + moment(this.gdForm.date[1]).format("HH:mm:ss")
              ).isBefore(ymd + moment(this.gdForm.date[0]).format("HH:mm:ss"))
            ) {
              this.$refs["gdForm"].validate();
              return false;
            }
          }
          const param = this.handleParams(this.formValue);
          // 新增情况下，排班信息不能为空
          if (!param.id && param.workingScheduleType === 2) {
            const filterRules = param.ruleList.filter((item) => {
              return item.userList.length > 0;
            });
            param.ruleList = filterRules;
          }
          // 检测开始时间和结束时间 固定排班
          if (param.workingScheduleType === 1) {
            const fixedWorkingSchedule = param.fixedWorkingSchedule;
            const { receptionStartTime, receptionEndTime } =
              fixedWorkingSchedule;
            const ymd = moment().format("YYYY/MM/DD ");
            if (
              moment(
                ymd + moment(receptionEndTime).format("HH:mm:ss")
              ).isBefore(ymd + moment(receptionStartTime).format("HH:mm:ss"))
            ) {
              this.$refs["gdForm"].validate();
              return false;
            }
          }

          if (
            !param.id &&
            param.workingScheduleType === 2 &&
            !param.ruleList.length
          ) {
            this.$message({
              type: "error",
              message: "请添加排班人员",
            });
            return false;
          } else {
            this.submit(param);
          }
        } else {
          return false;
        }
      });
    },
    //获取排班详情
    async getSeatDetail() {
      console.log("获取排班详情getSeatDetail---------", this.form);
      // console.log(this.form.id);
      // console.log(this.currentWeek);
      const date = moment(this.currentWeek).format("YYYY-MM-DD");
      const startDate = date;
      const endDate = moment(date).add(6, "day").format("YYYY-MM-DD");
      const params = {
        id: this.form.id,
      };
      if (this.form.workingScheduleType === 2) {
        const flag = this.currentWeekHasClass();
        if (flag) {
          this.getTableData();
          return;
        }
        params.startDate = startDate;
        params.endDate = endDate;
      }
      let res;
      try {
        this.tableLoading = true;
        res = await seatApi.getSeatDetail(params);
        this.tableLoading = false;
      } catch (error) {
        res = false;
        this.tableLoading = false;
      }

      // 数据查询错误
      if (!res) {
        console.log("数据查询错误-----");
        this.getTableData();
        return;
      }

      const detail = res.data.data;
      // 固定排班回显
      if (detail.workingScheduleType === 1) {
        const { fixedWorkingSchedule } = detail;

        const uniqueUserList = fixedWorkingSchedule.receptionUserLst.filter(
          (item, index, self) => {
            const i = self.findIndex((t) => t.userId === item.userId);
            return i === index;
          }
        );

        this.gdForm = {
          date: [
            moment(
              moment().format("YYYY-MM-DD ") +
                fixedWorkingSchedule.receptionStartTime +
                ":00"
            ),
            moment(
              moment().format("YYYY-MM-DD ") +
                fixedWorkingSchedule.receptionEndTime +
                ":00"
            ),
          ],
          reception_person: uniqueUserList.map((item) => ({
            id: item.userId,
            userId: item.userId,
            userName: item.userName,
            name: item.userName,
            avatar: "/img/transition/defaultAvater.png",
          })),
          fixed_week: fixedWorkingSchedule.receptionWeekList.map((item) => {
            return this.receipt_day[
              [
                "星期一",
                "星期二",
                "星期三",
                "星期四",
                "星期五",
                "星期六",
                "星期日",
              ].indexOf(item)
            ];
          }),
        };
        this.formValue = {
          ...detail,
        };
        if (this.$refs["gdForm"]) this.$refs["gdForm"].validate();
        // this.gdForm = {
        //   ...formValues,
        // };
        // console.log("this.gdForm", this.gdForm);
      } else {
        // 非固定排班回显
        this.handleStaticClassData(detail);
      }
    },
    handleStaticClassData(detail) {
      console.log("回显静态数据handleStaticClassData");
      // 总体对象 this.classTableList
      // 表格数据 this.currentWeekDataSource
      const currentDay = moment(this.currentWeek).format("YYYY-MM-DD");
      const currrentDays = [];
      for (let i = 0; i < 7; i++) {
        const day = moment(currentDay).add(i, "day").format("YYYY-MM-DD");
        currrentDays.push(day);
      }
      // 第一次回显如果没有数据则构建
      if (!this.classTableList[currentDay])
        this.createClassUserList(this.currentWeek);
      const classTableList = { ...this.classTableList };
      // console.log("classTableList", classTableList);

      const userList = detail.userList;

      //将人员筛到前端保存的对象中
      if (JSON.stringify(classTableList) !== "{}") {
        userList.forEach((item) => {
          const { userName: name, userId, workingSchedule } = item;
          workingSchedule.forEach((week, index) => {
            if (week[0].id !== -1) {
              // const classNames = week.split("、");
              week.forEach((c) => {
                if (c) {
                  let indexOfUserList = this.classList.findIndex(
                    (item) => item.id === c.id
                  );
                  // // 判断现有班次名称是否与后端返回一致，否则以后端返回为准
                  // if (indexOfUserList !== -1) {
                  //   if (this.classList[indexOfUserList].name !== c.name) {
                  //     classTableList[currrentDays[index]][
                  //       indexOfUserList
                  //     ].name = c.name;
                  //   }
                  // }

                  if (indexOfUserList === -1) {
                    const len = classTableList[currrentDays[index]].length;
                    classTableList[currrentDays[index]][len] = {
                      id: c.id,
                      name: c.name,
                      userList: [],
                    };
                    indexOfUserList = len; // 记录位置
                  }
                  // const currentDayClassUserList =
                  //   classTableList[currrentDays[index]][indexOfUserList]
                  //     .userList;
                  // console.log("currentDayClassUserList", currentDayClassUserList);

                  // const hasCurrentUser = currentDayClassUserList.findIndex(
                  //   (item) => item.id === userId
                  // );

                  // if (hasCurrentUser === -1) {
                  classTableList[currrentDays[index]][
                    indexOfUserList
                  ].userList.push({
                    name,
                    userId,
                    label: name,
                    userName: name,
                    id: userId,
                    className: c.name,
                  });
                  // }
                }
              });
            }
          });
        });
      }

      // console.log("classTableList", classTableList);
      this.classTableList = { ...classTableList };
      this.getTableData();
      this.formValue = {
        ...detail,
      };
    },
    //判断当前选中日期是否有数据，有数据以当前时间为准
    currentWeekHasClass() {
      console.log("判断当前选中日期是否有数据currentWeekHasClass");
      let flag = false;
      const currentDay = moment(this.currentWeek).format("YYYY/MM/DD");
      const currrentDays = [];
      const classTableList = { ...this.classTableList };
      for (let i = 0; i < 7; i++) {
        const day = moment(currentDay).add(i, "day").format("YYYY-MM-DD");
        currrentDays.push(day);
      }
      if (JSON.stringify(classTableList) !== "{}") {
        currrentDays.some((item) => {
          const curentDayClass = classTableList[item];
          if (flag) return true;
          curentDayClass.some((c) => {
            if (c.userList.length > 0) {
              flag = true;
              return true;
            }
          });
        });
      }
      if (flag) console.log("这周数据前端已经缓存有数据过了----");
      return flag;
    },
    // 处理提交参数
    handleParams(value) {
      // console.log("values", value);
      // console.log("table", this.classTableList);
      const weekMap = [
        "星期日",
        "星期一",
        "星期二",
        "星期三",
        "星期四",
        "星期五",
        "星期六",
      ];
      const data = { ...this.classTableList };
      const params = {
        name: value.name,
        receptionMaxLimit: value.receptionMaxLimit,
        workingScheduleType: value.workingScheduleType,
      };
      if (this.form.id) {
        params.id = this.form.id;
      }
      if (value.workingScheduleType === 1) {
        // 固定排班
        const fixedWorkingSchedule = {
          receptionWeekList: this.gdForm.fixed_week,
          receptionStartTime: moment(this.gdForm.date[0]).format("HH:mm:ss"),
          receptionEndTime: moment(this.gdForm.date[1]).format("HH:mm:ss"),
          receptionUserLst: this.gdForm.reception_person.map((item) => ({
            userId: item.id,
            userName: item.name,
          })),
        };
        params.fixedWorkingSchedule = fixedWorkingSchedule;
      } else {
        // 非固定排班
        const ruleList = [];

        const handleDateDatasSet = [...new Set(this.handleDateDatas)];
        const currentClassIdList = this.classList.map((item) => item.id);
        Object.keys(data).forEach((v) => {
          console.log("v", v);
          console.log("handleDateDatas", handleDateDatasSet);
          if (handleDateDatasSet.includes(v)) {
            // 只有操作过的日期才保存
            let userLists = data[v];
            // 新增时，排班空信息的不必传
            const workingSchedule = {
              receptionDate: v + " 00:00:00",
              fixedWeek: weekMap[moment(v).day()],
            };
            const userList = [];
            userLists.forEach((item) => {
              if (item.userList.length > 0) {
                // 只插入现有班次的数据
                if (currentClassIdList.includes(item.id)) {
                  const users = item.userList;
                  users.forEach((u) => {
                    userList.push({
                      workingScheduleId: item.id,
                      userId: u.userId,
                      userName: u.name,
                      startTime: item.startTime,
                      endTime: item.endTime,
                    });
                  });
                }
              }
            });
            workingSchedule.userList = userList;
            ruleList.push(workingSchedule);
          }
        });

        params.ruleList = ruleList;
      }
      return params;
    },
    handleChangeWeekValue(val) {
      this.currentWeek = val;
    },
    // 排班规则
    handleClassRuleDialogClose() {
      this.addClassRuleVisible = false;
    },
    handleClassRuleDialogSumbit(value, date) {
      console.log("排班规则提交", value);
      this.handleDateDatas = [...this.handleDateDatas, date];
      if (date) {
        this.classTableList[date] = value;
      }
      this.getTableData();
    },
    handleAddClassRule(value) {
      // 检查是否有排班否则弹窗提醒创建
      if (this.classList.length === 0) {
        this.tipContent = "尚未添加班次，请先前往班次管理添加";
        this.tipDialogVisible = true;
        return;
      }
      if (!this.classTableList[value]) {
        this.createClassUserList(this.currentWeek);
      }
      const currentClassIdList = this.classList.map((item) => item.id);
      const currentDayUserList = [...this.classTableList[value]].filter(
        (item) => {
          return currentClassIdList.includes(item.id);
        }
      );
      this.addClassRuleInfoForm = {
        userList: currentDayUserList,
        date: value,
      };
      // const day = currentWeek
      this.addClassRuleVisible = true;
    },

    // 批量排班规则
    handleBatchClassRuleDialogClose() {
      this.batchClassRuleVisible = false;
    },
    handleBatchClassRuleDialogSumbit(value, date) {
      console.log("批量排班规则提交", value);
      // 记录操作过的日期数组
      this.handleDateDatas = [...this.handleDateDatas, ...Object.keys(value)];

      // console.log(
      //   "批量排班规则提交date------",
      //   moment(date).format("YYYY-MM-DD")
      // );
      const acceptDate = moment(date).format("YYYY/MM/DD");

      const currentDay = moment(this.currentWeek).format("YYYY/MM/DD");

      if (acceptDate !== currentDay) {
        this.currentWeek = date;
      }

      // 第一次回显如果没有数据则构建
      if (!this.classTableList[moment(this.currentWeek).format("YYYY-MM-DD")]) {
        this.createClassUserList(this.currentWeek);
      }

      const data = JSON.parse(JSON.stringify(this.classTableList));
      this.classTableList = { ...data, ...value };
      this.getTableData();
    },
    handleBatchClassRule(value) {
      // 检查是否有排班否则弹窗提醒创建;
      if (this.classList.length === 0) {
        this.tipContent = "尚未添加班次，请先前往班次管理添加";
        this.tipDialogVisible = true;
        return;
      }

      // console.log("点击排班规则表格编辑", value);
      const startDay = moment(this.currentWeek).format("YYYY/MM/DD");

      this.batchClassRuleInfoForm = {
        date: startDay,
        userList: this.currentWeekDataSource,
        classList: this.classList,
        classTableList: this.classTableList,
      };
      this.batchClassRuleVisible = true;
    },

    // 检查是否有添加班次，没有则弹出引导
    async checkHasClass() {
      if (this.classList.length > 0) {
        return true;
      } else {
        return false;
      }
    },

    // 批量添加排班规则检查是否有冲突
    checkClassRuleIsClash() {
      return true;
    },

    // 弹窗
    handleCloseTip() {
      this.tipDialogVisible = false;
    },
    goToAddClass() {
      this.tipDialogVisible = false;
      this.openExtral();
    },

    // 处理表格回显数据
    getTableData() {
      const currentDay = moment(this.currentWeek).format("YYYY/MM/DD");
      const userList = this.classTableList;
      const weekArr = [];
      let result = [];
      for (let i = 0; i < 7; i++) {
        const day = moment(currentDay).add(i, "day").format("YYYY-MM-DD");
        weekArr.push(moment(currentDay).add(i, "day").format("YYYY-MM-DD"));
        if (userList[day])
          result = [
            ...result,
            ...userList[day].map((item) => ({
              id: item.id,
              name: item.name,
              userList: item.userList,
              date: day,
            })),
          ];
      }
      let dataSource = [];
      for (let i = 0; i < result.length; i++) {
        let data = result[i];
        const { userList, date, id, name } = data;
        for (let j = 0; j < userList.length; j++) {
          const user = userList[j];
          const index = dataSource.findIndex((item) => item.id === user.id);
          if (index !== -1) {
            const userData = dataSource[index];
            const content = userData.content;
            const dateIndex = weekArr.indexOf(date);
            if (content[dateIndex] === "休息") {
              content[dateIndex] = user.className || name;
            } else {
              content[dateIndex] += `、${user.className || name}`;
            }
            dataSource[index] = { ...userData, content };
          } else {
            const content = Array(7).fill("休息");
            const dateIndex = weekArr.indexOf(date);
            content[dateIndex] = user.className || name;
            dataSource.push({
              id: user.id,
              userId: user.userId,
              label: user.label,
              content: content,
              name: user.name,
            });
          }
        }
      }
      // console.log("dataSource", dataSource);
      this.currentWeekDataSource = dataSource;

      console.log("处理表格回显数据getTableData", this.currentWeekDataSource);
    },

    // 优化
    
  },
  computed: {
    getCurrentWeekDay() {
      return moment(this.currentWeek).format("YYYY-MM-DD");
    },
  },
};
</script>

<style lang="scss" scoped>
// 弹窗
::v-deep .el-drawer__header {
  height: 24px;
  line-height: 24px;
  padding: 24px;
  background: #fff;
  border-radius: 4px 4px 0px 0px;
  border-bottom: 1px solid #e6eaf2;
  color: #292c33;
  font-weight: 600;
  margin-bottom: 0;
}
::v-deep .el-drawer__header > :first-child {
  font-size: 18px;
  font-weight: 600;
  color: #292c33;
}

::v-deep .el-drawer__headerbtn {
  top: 30px;
  right: 31px;
}
::v-deep .el-drawer__headerbtn .el-drawer__close {
  color: #292c33;
  font-weight: 600;
}

::v-deep .el-drawer__body {
  padding-top: 24px;
}

::v-deep .el-drawer__footer {
  padding: 24px;
  border-top: 1px solid #e6eaf2;
}

.add-class-content {
  padding: 0 24px;
}

.add-class-content ::v-deep .el-form-item {
  margin-bottom: 24px;
}

.drawer-footer {
  position: absolute;
  width: 100%;
  text-align: end;
  bottom: 0;
  padding: 24px;
  border-top: 1px solid #e6eaf2;
}

// 时间选择
.my-time-picker ::v-deep .el-date-editor .el-range__icon {
  line-height: 26px;
}
.my-time-picker ::v-deep .el-date-editor .el-range__close-icon {
  line-height: 26px;
}

.my-input ::v-deep .el-input__inner {
  width: 295px;
  border-radius: 4px;
  height: 32px;
}
.my-input ::v-deep .el-input {
  width: 295px;
}

.my-input-number ::v-deep .el-input-number .el-input__inner {
  width: 136px;
  height: 32px;
  border-radius: 2px;
}

.my-input-number ::v-deep .el-input-number__decrease,
.my-input-number ::v-deep .el-input-number__increase {
  height: 30px;
  width: 33px;
  top: 4px;
  line-height: 32px;
  background: #f7f9fc;
}

.my-input-number ::v-deep .el-input-number__increase {
  left: 101px;
  top: 4px;
}

.my-input-number ::v-deep .el-input-number__decrease {
  top: 4px;
}

.my-input-number {
  .help {
    height: 22px;
    font-size: 13px;
    font-weight: normal;
    line-height: 22px;
    /* 文字色/Placeholder */
    color: #b8becc;
  }
}

.class-rule {
  width: 807px;
  margin-left: 105px;
  background: #f7f9fc;
  padding: 8px 0 24px;
}

.class-rule-table {
  width: 807px;
  // height: 388px;
  border-radius: 4px;
  box-sizing: border-box;
  /* 边框色/Border Base */
  /* 样式描述：线框、输入框默认文字、禁用置灰 */
  border: 1px solid #dadee5;
  padding: 0px 12px 12px 12px;
  margin-left: 105px;
  margin-bottom: 24px;

  .table-header {
    display: flex;
    justify-content: center;
    align-items: center;
    position: relative;
    height: 48px;
    text-align: center;
    margin: 0 auto;

    .table-header-opetetor {
      position: absolute;
      right: 0;
    }
  }
}
</style>
