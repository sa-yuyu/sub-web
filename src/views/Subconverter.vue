<template>
  <div class="root">
    <el-row style="margin-top: 10px">
      <el-col>
        <el-card>
          <div slot="header">订阅转换, 仅支持clash订阅</div>
          <el-container>
            <el-form
              :model="form"
              label-width="80px"
              label-position="left"
              style="width: 100%"
            >
              <el-form-item label="订阅链接:">
                <el-input
                  v-model="form.sourceSubUrl"
                  clearable
                  placeholder="仅支持订阅clash链接"
                  @blur="saveSubUrl"
                />
              </el-form-item>

              <div style="margin-top: 50px"></div>

              <el-divider content-position="center">
                <i class="el-icon-magic-stick"></i>
              </el-divider>

              <el-form-item label="定制订阅:">
                <el-input class="copy-content" disabled v-model="customSubUrl">
                  <el-button
                    slot="append"
                    v-clipboard:copy="customSubUrl"
                    v-clipboard:success="onCopy"
                    ref="copy-btn"
                    icon="el-icon-document-copy"
                    >复制</el-button
                  >
                </el-input>
              </el-form-item>
              <el-form-item label="订阅短链:">
                <el-input
                  class="copy-content"
                  disabled
                  v-model="curtomShortSubUrl"
                >
                  <el-button
                    slot="append"
                    v-clipboard:copy="curtomShortSubUrl"
                    v-clipboard:success="onCopy"
                    ref="copy-btn"
                    icon="el-icon-document-copy"
                    >复制</el-button
                  >
                </el-input>
              </el-form-item>

              <el-form-item
                label-width="0px"
                style="margin-top: 40px; text-align: center"
              >
                <el-button
                  style="width: 120px"
                  type="danger"
                  @click="makeUrl"
                  :disabled="form.sourceSubUrl.length === 0"
                  >生成订阅链接</el-button
                >
                <el-button
                  style="width: 120px"
                  type="danger"
                  @click="makeShortUrl"
                  :loading="loading"
                  :disabled="customSubUrl.length === 0"
                  >生成短链接</el-button
                >
                <!-- <el-button style="width: 120px" type="primary" @click="surgeInstall" icon="el-icon-connection">一键导入Surge</el-button> -->
              </el-form-item>
              <el-form-item label-width="0px" style="text-align: center">
                <el-button
                  style="width: 250px"
                  type="primary"
                  @click="clashInstall"
                  icon="el-icon-connection"
                  :disabled="customSubUrl.length === 0"
                  >一键导入Clash</el-button
                >
              </el-form-item>
            </el-form>
          </el-container>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
const defaultBackend =
  process.env.VUE_APP_SUBCONVERTER_DEFAULT_BACKEND + "/sub?";
const shortUrlBackend = process.env.VUE_APP_MYURLS_DEFAULT_BACKEND + "/short";


export default {
  data() {
    return {
      backendVersion: "",
      advanced: "1",

      // 是否为 PC 端
      isPC: true,

      options: {
        backendOptions: [{ value: "http://127.0.0.1:25500/sub?" }]
      },
      form: {
        sourceSubUrl: "",
        clientType: "",
        customBackend: "",

      },

      loading: false,
      customSubUrl: "",
      curtomShortSubUrl: "",
    };
  },
  created() {
    document.title = "Subscription Converter";
    this.isPC = this.$getOS().isPc;

    // 获取 url cache
    if (process.env.VUE_APP_USE_STORAGE === "true") {
      this.form.sourceSubUrl = this.getLocalStorageItem("sourceSubUrl");
    }
  },

  methods: {
    onCopy() {
      this.$message.success("Copied!");
    },

    clashInstall() {
      if (this.customSubUrl === "") {
        this.$message.error("请先填写必填项，生成订阅链接");
        return false;
      }

      const url = "clash://install-config?url=";
      window.open(
        url +
          encodeURIComponent(
            this.curtomShortSubUrl !== ""
              ? this.curtomShortSubUrl
              : this.customSubUrl
          )
      );
    },
    surgeInstall() {
      if (this.customSubUrl === "") {
        this.$message.error("请先填写必填项，生成订阅链接");
        return false;
      }

      const url = "surge://install-config?url=";
      window.open(url + this.customSubUrl);
    },
    makeUrl() {
      if (this.form.sourceSubUrl === "") {
        this.$message.error("请填写订阅链接");
        return false;
      }

      let backend =
        this.form.customBackend === ""
          ? defaultBackend
          : this.form.customBackend;

      let sourceSub = this.form.sourceSubUrl;
      sourceSub = sourceSub.replace(/(\n|\r|\n\r)/g, "|");

      this.customSubUrl =
        backend +
        "&url=" +
        encodeURIComponent(sourceSub);


      this.$copyText(this.customSubUrl);
      this.$message.success("定制订阅已复制到剪贴板");
    },
    makeShortUrl() {
      if (this.customSubUrl === "") {
        this.$message.warning("请先生成订阅链接，再获取对应短链接");
        return false;
      }

      this.loading = true;

      let data = new FormData();
      data.append("longUrl", btoa(this.customSubUrl));

      this.$axios
        .post(shortUrlBackend, data, {
          header: {
            "Content-Type": "application/form-data; charset=utf-8",
          },
        })
        .then((res) => {
          if (res.data.Code === 1 && res.data.ShortUrl !== "") {
            this.curtomShortSubUrl = res.data.ShortUrl;
            this.$copyText(res.data.ShortUrl);
            this.$message.success("短链接已复制到剪贴板");
          } else {
            this.$message.error("短链接获取失败：" + res.data.Message);
          }
        })
        .catch(() => {
          this.$message.error("短链接获取失败");
        })
        .finally(() => {
          this.loading = false;
        });
    },
    backendSearch(queryString, cb) {
      let backends = this.options.backendOptions;

      let results = queryString
        ? backends.filter(this.createFilter(queryString))
        : backends;

      // 调用 callback 返回建议列表的数据
      cb(results);
    },
    createFilter(queryString) {
      return (candidate) => {
        return (
          candidate.value.toLowerCase().indexOf(queryString.toLowerCase()) === 0
        );
      };
    },

    saveSubUrl() {
      if (this.form.sourceSubUrl !== "") {
        this.setLocalStorageItem("sourceSubUrl", this.form.sourceSubUrl);
      }
    },
    getLocalStorageItem(itemKey) {
      const now = +new Date();
      let ls = localStorage.getItem(itemKey);

      let itemValue = "";
      if (ls !== null) {
        let data = JSON.parse(ls);
        if (data.expire > now) {
          itemValue = data.value;
        } else {
          localStorage.removeItem(itemKey);
        }
      }

      return itemValue;
    },
    setLocalStorageItem(itemKey, itemValue) {
      const ttl = process.env.VUE_APP_CACHE_TTL;
      const now = +new Date();

      let data = {
        setTime: now,
        ttl: parseInt(ttl),
        expire: now + ttl * 1000,
        value: itemValue,
      };
      localStorage.setItem(itemKey, JSON.stringify(data));
    },
  },
};
</script>

<style scoped>
.root {
  max-width: 500px;
  margin: 0 auto;
}
</style>
