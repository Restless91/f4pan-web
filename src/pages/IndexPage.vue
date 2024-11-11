<script setup lang="ts">
import Panel from 'primevue/panel';
import Menu from 'primevue/menu';
import Button from 'primevue/button';
import InputText from 'primevue/inputtext';
import { ref, watch, onMounted } from 'vue';
import { useCacheStore, useSystemConfigStore, useUserStore } from '@/store';
import { useMessage } from '@/hooks/useMessage';
import type { MenuItem } from 'primevue/menuitem';
import { parseBaiduShareUrl } from '@/utils/parse-baidu-share-url';
import { getRequestId } from '@/services/parse';
import { useRouter } from 'vue-router';
import { actionOne } from '@/utils/show-driver';
import Swal from 'sweetalert2';
import CryptoJS from 'crypto-js';

const router = useRouter();
const cacheStore = useCacheStore();
const pwd = ref('');
const url = ref('');
const surl = ref('');
const loading = ref(false);
const userStore = useUserStore();
const message = useMessage();
const systemConfigStore = useSystemConfigStore();
const menu = ref<Menu>();
const items = ref<MenuItem[]>([
  {
    label: '更多操作',
    items: [
      {
        label: '公告',
        icon: 'pi pi-bell',
        command: () => {
          cacheStore.noticeDialog = !cacheStore.noticeDialog;
        }
      },
      {
        label: '登录后台',
        icon: 'pi pi-cloud',
        command: () => {
          cacheStore.adminLogin = !cacheStore.adminLogin;
        }
      },
      {
        label: '新手教程',
        icon: 'pi pi-graduation-cap',
        command: () => {
          window.localStorage.removeItem('driver-step-done');
          actionOne();
        }
      }
    ]
  }
]);

const toggle = (event: Event) => {
  menu?.value?.toggle(event);
};

const adsWatched = ref(false);

const doParser = async () => {
  let password;
  if (systemConfigStore.requires_key !== 'none') {
    cacheStore.parseVerify = true;
    password = await cacheStore.awaitForParseVerify();
    if (password.length === 0) {
      return;
    }
  }
  loading.value = true;
  message.default('正在加载……');
  getRequestId(surl.value, pwd.value, password).then(({ data: res }) => {
    userStore.req_id = res.data;
    router.push({
      name: 'Parsed',
      params: {
        reqId: res.data,
        surl: surl.value,
        pwd: pwd.value
      }
    });
  });
};

const showAdPrompt = async () => {
  const uid = 1; // 示例UID，请替换为实际值
  const secret = "sdg45ffhxxfgsfgd"; // 用户签名密钥，请替换为实际值
  const adid = generateAdID();
  const key = generateKey(uid, secret, adid);

  // 创建广告请求
  try {
    const response = await fetch("https://ad.91ios.fun/api/create.php", {
      method: "POST",
      headers: {
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: new URLSearchParams({ uid: uid.toString(), adid: adid, key: key }),
    });

    const data = await response.json();
    if (data.code === 200) {
      const ticket = data.link;
      const query = encodeURIComponent(data.query);
      const wxLink = `${ticket}&query=${query}`;

      const userAgent = navigator.userAgent.toLowerCase();
      const isMobile = /iphone|ipod|android|blackberry|iemobile|opera mini/i.test(userAgent);
      let htmlContent;

      if (isMobile) {
        htmlContent = `<p>1.微信小程序看完广告后可进行对应操作。</p><p>2.每日看完5次广告后无限制下载。</p><p>3.资源整理不易，感谢支持。</p><a href="${wxLink}" target="_blank" style="color: blue;">点击观看广告解锁</a>`;
      } else {
        const redirectTo = encodeURIComponent(`https://ad.91ios.fun/api/h5.html?wxLink=${wxLink}`);
        const qrCodeURL = `https://api.qrserver.com/v1/create-qr-code/?data=${redirectTo}&size=150x150`;
        htmlContent = `<p>微信扫码观看广告后可进行对应操作</p><div style="display: flex; justify-content: center; align-items: center;"><img src="${qrCodeURL}" width="150" height="150"></div><p>1.激励视频广告需播放完毕才能得到激励。</p><p>2.每日看5次广告后无限制下载。</p><p>3.资源整理不易，感谢支持。</p>`;
      }

      Swal.fire({
        html: htmlContent,
        showConfirmButton: false,
        allowOutsideClick: false,
        allowEscapeKey: false,
      });

      const adCheckInterval = setInterval(async () => {
        try {
          const checkResponse = await fetch("https://ad.91ios.fun/api/check.php", {
            method: "POST",
            headers: { "Content-Type": "application/x-www-form-urlencoded" },
            body: new URLSearchParams({ uid: uid.toString(), adid: adid, key: key }),
          });

          const checkData = await checkResponse.json();
          if (checkData.code === 200) {
            clearInterval(adCheckInterval);
            Swal.fire({
              title: "广告观看完成，感谢您的支持",
              icon: "success",
              showConfirmButton: false,
              timer: 2000,
            }).then(() => {
              adsWatched.value = true;
              doParser();
            });
          } else if (checkData.code === 400) {
            console.log("广告未观看完毕");
          } else if (checkData.code === 500) {
            clearInterval(adCheckInterval);
            Swal.fire({
              title: "广告不存在或加载失败，请重试",
              icon: "error",
              showConfirmButton: true,
              timer: 2000,
            });
          }
        } catch (error) {
          console.error("Error checking ad status:", error);
        }
      }, 5000);
    } else {
      Swal.fire({
        icon: "error",
        title: "加载广告失败，请稍后再试",
        showConfirmButton: true,
        timer: 1500,
      });
    }
  } catch (error) {
    console.error("Failed to load ad:", error);
    Swal.fire({
      icon: "error",
      title: "加载广告失败，请稍后再试",
      showConfirmButton: true,
      timer: 1500,
    });
  }
};

const generateAdID = () => {
  const base = "custom";
  const timestamp = new Date().getTime();
  const randomString = Math.random().toString(36).substring(2, 15);
  return base + timestamp + randomString;
};

const generateKey = (uid: number, secret: string, token: string) => {
  return CryptoJS.MD5(uid + secret + token).toString();
};

watch(url, value => {
  const result = parseBaiduShareUrl(value);
  if (result.pwd) {
    pwd.value = result.pwd;
    message.success('识别到密码，已自动填充！');
  }
  if (result.surl) {
    surl.value = result.surl;
  }
});

</script>

<template>
  <div class="main-container">
    <div class="main">
      <Panel>
        <template #header>
          <div class="title">
            <span class="font-bold">度盘不限速加速解析</span>
          </div>
        </template>
        <template #icons>
          <Menu ref="menu" id="overlay_menu" :model="items" :popup="true"/>
          <button class="p-panel-header-icon p-link mr-2" @click="toggle" :disabled="loading"
                id="driver-step-menu">
            <span class="pi pi-cog"></span>
          </button>
        </template>
        <div class="p-fluid">
          <div class="field" id="driver-step-pan-url-input">
            <label for="disk-url">链接</label>
            <InputText type="text" v-model="url"/>
          </div>
          <div class="field" id="driver-step-pan-pwd-input">
            <label for="disk-password">密码</label>
            <InputText type="text" v-model="pwd"/>
          </div>
          <!-- 根据广告观看状态显示不同的按钮 -->
          <Button v-if="adsWatched" @click="doParser" label="提交" id="driver-step-action-parse"/>
          <Button v-else @click="showAdPrompt" label="看广告免费解析" id="driver-step-action-ad"/>
        </div>
      </Panel>
    </div>
  </div>
</template>

<style scoped>

</style>
