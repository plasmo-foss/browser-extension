<template>
  <div class="content">
    <!-- Login/register screen -->
    <div v-if="!isShowMfa" class="p-6 container" style="min-height: 350px;">
      <h1 class="h5 mb-3">
        Welcome to
        <a href="https://simplelogin.io" target="_blank"
          >SimpleLogin
          <font-awesome-icon
            icon="long-arrow-alt-up"
            :transform="{ rotate: 45 }" /></a
        >, the most powerful email alias solution!
      </h1>

      <form @submit.prevent="login">
        <div class="form-group">
          <label>Email</label>

          <input
            v-model="email"
            class="form-control"
            type="email"
            autofocus
            required
          />
        </div>

        <div class="form-group">
          <label>Password</label>
          <input v-model="password" type="password" class="form-control" />
        </div>

        <button class="btn btn-primary btn-block mt-2">Login</button>
      </form>

      <div class="text-center">
        <a
          :href="apiUrl + '/auth/register?next=%2Fdashboard%2Fsetup_done'"
          target="_blank"
          class="mt-2 btn btn-outline-success btn-block"
        >
          Sign Up <font-awesome-icon icon="external-link-alt" />
        </a>
      </div>

      <div class="text-center">
        <button @click="showApiKeySetup" class="mt-2 btn btn-link text-center">
          Sign in with API Key
        </button>
      </div>
    </div>
    <!-- END Login/register screen -->

    <!-- MFA screen -->
    <div v-else class="p-6 container" style="min-height: 350px;">
      <div class="p-3">
        <div class="mb-2">
          Your account is protected with Two Factor Authentication. <br />
        </div>

        <div>
          <b>Token</b>
          <p>Please enter the 2FA code from your 2FA authenticator</p>
        </div>

        <div style="margin: auto;">
          <input
            v-model="mfaCode"
            v-on:keyup.enter="submitMfaCode"
            placeholder="xxxxxx"
            autofocus
            class="form-control mt-3 w-100"
          />
          <button @click="submitMfaCode" class="btn btn-primary btn-block mt-2">
            Submit
          </button>
        </div>
      </div>
    </div>
    <!-- END MFA screen -->
  </div>
</template>

<script>
import Utils from "../Utils";
import SLStorage from "../SLStorage";
import EventManager from "../EventManager";
import Navigation from "../Navigation";
import { callAPI, API_ROUTE, API_ON_ERR } from "../APIService";

export default {
  data() {
    return {
      email: "",
      password: "",
      mfaKey: "",
      mfaCode: "",
      isShowMfa: false,
      apiUrl: "",
    };
  },
  async mounted() {
    this.apiUrl = await SLStorage.get(SLStorage.SETTINGS.API_URL);
  },
  methods: {
    async login() {
      try {
        const res = await callAPI(
          API_ROUTE.LOGIN,
          {},
          {
            email: this.email,
            password: this.password,
            device: Utils.getDeviceName(),
          }
        );

        if (res.data.api_key) {
          const userName = res.data.name || res.data.email;
          await SLStorage.set(SLStorage.SETTINGS.API_KEY, res.data.api_key);
          EventManager.broadcast(EventManager.EVENT.SETTINGS_CHANGED);

          Utils.showSuccess(`Hi ${userName}!`);

          Navigation.navigateTo(Navigation.PATH.MAIN);
        } else if (res.data.mfa_enabled) {
          this.mfaKey = res.data.mfa_key;
          this.isShowMfa = true;
        }
      } catch (err) {
        // FIDO
        if (err.response.status === 403) {
          Utils.showError(
            "WebAuthn/FIDO is not supported on browser extension yet, please use API Key to login"
          );
        } else {
          Utils.showError("Email or Password incorrect");
        }
      }
    },

    async submitMfaCode() {
      try {
        const res = await callAPI(
          API_ROUTE.MFA,
          {},
          {
            mfa_token: this.mfaCode,
            mfa_key: this.mfaKey,
            device: Utils.getDeviceName(),
          }
        );

        const userName = res.data.name || res.data.email;
        await SLStorage.set(SLStorage.SETTINGS.API_KEY, res.data.api_key);
        EventManager.broadcast(EventManager.EVENT.SETTINGS_CHANGED);

        Utils.showSuccess(`Hi ${userName}!`);

        Navigation.navigateTo(Navigation.PATH.MAIN);
      } catch (err) {
        Utils.showError("Incorrect MFA Code");
        this.mfaCode = "";
      }
    },

    showApiKeySetup: function () {
      Navigation.navigateTo(Navigation.PATH.API_KEY_SETTING, true);
    },
  },
  computed: {},
};
</script>
