<template>
  <div class="md-layout-item md-size-50 md-small-size-100">
    <md-card>
      <md-progress-bar md-mode="indeterminate" v-if="showProgress" />
      <md-card-header>
        <div class="md-title">{{ $t("message.settings.title") }}</div>
      </md-card-header>
      <md-card-content>
        <md-card class="md-layout-item md-small-size-100">
          <md-card-header>
            <div class="md-subheading">{{ $t("message.settings.general.title") }}</div>
          </md-card-header>
          <md-card-content>
            <md-field>
              <label for="generalData.externalAccountsGroup">{{ $t("message.settings.general.externalAccountsGroupLabel") }}</label>
                <md-select v-model="generalData.externalAccountsGroup" name="externalAccountsGroup" id="externalAccountsGroup" md-dense>
                  <md-option value="">None</md-option>
                  <md-option v-for="group in googleGroups" :key="group.email" :value="group.email">
                    {{ group.email.match(/^([^@]*)@/)[1] }} ({{ group.name }})
                  </md-option>
                </md-select>
            </md-field>
          </md-card-content>
          <md-card-actions>
            <md-button class="md-raised md-primary" @click.native="saveGeneralSettings">{{ $t("message.settings.general.saveButton") }}</md-button>
          </md-card-actions>
        </md-card>
        <md-card class="md-layout-item md-small-size-100">
          <md-card-header>
            <div class="md-subheading">{{ $t("message.settings.push.title") }}</div>
          </md-card-header>
          <md-card-content>
            <md-field v-if="!pushData.enabled">
              <label>{{ $t("message.settings.push.endpointHostnameEdit") }}</label>
              <md-input name="domain" id="domain" v-model="pushData.hostname" />
            </md-field>
            <md-content v-if="!pushData.enabled">{{ $t("message.settings.push.endpointHostnameLabel") + " https://" + pushData.hostname + "/cxf/push/notify" }}</md-content>
            <md-content v-if="pushData.enabled">{{ $t("message.settings.push.enabledLabel") }}</md-content>
          </md-card-content>
          <md-card-actions>
            <md-button class="md-raised md-primary" v-if="!pushData.enabled" @click.native="enablePushNotifications">{{ $t("message.settings.push.enableButton") }}</md-button>
            <md-button class="md-raised md-primary" v-if="pushData.enabled" @click.native="disablePushNotifications">{{ $t("message.settings.push.disableButton") }}</md-button>
          </md-card-actions>
        </md-card>
        <md-card class="md-layout-item md-small-size-100">
          <md-card-header>
            <div class="md-subheading">{{ $t("message.settings.sync.title") }}</div>
          </md-card-header>
          <md-card-content/>
          <md-card-actions>
            <md-button class="md-raised md-primary" @click.native="synchronizeGroups">{{ $t("message.settings.sync.groupsButton") }}</md-button>
            <md-button class="md-raised md-primary" @click.native="synchronizeUsers">{{ $t("message.settings.sync.usersButton") }}</md-button>
          </md-card-actions>
        </md-card>
      </md-card-content>
    </md-card>
  </div>
</template>

