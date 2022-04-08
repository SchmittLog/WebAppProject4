<template>
  <div class="home">
    <world-time />
    <button @click="outtahere">Logout</button>
    <button @click="deleteuser">Delete Account</button>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import WorldTime from "../components/WorldTime.vue";
import {
  getAuth,
  onAuthStateChanged,
  User,
  Auth,
  signOut,
  deleteUser,
} from "firebase/auth";

@Component({
  components: {
    WorldTime,
  }
})

export default class HomeView extends Vue {
  userPhotoURL = "";
  auth: Auth | null = null;
  userInfo = "";
  
  mounted(): void {
    this.auth = getAuth();
    onAuthStateChanged(this.auth, (user: User | null) => {
      console.log("Auth changed", user);
      if (user) {
        this.userPhotoURL = user.photoURL ?? "";
        this.userInfo = `${user.displayName}`;
      }
    });
  }

  outtahere(): void {
    if (this.auth) signOut(this.auth);

    // Back to the previous page
    this.$router.back();
  }

deleteuser(): void{
  const delAuth = getAuth();
  const user = delAuth.currentUser;
  
  deleteUser(user!).then(() => {
    // User deleted.
    this.$router.back();
  }).catch((error) => {
    // An error ocurred
    // ...
  });
}
}
</script>
