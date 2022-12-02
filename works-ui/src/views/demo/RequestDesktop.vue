<template>
  <div style="text-align: center">
    <img alt="newzen" src="@/assets/new4.png" style="width: 100%" />

    <a-button
      type="primary"
      shape="round"
      size="large"
      @click="
        () => {
          addModalView = true;
        }
      "
    >
      <template #icon>
        <PlusOutlined />
      </template>
      자원 생성 및 할당 신청
    </a-button>
    &nbsp;
    <a-button shape="round" size="large" @click="viewDrawer(true)">
      <template #icon>
        <fund-view-outlined />
      </template>
      자원 생성 및 할당 상태
    </a-button>
  </div>
  <img alt="newzen" src="@/assets/new3.png" style="width: 100%" />

  <a-drawer title="자원 생성 및 할당 상태" placement="bottom" height="20%" :visible="drawerView" @close="() => (drawerView = false)">
    <!-- <div v-if="wsDataList.length > 0" v-for="ws in wsDataList"> -->
    <div v-if="wsCount">
      <span style="font-size: 20px">비즈니스 명 : {{ wsName }}</span>
      <a-steps>
        <a-step :status="step1" title="워크스페이스 생성">
          <template #icon>
            <CloudOutlined />
          </template>
        </a-step>
        <a-step :status="step2" title="사용자 계정 생성">
          <template #icon>
            <UserOutlined />
          </template>
        </a-step>
        <a-step :status="step3" title="워크스페이스 사용자 추가">
          <template #icon>
            <TeamOutlined />
          </template>
        </a-step>
        <a-step :status="step4" title="테스트 가상머신 생성">
          <template #icon>
            <DesktopOutlined />
          </template>
        </a-step>
        <a-step :status="step5" title="데스크톱 생성">
          <template #icon>
            <LaptopOutlined />
          </template>
        </a-step>
        <a-step :status="step6" title="데스크톱 사용자 할당">
          <template #icon>
            <LinkOutlined />
          </template>
        </a-step>
        <a-step :status="step7" title="완료">
          <template #icon>
            <SmileOutlined />
          </template>
        </a-step>
      </a-steps>
      <a-divider></a-divider>
    </div>

    <div v-else><a-empty /></div>
  </a-drawer>

  <a-modal
    v-model:visible="addModalView"
    title="자원 생성 및 할당 신청"
    :confirm-loading="confirmLoading"
    :ok-text="$t('label.ok')"
    :cancel-text="$t('label.cancel')"
    @cancel="handleCancel"
    @ok="createWorkspace"
  >
    <a-form :ref="formRef" :model="form" :rules="rules" layout="vertical" autocomplete="off" @finish="onFinish" @finishFailed="onFinishFailed">
      <a-form-item has-feedback name="name" label="회사 명">
        <a-input v-model:value="form.name" placeholder="회사명을 입력해주세요.(영문포함 7자 이하)" />
      </a-form-item>

      <a-form-item name="workspaceType" :label="$t('label.type')">
        <a-radio-group v-model:value="form.workspaceType" button-style="solid" @change="changeWorkspaceType">
          <a-radio-button value="C">클라이언트</a-radio-button>
          <a-radio-button value="S">서버</a-radio-button>
          <a-radio-button value="A">어플리케이션</a-radio-button>
        </a-radio-group>
      </a-form-item>
      <a-form-item name="desktopCount" label="데스크톱 수">
        <a-input-number v-model:value="form.desktopCount" title="데스크톱 수" min="1" max="10" style="width: 100%" />
      </a-form-item>
    </a-form>
  </a-modal>
