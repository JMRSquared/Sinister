<template>
  <Page>
    <GridLayout rows="auto,*" columns="*">
      <CardView elevation="0" row="0">
        <GridLayout class="top-nav" width="100%" columns="auto,*,auto">
          <Label
            :text="'mdi-menu' | fonticon"
            verticalAlignment="center"
            :fontSize="30"
            class="mdi"
            @tap="$refs.drawer.nativeView.toggleDrawerState()"
            col="0"
          />
          <Label
            class="title"
            verticalAlignment="center"
            :text="
              `Sinister${
                selectedVictim ? ` - ${selectedVictim.displayName}` : ''
              }!`
            "
            col="1"
          />
          <Label
            @tap="goHomeClear()"
            :text="'mdi-home-outline' | fonticon"
            col="2"
            verticalAlignment="center"
            :fontSize="30"
            class="mdi"
          />
        </GridLayout>
      </CardView>
      <RadSideDrawer row="1" ref="drawer">
        <StackLayout ~drawerContent backgroundColor="#ffffff">
          <GridLayout
            v-if="uniqueID"
            @tap="copyUniqueId()"
            class="drawer-header"
            rows="auto,auto"
            columns="auto,*"
          >
            <Image
              class="m-5"
              rowSpan="2"
              height="50"
              width="50"
              verticalAlignment="center"
              textAlignment="center"
              src="res://ic_icon"
            ></Image>
            <Label
              :text="
                uniqueID
                  ? uniqueID
                      .split('.')
                      .slice(0, 2)
                      .join(' ')
                  : 'Device'
              "
              class="t-16"
              col="1"
            />
            <Label
              text="Tap me!"
              class="font-weight-bold t-18"
              row="1"
              col="1"
            />
          </GridLayout>
          <Label
            v-if="!uniqueID"
            @tap="reloadUniqueID()"
            text="Something is not right here, do you have internet connection?"
            :textWrap="true"
            class="m-5"
            textAlignment="center"
          ></Label>
          <Ripple
            v-if="hasCriticalPermissions"
            @tap="requestCriticalPermissions()"
          >
            <GridLayout class="p-5" rows="auto,auto,auto" columns="auto,*,auto">
              <Label
                :text="'mdi-alert' | fonticon"
                rowSpan="2"
                verticalAlignment="center"
                :fontSize="25"
                class="mdi m-10 text-dark-orange"
              />
              <Label text="Tap me" class="t-14" col="1" />
              <Label
                text="Permission is required"
                class="font-weight-bold t-16"
                row="1"
                col="1"
              />
              <StackLayout
                class="bottom-line p-x-25 p-y-30"
                colSpan="3"
                row="2"
              ></StackLayout>
            </GridLayout>
          </Ripple>
          <GridLayout class="p-x-5 p-y-10" rows="auto" columns="*,auto">
            <Label
              class="drawer-item text-light-orange font-weight-bold"
              verticalAlignment="center"
              :fontSize="20"
              text="Users"
            />
            <Ripple
              @tap="loadVictims"
              col="1"
              rowSpan="2"
              verticalAlignment="center"
            >
              <Label
                :text="'mdi-refresh' | fonticon"
                verticalAlignment="center"
                :fontSize="30"
                class="mdi m-10"
              />
            </Ripple>
          </GridLayout>
          <GridLayout class="drawer-item" rows="*,auto">
            <ScrollView>
              <GridLayout
                v-for="(victim, i) in victims"
                :key="i"
                columns="*,auto"
              >
                <Ripple @tap="goToNotificationScreen(victim)">
                  <GridLayout
                    class="p-5"
                    rows="auto,auto,auto"
                    columns="auto,*,auto"
                  >
                    <Label
                      :text="'mdi-account-circle-outline' | fonticon"
                      rowSpan="2"
                      verticalAlignment="center"
                      :fontSize="25"
                      class="mdi m-10"
                    />
                    <Label :text="victim.displayName" class="t-14" col="1" />
                    <Label
                      :text="victim.userId"
                      class="font-weight-bold t-16"
                      row="1"
                      col="1"
                    />
                    <StackLayout
                      class="bottom-line p-x-25 p-y-30"
                      colSpan="3"
                      row="2"
                    ></StackLayout>
                  </GridLayout>
                </Ripple>
                <Ripple @tap="removeVictim(victim)" col="1">
                  <Label
                    :text="'mdi-close' | fonticon"
                    rowSpan="2"
                    verticalAlignment="center"
                    :fontSize="25"
                    class="mdi m-10"
                  />
                </Ripple>
              </GridLayout>
            </ScrollView>
            <Fab
              @tap="addNewVictim"
              icon="~/assets/images/ic_plus_white_36dp.png"
              class="fab-button text-white"
            >
            </Fab>
            <ad-banner row="1"></ad-banner>
          </GridLayout>
        </StackLayout>

        <GridLayout ~mainContent columns="*" rows="auto,*">
          <Navigator row="1" defaultRoute="/home" />
        </GridLayout>
      </RadSideDrawer>
    </GridLayout>
  </Page>
</template>

<script lang="ts">
import * as geolocation from "nativescript-geolocation";
import ScanBarCode from "./ScanBarCode.vue";
import ShowBarCode from "./ShowBarCode.vue";
import * as Permissions from "nativescript-permissions";
import { Vibrate } from "nativescript-vibrate";
const vibrate = new Vibrate();
import { prompt, confirm } from "tns-core-modules/ui/dialogs";
const appSettings = require("tns-core-modules/application-settings");
import AdBanner from "./AdBanner.vue";
import permissions from "nativescript-permissions";
const ad = require("tns-core-modules/utils/utils").ad;
declare var com: any;
const context = ad.getApplicationContext();

