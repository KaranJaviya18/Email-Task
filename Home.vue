<template>
  <h2 class="welcome">welcome {{ name }}</h2>
  <nav><button type="submit" v-on:click="Logout()">Logout</button></nav>
  <button type="submit" v-on:click="displayform()">Compose Mail</button>
  <button type="submit" v-on:click="closeform()">Close Mail</button>

  <div class="container">
    <div id="toogle" style="display: none">
      <form @submit.prevent="onload">
        <label>from</label>
        <input type="email" v-model="from" name="email"  readonly/><br /><br />
        <label>To<span v-if="toValidation" style="color: red">{{ "*" }}</span></label>
        <input type="email" v-model="To" name="To" /><br /><br />
        <label>Subject</label>
        <input type="text" v-model="Subject" name="Subject" /><br /><br />
        <label>Attachment</label>
        <input type="file" @change="getfile" id="myInputFileID" /><br /><br />
        <label>Compose</label>
        <textarea name="Compose" v-model="Compose" cols="50" rows="15">
        </textarea
        ><br /><br />

        <button type="submit" v-on:click="send()">Send</button>
      </form>
    </div>
  </div>
  <table class="style18">
    SENT BOX
       <td><button v-on:click="delAll()">Delete All</button></td>
    <tr>
      <th>From</th>
      <th>To</th>
      <th>Subject</th>
      <th>Compose</th>
      <th>Attachment</th>
      <th>Delete</th>
    </tr>
    <tr v-for="mail in sentmail" :key="mail.id">
      <td>{{ mail.from }}</td>
      <td>{{ mail.To }}</td>
      <td>{{ mail.Subject }}</td>
      <td>{{ mail.Compose }}</td>
      <td v-if="mail.Attachment == undefined"></td>
      <td class="attach" v-for="attch in mail.Attachment" :key="attch">
        <a v-bind:href="attch" target="new">{{ attch }}</a>
      </td>

      <td><button v-on:click="del(mail.id)">Delete</button></td>
    </tr>
  </table>
  <br /><br /><br />
  <table class="style18">
    INBOX
    <tr>
      <th>From</th>
      <th>To</th>
      <th>Subject</th>
      <th>Compose</th>
      <th>Attachment</th>
      <th>Delete</th>
    </tr>

    <tr v-for="mail in inboxmail" :key="mail.id">
      <td>{{ mail.from }}</td>
      <td>{{ mail.To }}</td>
      <td>{{ mail.Subject }}</td>
      <td>{{ mail.Compose }}</td>
      <td v-if="mail.Attachment == undefined"></td>
      <td class="attach" v-for="attch in mail.Attachment" :key="attch">
        <a v-bind:href="attch" target="new">{{ attch }}</a>
      </td>
      <td><button v-on:click="del1(mail.id)">Delete</button></td>
    </tr>
  </table>
</template>

<script>
import { auth, db } from "../firebaseConfig";
import { signOut } from "firebase/auth";
import {
  collection,
  addDoc,
  updateDoc,
  query,
  where,
  orderBy,
  onSnapshot,
  doc,

} from "firebase/firestore";
import {
  getStorage,
  ref,
  uploadBytesResumable,
  getDownloadURL,
} from "firebase/storage";
  import { useToast } from "vue-toastification";