</template>
<script>
import { defineComponent, reactive, ref } from "vue";
export default defineComponent({
  name: "Dashboard",
  components: {},
  props: {},
  setup() {
    const onClose = () => {
      visible.value = false;
    };
    const state = reactive({});
    return {
      state,
      onClose,
    };
  },
  data() {
    return {
      wsCount: ref(false),
      wsName: ref(""),
      confirmLoading: ref(false),
      wsDataList: ref([]),
      addModalView: ref(false),
      drawerView: ref(false),
      timer1: ref(null),
      timer2: ref(null),
      spinning: ref(false),
      workspaceCount: ref("0"),
      desktopVmCount: ref("0"),
      appVmCount: ref("0"),
      accountCount: ref("0"),
      desktopConCount: ref("0"),
      appConCount: ref("0"),
      stepStatus: ref("error"),
      dashboardStep: ref(0),
      step1: ref("wait"),
      step2: ref("wait"),
      step3: ref("wait"),
      step4: ref("wait"),
      step5: ref("wait"),
      step6: ref("wait"),
      step7: ref("wait"),
    };
  },
  created() {
    this.initForm();
  },
  unmounted() {
    clearInterval(this.timer1);
  },
  methods: {
    viewDrawer(val) {
      this.wsStatus();
      this.drawerView = val;
    },
    initForm() {
      let params = new URLSearchParams();
      params.append("id", "administrator");
      params.append("password", "Ablecloud1!");
      this.$worksApi.post("/api/login", params).then((response) => {
        //console.log(response);
        const data = response.data.result;
        if (response.status === 200 && data.login === true) {
          this.$store.dispatch("loginCommit", response.data);
          sessionStorage.setItem("token", data.token);
        }
      });

      this.formRef = ref();
      this.form = reactive({
        workspaceType: "C",
        desktopCount: 1,
      });
      this.rules = reactive({
        name: { required: true, max: 7, validator: this.validateName, trigger: "change", message: "회사명을 입력해주세요.(영문포함 7자 이하)" },
        workspaceType: { required: true },
        desktopCount: { required: true },
      });
    },
    async validateName(rule, value) {
      let lengthCheck = value.length > rule.max ? true : false; //길이체크
      //let containsEng = /[a-zA-Z]/.test(value); // 대소문자
      //let containsEngUpper = /[A-Z]/.test(value); 대문자
      //let containsNumber = /[0-9]/.test(value);
      let containsSpecial = /[~!@#$%^&*()\-_+|<>?:{}]/.test(value);
      let containsHangle = /[ㄱ-ㅎ|ㅏ-ㅣ|가-힣]/.test(value);
      let firstEng = /^[a-zA-Z]/.test(value);
      if (value === "" || containsHangle || containsSpecial || lengthCheck || !firstEng) {
        return Promise.reject();
      } else {
        return Promise.resolve();
      }
    },
    async wsStatus() {
      await this.$worksApi
        .get("/api/v1/workspace")
        .then(async (response) => {
          if (response.status == 200) {
            this.ws = response.data.result.list.filter((it) => it.name === this.wsName)[0];
            console.log("this.status :>> ", this.ws.template_ok_check);
            if (this.ws.template_ok_check === "Ready") {
              this.step4 = "finish";
              this.step5 = "process";

              clearInterval(this.timer1);
              await this.createDesktop();
              this.timer2 = setInterval(() => {
                this.vmStatus();
              }, 5000);
            }
          }
        })
        .catch((error) => {
          console.log(error);
        })
        .finally(() => {});
    },
    async vmStatus() {
      await this.$worksApi
        .get("/api/v1/instance/all")
        .then(async (response) => {
          if (response.status == 200) {
            if (response.data.result.instanceInfo !== null) {
              const vmDataList = response.data.result.instanceInfo.filter((it) => it.workspace_name === this.wsName);
              const vmDataReadyList = vmDataList.filter((it) => it.handshake_status == "Ready");

              console.log("vmDataList.length :>> ", vmDataList.length);
              console.log("vmDataReadyList.length :>> ", vmDataReadyList.length);
              if (vmDataList.length == this.form.desktopCount && vmDataReadyList == this.form.desktopCount) {
                this.step5 = "finish";
                this.step6 = "process";

                vmDataList.forEach(async (value, index, array) => {
                  await this.allocateVmUser(index + 1, array[index].uuid);
                });

                this.step6 = "finish";
                this.step7 = "finish";
                clearInterval(this.timer2);
              } else {
                return false;
              }
            }
          }
        })
        .catch((error) => {
          console.log(error);
        })
        .finally(() => {});
    },
    async allocateVmUser(count, vmUuid) {
      console.log("count :>> ", count);
      console.log("vmUuid :>> ", vmUuid);
      const account = this.form.name + "_U" + count;
      await this.$worksApi
        .put("/api/v1/connection/" + vmUuid + "/" + account)
        .then(async (response) => {})
        .catch((error) => {
          console.log(error);
        })
        .finally(() => {});
    },
    async createDesktop() {
      let params = new URLSearchParams();
      params.append("uuid", this.ws.uuid);
      params.append("quantity", this.form.desktopCount);
      await this.$worksApi
        .put("/api/v1/instance", params)
        .then(async (response) => {})
        .catch((error) => {
          console.log(error);
        })
        .finally(() => {});
    },
    createWorkspace() {
      let workspaceType = "";
      let masterTemplateName = "";
      let masterTemplateId = "";
      let description = "";
      if (this.form.workspaceType == "C") {
        masterTemplateName = "세무사랑 클라이언트 (3.0.0)";
        masterTemplateId = "03d75b88-37d4-4083-997e-59b583d8f86b";
        workspaceType = "desktop";
        description = "세무사랑 클라이언트가 설치된 가상머신을 사용하는 워크스페이스 입니다.";
      } else if (this.form.workspaceType == "S") {
        masterTemplateName = "세무사랑 클라이언트+서버 (3.0.1)";
        masterTemplateId = "da7df05d-96e5-4562-bd06-2cfecc1966d7";
        workspaceType = "desktop";
        description = "세무사랑 서버와 클라이언트가 설치된 가상머신을 사용하는 워크스페이스 입니다.";
      } else {
        masterTemplateName = "세무사랑 클라이언트 (3.0.0)";
        masterTemplateId = "03d75b88-37d4-4083-997e-59b583d8f86b";
        workspaceType = "application";
        description = "세무사랑 클라이언트가 설치된 가상머신을 사용하는 워크스페이스 입니다.(세무사랑 프로그램만 사용)";
      }

      this.wsName = this.form.name;

      let params = new URLSearchParams();
      params.append("name", this.form.name);
      params.append("description", "");
      params.append("type", workspaceType);
      params.append("shared", false);
      params.append("masterTemplateName", masterTemplateName);
      params.append("templateUuid", masterTemplateId);
      params.append("computeOfferingUuid", "fd82fdc9-bcfa-431a-a087-20e05a09905a");
      //console.log(params);
      this.formRef.value
        .validate()
        .then(() => {
          this.confirmLoading = true;
          this.$worksApi
            .get("/api/v1/group/" + this.form.name) //이름 중복 확인
            .then((response) => {
              if (response.status === 200) {
                //이름 중복일때 메시지 확인
                this.$message.error(this.$t("message.name.dupl"));
                this.confirmLoading = false;
              }
            })
            .catch((error) => {
              //이름 중복이 아닐때(status code = 401)

              this.$worksApi
                .post("/api/v1/workspace", params)
                .then(async (response) => {
                  this.$message.destroy();
                  if (response.status === 200) {
                    this.wsCount = true;
                    this.$message.success("자원 생성 요청이 완료되었습니다. 정상적인 접속을 하기 위해 테스트 및 준비 작업에 약 약 30분정도 소요됩니다.");
                    this.handleCancel();
                    this.timer1 = setInterval(() => {
                      this.wsStatus();
                    }, 5000);
                    for (var i = 1; i < this.form.desktopCount + 1; i++) {
                      await this.createAccount(this.form.name, i);
                    }

                    this.step1 = "finish";
                    this.step2 = "finish";
                    this.step3 = "finish";
                    this.step4 = "process";
                  }
                })
                .catch((error) => {
                  console.log(error);
                })
                .finally(() => {});
            });
        })
        .catch((error) => {
          console.log("error", error);
        });
    },
    handleCancel() {
      this.addModalView = false;
      this.confirmLoading = false;
    },
    createAccount(name, count) {
      var params = new URLSearchParams();
      var account = name + "_U" + count;

      // console.log("name :>> ", account);
      // console.log("num :>> ", count);
      params.append("username", account);
      params.append("firstName", account);
      params.append("lastName", account);
      params.append("password", "newzen12#");
      params.append("email", "");
      params.append("phone", "");
      params.append("title", "");
      params.append("department", "");

      this.confirmLoading = true;
      this.$worksApi //이름 중복 확인 체크
        .get("/api/v1/user/" + account)
        .then((response) => {
          if (response.status === 200) {
            //중복일 때
            this.$message.error(this.$t("message.name.dupl"));
            this.confirmLoading = false;
          }
        })
        .catch(async (error) => {
          //중복 이름 없을 때

          await this.$worksApi
            .put("/api/v1/user", params)
            .then((response) => {})
            .catch((error) => {})
            .finally(() => {});

          await this.$worksApi
            .put("/api/v1/group/" + name + "/" + account)
            .then((response) => {})
            .catch((error) => {})
            .finally(() => {});
        });
    },
  },
});
</script>

<style scoped>
.dashboard-a-card-cl {
  width: 100%;
  height: 100%;
  text-align: center;
}
.dashboard-a-col-one {
  text-align: right;
  padding: 10px;
  padding-bottom: 0px;
}

.dashboard-a-col {
  padding-bottom: 10px;
}
#content-action {
  text-align: right;
}
</style>
