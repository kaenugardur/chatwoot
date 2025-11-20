<script>
import LoadingState from 'dashboard/components/widgets/LoadingState.vue';
import CryptoJS from 'crypto-js';

export default {
  components: {
    LoadingState,
  },
  props: {
    config: {
      type: Array,
      default: () => [],
    },
    currentChat: {
      type: Object,
      default: () => ({}),
    },
    isVisible: {
      type: Boolean,
      default: false,
    },
    position: {
      type: Number,
      required: true,
    },
  },
  data() {
    return {
      hasOpenedAtleastOnce: false,
      iframeLoading: true,
      signedConfig: [],
    };
  },
  computed: {
    dashboardAppContext() {
      return {
        conversation: this.currentChat,
        contact: this.$store.getters['contacts/getContact'](this.contactId),
        currentAgent: this.currentAgent,
      };
    },
    contactId() {
      return this.currentChat?.meta?.sender?.id;
    },
    currentAgent() {
      const { id, name, email } = this.$store.getters.getCurrentUser;
      return { id, name, email };
    },
  },
  watch: {
    isVisible() {
      if (this.isVisible) {
        this.hasOpenedAtleastOnce = true;
      }
    },
    config: {
      immediate: true,
      deep: true,
      handler(newConfig) {
        this.signedConfig = newConfig.map(item => {
          if (!item.url) return item;
          const signed = this.generateSignedUrl(item.url);
          return {
            ...item,
            signed_url: signed,
          };
        });
      },
    },
  },
  mounted() {
    window.addEventListener('message', this.triggerEvent);
  },
  unmounted() {
    window.removeEventListener('message', this.triggerEvent);
  },
  methods: {
    generateSignedUrl(url) {
      const secret = import.meta.env.VITE_DASHBOARD_APP_SECRET || '';
      if (!secret) return url;
      const payload = {
        user_id: this.currentAgent.id,
        email: this.currentAgent.email,
        name: this.currentAgent.name,
        role: this.currentAgent.role,
        account_id: this.$store.getters.getCurrentAccountId,
        exp: Math.floor(Date.now() / 1000) + 1800, // +30 minutes lifetime
      };
      const payloadJson = JSON.stringify(payload);
      const payloadB64 = btoa(unescape(encodeURIComponent(payloadJson)))
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=+$/, '');
      const signature = CryptoJS.HmacSHA256(payloadB64, secret).toString(
        CryptoJS.enc.Hex
      );
      const signedurl = new URL(url);
      signedurl.searchParams.set('cw_payload', payloadB64);
      signedurl.searchParams.set('cw_sig', signature);
      return signedurl.toString();
    },
    triggerEvent(event) {
      if (!this.isVisible) return;
      if (event.data === 'chatwoot-dashboard-app:fetch-info') {
        this.onIframeLoad(0);
      }
    },
    getFrameId(index) {
      return `dashboard-app--frame-${this.position}-${index}`;
    },
    onIframeLoad(index) {
      // A possible alternative is to use ref instead of document.getElementById
      // However, when ref is used together with v-for, the ref you get will be
      // an array containing the child components mirroring the data source.
      const frameElement = document.getElementById(this.getFrameId(index));
      if (frameElement) {
        const eventData = {
          event: 'appContext',
          data: this.dashboardAppContext,
        };
        frameElement.contentWindow.postMessage(JSON.stringify(eventData), '*');
      }
      this.iframeLoading = false;
    },
  },
};
</script>

<!-- eslint-disable-next-line vue/no-root-v-if -->
<template>
  <div v-if="hasOpenedAtleastOnce" class="dashboard-app--container">
    <div
      v-for="(configItem, index) in signedConfig"
      :key="index"
      class="dashboard-app--list"
    >
      <LoadingState
        v-if="iframeLoading"
        :message="$t('DASHBOARD_APPS.LOADING_MESSAGE')"
        class="dashboard-app_loading-container"
      />
      <iframe
        v-if="
          configItem.type === 'frame' &&
          (configItem.signed_url || configItem.url)
        "
        :id="getFrameId(index)"
        :src="configItem.signed_url || configItem.url"
        @load="() => onIframeLoad(index)"
      />
    </div>
  </div>
</template>

<style scoped>
.dashboard-app--container,
.dashboard-app--list,
.dashboard-app--list iframe {
  height: 100%;
  width: 100%;
}
.dashboard-app--list iframe {
  border: 0;
}
.dashboard-app_loading-container {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  width: 100%;
}
</style>