<script>
export default {
  name: 'Settings',
  data() {
    return {
      showProgress: false,
      generalData: {externalAccountsGroup: null},
      googleGroups:[],
      pushData: {hostname: window.location.hostname, enabled: false}
    };
  },
  created: function() {
    this.loadPushNotificationStatus()
    this.loadGoogleGroups()
    this.loadGeneralSettings()
    if (!this.$auth.userInfo.amAdmin) {
      this.$auth.logout()
    }
  },
  methods: {
    saveGeneralSettings() {
      var _this = this;
      this.$validator
        .validateAll()
        .then(res => {
          if (!res) {
            // Catch errors
            console.warn("Form Invalid: " + _this.errors);
            return;
          }
          _this.showProgress = true;
          console.info("Valid. Saving general settings");
          _this.$http
            .put(_this.$apiPrefix + "/identity/admin/general-settings", _this.generalData)
            .then(function(response) {
              console.info("General Settings saved!" + response.data)
              _this.showProgress = false
              _this.notifySuccess("message.settings.general.saved")
            })
            .catch(function(error) {
              console.warn("Error while saving general settings! " + error)
              _this.showProgress = false
              _this.notifyError(error.response)
            });
        })
        .catch(function(e) {});
    },
    enablePushNotifications() {
      var _this = this;
      this.$validator
        .validateAll()
        .then(res => {
          if (!res) {
            // Catch errors
            console.warn("Form Invalid: " + _this.errors);
            return;
          }
          _this.showProgress = true;
          console.info("Valid. Enabling push notifications");
          // Create acccount
          _this.$http
            .put(_this.$apiPrefix + "/identity/admin/push/enable", null, { params: { hostname: _this.pushData.hostname } })
            .then(function(response) {
              console.info("Push notifications enabled!" + response.data)
              _this.showProgress = false
              _this.loadPushNotificationStatus()
              _this.notifySuccess("message.settings.push.success")
            })
            .catch(function(error) {
              console.warn("Error while enabling push notifications! " + error)
              _this.showProgress = false
              _this.notifyError(error.response)
            });
        })
        .catch(function(e) {});
    },
    disablePushNotifications() {
      var _this = this;
      this.$validator
        .validateAll()
        .then(res => {
          if (!res) {
            // Catch errors
            console.warn("Form Invalid: " + _this.errors)
            return
          }
          _this.showProgress = true;
          console.info("Valid. Disabling push notifications")
          // Create acccount
          _this.$http
            .put(_this.$apiPrefix + "/identity/admin/push/disable", null)
            .then(function(response) {
              console.info("Push notifications disabled!" + response.data)
              _this.showProgress = false
              _this.loadPushNotificationStatus()
              _this.notifySuccess("message.settings.push.success")
            })
            .catch(function(error) {
              console.warn("Error while disabling push notifications! " + error)
              _this.showProgress = false
              _this.notifyError(error.response)
            });
        })
        .catch(function(e) {});
    },
    synchronizeGroups: function(event) {
      var _this = this;
      this.$validator
        .validateAll()
        .then(function(response) {
          _this.showProgress = true;
          console.info('Synchronizing all Gsuite groups to LDAP');
          _this.$http
            .put(_this.$apiPrefix + '/identity/admin/sync/groups')
            .then(function(response) {
              console.info('All GSuite groups synchronized');
              _this.notifySuccess("message.settings.sync.successGroups");
              _this.showProgress = false;
            })
            .catch(function(error) {
              console.warn('Error while synchronizing groups! ' + error);
              _this.showProgress = false;
              _this.notifyError(error.response);
            });
        })
        .catch(function(e) {
          // Catch errors
          console.warn('Form Invalid: ' + e);
        });
    },
    synchronizeUsers: function(event) {
      var _this = this;
      this.$validator
        .validateAll()
        .then(function(response) {
          _this.showProgress = true;
          console.info('Synchronizing Gsuite user attributes to LDAP');
          _this.$http
            .put(_this.$apiPrefix + '/identity/admin/sync/users')
            .then(function(response) {
              console.info('All GSuite user attributes synchronized');
              _this.notifySuccess("message.settings.sync.successUsers");
              _this.showProgress = false;
            })
            .catch(function(error) {
              console.warn(
                'Error while synchronizing user attributes! ' + error
              );
              _this.showProgress = false;
              _this.notifyError(error.response);
            });
        })
        .catch(function(e) {
          // Catch errors
          console.warn('Form Invalid: ' + e);
        });
    },
    loadPushNotificationStatus() {
      var _this = this;
      this.$http
            .get(this.$apiPrefix + "/identity/admin/push/status")
            .then(function(response) {
              console.info("Push notifications status!" + response.data);
              _this.pushData.enabled = response.data.enabled
            })
            .catch(function(error) {
              console.warn("Error while creating account! " + error);
              _this.showProgress = false;
              _this.notifyError(error.response);
            });
    },
    loadGoogleGroups() {
      var _this = this;
      this.$http
            .get(this.$apiPrefix + "/identity/admin/google/groups")
            .then(function(response) {
              _this.googleGroups = response.data
            })
            .catch(function(error) {
              console.warn("Error while getting external users groups! " + error);
              _this.showProgress = false;
              _this.notifyError(error.response);
            });
    },
    loadGeneralSettings() {
      var _this = this;
      this.$http
            .get(this.$apiPrefix + "/identity/admin/general-settings")
            .then(function(response) {
              _this.generalData = response.data
            })
            .catch(function(error) {
              console.warn("Error while getting general settings! " + error);
              _this.showProgress = false;
              _this.notifyError(error.response);
            });
    },
    notifySuccess(prefix) {
      this.$swal({
        toast: true,
        position: 'top',
        showConfirmButton: false,
        timer: 4000,
        type: 'success',
        title: this.$t(prefix + "Title"),
        text: this.$t(prefix + "Text")
      });
    },
    notifyError(response) {
      let data = response.data;
      let message = typeof data === "object" ? data.message : data;
      this.$swal({
        type: "error",
        title: "Error Occured",
        text: response.status === 404 ? "Resource not found!" : message
      });
    }
  }
};
</script>

<style>
  .md-card {
    padding: 8px;
    margin: 8px;
    vertical-align: top;
  }

.error-label {
  background-color: hotpink;
  color: black;
  padding: 5px;
}

.msg-label {
  background-color: darkseagreen;
  color: black;
  padding: 5px;
}
</style>