export default {
  data() {
    return {
      msg: "Hello World!",
      selectedVictim: null,
      hasCriticalPermissions: false
    };
  },
  components: {
    AdBanner
  },
  mounted() {
    console.log("Global vics", this.victims);
    this.hasCriticalPermissions =
      this.canReadNotifications() && this.hasLocationPermission();
    this.selectedVictim = null;
    setTimeout(() => {
      if (!this.uniqueID) {
        this.reloadUniqueID();
      }
      this.$forceUpdate();
    }, 2000);
  },
  methods: {
    requestPermissions() {
      permissions
        .requestPermissions(
          [
            "android.permission.ACCESS_FINE_LOCATION",
            "android.permission.ACCESS_BACKGROUND_LOCATION"
          ],
          "I need these permissions because I'm cool"
        )
        .then(() => {
          console.log("Woo Hoo, I have the power!");
          this.hasCriticalPermissions =
            this.canReadNotifications() && this.hasLocationPermission();
        })
        .catch(err => {
          console.log("Uh oh, no permissions - plan B time!", err);
        });
    },
    hasLocationPermission() {
      return (
        permissions.hasPermission("android.permission.ACCESS_FINE_LOCATION") &&
        permissions.hasPermission(
          "android.permission.ACCESS_BACKGROUND_LOCATION"
        )
      );
    },
    requestCriticalPermissions() {
      if (!this.hasLocationPermission()) {
        this.requestPermissions();
      }
      if (!this.canReadNotifications()) {
        this.requestToReadNotifications();
      }
    },
    requestToReadNotifications() {
      try {
        const intent = new android.content.Intent(
          android.provider.Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS
        );
        intent.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
        context.startActivity(intent);
        this.hasCriticalPermissions =
          this.canReadNotifications() && this.hasLocationPermission();
      } catch (err) {
        console.log("Can not open notification settings screen", err);
      }
    },
    canReadNotifications() {
      const enabledNotificationListeners = android.provider.Settings.Secure.getString(
        context.getContentResolver(),
        "enabled_notification_listeners"
      );
      return (
        enabledNotificationListeners != null &&
        enabledNotificationListeners.indexOf(context.getPackageName()) >= 0
      );
    },
    reloadUniqueID() {
      this.uniqueID = this.$firebase.generateUniqueID();
    },
    copyUniqueId() {
      this.$showModal(ShowBarCode);
      this.copyToClipboard(
        this.uniqueID,
        "Unique Id is copied to your clipboard"
      );
    },
    goHomeClear() {
      this.navigate("/home", null, {
        clearHistory: true
      });
    },
    goToNotificationScreen(victim) {
      this.selectedVictim = victim;
      this.$refs.drawer.nativeView.closeDrawer();
      this.navigate("/notification/screen", {
        victimID: victim.userId
      });
    },
    removeVictim(victim) {
      confirm({
        title: "Are you sure?",
        message: "The action can not be reverted",
        okButtonText: "Remove user",
        cancelButtonText: "cancel"
      }).then(result => {
        if (result) {
          this.removeVictimLocally(victim);
          this.$refs.drawer.nativeView.closeDrawer();
          this.navigate("/home", {}, { clearHistory: true });
        }
      });
    },
    async addNewVictim() {
      Permissions.requestPermission(
        android.Manifest.permission.CAMERA,
        "Needed for connectivity status"
      )
        .then(() => {
          console.log("Permission granted!");
          this.$showModal(ScanBarCode).then(userID => {
            if (userID == "Invalid") {
              this.showSnackBar("Only sinister QR-Codes are supported.");
              return;
            } else if (userID) {
              vibrate.vibrate(2000);
              prompt({
                title: "Provide a display name to use",
                okButtonText: "Save",
                cancelButtonText: "Cancel"
              }).then(async displayName => {
                if (displayName && displayName.result) {
                  if (!displayName.text || displayName.text.length < 3) {
                    return alert("Provide a valid Display name");
                  }
                  try {
                    await this.$firebase.admob.showInterstitial();
                  } catch (err) {
                    console.log("Error loading ad", err);
                  }
                  this.saveNewVictimLocally(userID, displayName.text);
                  this.loadVictims();
                  this.$refs.drawer.nativeView.closeDrawer();
                  this.navigate("/notification/screen", {
                    victimID: userID
                  });
                }
              });
            }
          });
        })
        .catch(() => {
          console.log("Permission is not granted (sadface)");
        });
    }
  }
};
</script>

<style scoped>
.bottom-line {
  height: 1;
  background-color: #ff6d00;
  opacity: 0.3;
}

.top-nav {
  padding: 10;
  background-color: #c43c00;
  color: #ffffff;
}

.title {
  text-align: left;
  padding-left: 16;
}

.message {
  vertical-align: center;
  text-align: center;
  font-size: 20;
  color: #c43c00;
}

.drawer-header {
  padding: 20 16 10 4;
  margin-bottom: 16;
  background-color: #c43c00;
  color: #ffffff;
  font-size: 24;
}

.drawer-item {
  padding: 8;
  font-size: 16;
}
</style>