export default {
  name: "Home",
  data() {
    return {
      from: localStorage.getItem("userEmail"),
      To: "",
      Subject: "",
      Compose: "",
      sentmail: [],
      inboxmail: [],
      toValidation: false, 
      fileAttach: [],
      name: localStorage.getItem("username"),
      imgurl: [],
      abc: true,
    };
  },
  created() {
    this.sentbox(), this.inbox();
  },
  methods: {


       //get file for attachment
    getfile(e) {                             
      console.log(e.target.files[0]);
      var fileObj = e.target.files[0];
      this.fileAttach = fileObj;

      console.log(fileObj);
    },


        //for show compose mail form
    displayform() {                                                     
      document.getElementById("toogle").style.display = "block";
    },

    // for hide compose mail form
    closeform() {
      document.getElementById("toogle").style.display = "none";
    },

    //for logout 
    Logout() {
      signOut(auth, this.email, this.password).then((usercredential) => {
        this.$router.push({
          name: "Login",
        });
        localStorage.setItem("userAuth", false);
        localStorage.removeItem("userEmail", this.email);
        localStorage.removeItem("username");
        console.log(usercredential);
      });
    },

    //for send Mail
    async send() {
      if (this.To == "") {                       //for To validation
        this.toValidation = true;
           const toast = useToast();
             toast.error("Mail Not Sent ", {
        timeout: 7000
      });

      } else {
        const storage = getStorage();
        if (this.fileAttach.length != 0) {         //for checking if attachment or not
          const storageRef = ref(storage, "images/" + this.fileAttach.name);
          const uploadTask = uploadBytesResumable(storageRef, this.fileAttach);
          uploadTask.on(
            "state_changed",
            (snapshot) => {
              const progress =
                (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
              console.log("Upload is " + progress + "% done");
              switch (snapshot.state) {
                case "paused":
                  console.log("Upload is paused");
                  break;
                case "running":
                  console.log("Upload is running");
                  break;
              }
            },
            (error) => {
              switch (error.code) {
                case "storage/unauthorized":
                  break;
                case "storage/canceled":
                  break;

                case "storage/unknown":
                  break;
              }
            },
            () => {
              this.imgurl = [];                             // store attachment for set in firebase                              
              getDownloadURL(uploadTask.snapshot.ref).then(
                async (downloadURL) => {
                  console.log("File available at", downloadURL);
                  this.imgurl.push(downloadURL);

                  const docRef = await addDoc(collection(db, "Email"), {
                    from: this.from,
                    To: this.To,
                    Subject: this.Subject,
                    Compose: this.Compose,
                    createdAt: new Date(),
                    updatedAt: new Date(),
                    Attachment: this.imgurl,
                    deleteBySender: false,                    //for delete 
                    deleteByReceiver: false,       
                  });
                  console.log(docRef);

                  const updateDocref = updateDoc(docRef, {
                    id: docRef.id,
                  });
                  console.log(updateDocref);
                  console.log("Document written with ID: ", docRef.id);
                  document.getElementById("toogle").style.display = "none";
                  this.To = this.Subject = this.Compose = "";                       //for clear data when we send mail
                  this.fileAttach = [];
                  document.getElementById("myInputFileID").value = null;
                }
              );
            }
          );
        } else {
          const docRef = await addDoc(collection(db, "Email"), {
            from: this.from,
            To: this.To,
            Subject: this.Subject,
            Compose: this.Compose,
            createdAt: new Date(),
            updatedAt: new Date(),
            deleteBySender: false,
            deleteByReceiver: false,
          });

          const updateDocref = updateDoc(docRef, {
            id: docRef.id,
          });
          console.log(updateDocref);
          console.log("Document written with ID: ", docRef.id);
          document.getElementById("toogle").style.display = "none";
          this.To = this.Subject = this.Compose = "";          //for clear data when we send mail
          this.fileAttach = [];
          // alert("mail send succesfully")
            const toast = useToast();
             toast.success("Mail Sent  succesfully", {
        timeout: 7000
      });
          document.getElementById("myInputFileID").value = null; 
        }
      }
    },

    async inbox() {
      const q = query(
        collection(db, "Email"),
        where("To", "==", localStorage.getItem("userEmail")),
        where("deleteByReceiver", "==", false),     //for delete
        orderBy("createdAt")
      );
      const unsubscribe = onSnapshot(q, (querySnapshot) => {
        this.inboxmail = [];
        querySnapshot.forEach((data) => {
          console.log(data.id, " => ", data.data());
          this.inboxmail.push(data.data());
        });
      });
      console.log(unsubscribe);
    },

    async sentbox() {
      const q = query(
        collection(db, "Email"),
        where("from", "==", localStorage.getItem("userEmail")),
        where("deleteBySender", "==", false),
        orderBy("createdAt")
      );
      const unsubscribe = onSnapshot(q, (querySnapshot) => {
        this.sentmail = [];
        querySnapshot.forEach((data) => {
          console.log(data.id, " => ", data.data());
          this.sentmail.push(data.data());
        });
      });
      console.log(unsubscribe);
    },
    async del(id) {
      var result = confirm("Are you sure you want to delete mail");
      if (result) {
        console.log(id);

        console.log("deleted by sender", id);

        const deleteref = doc(db, "Email", id);
        await updateDoc(deleteref, {
          deleteBySender: true,
        }).catch((error) => console.log("Error", error));
      }
    },
    async del1(id) {
      var result = confirm("Are you sure you want to delete mail");
      if (result) {
        console.log(id);
        console.log("deleted by reciver");
        const deleteref = doc(db, "Email", id);
        await updateDoc(deleteref, {
          deleteByReceiver: true,
        });
      }
    },

      // updateDeletebyReciver(){
    //      const q1 = query(        //for new task of update recevier 
    //     collection(db, "Email"), 
    //   );
       
    //      onSnapshot(q1, (querySnapshot) => {
    //     querySnapshot.forEach((data) => {
    //       console.log(data.id, " => ", data.data());
    //       const updateDeleteby = doc(db, "Email" , data.id)   //add new deletereceiver
    //       updateDoc(updateDeleteby,{
    //         deletebyReceiver:false
    //       })

    //       const deleteDeleteby = doc(db, "Email",data.id)
    //       updateDoc(deleteDeleteby,{
    //         deleteByReciver: deleteField()
    //       })
    //     });
    //   });
    // }
  },
};
</script>

<style>
.style18 {
  width: 100%;
}

table {
  empty-cells: show;
}

th,
td {
  text-align: center;
  padding: 6px;
}

tr:nth-child(even) {
  background-color: #f2f2f2;
}

th {
  background-color: #485c55;
  color: white;
}

table,
th,
td {
  border: 1px solid black;
  border-collapse: collapse;
}

.attach td {
  empty-cells: show;
}
</style>
