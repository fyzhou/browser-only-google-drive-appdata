<template>
  <div>
    <button @click="handleSignInClick" >Sign In </button>
    <button @click="handleSignOutClick">Sign Out</button>
    <div v-show="loggedIn">{{ displayName }}</div>
    <hr>
    <button @click="listFilesFromDrive">List From Drive</button>
    <hr>
    <template v-for="file in driveFiles">
      <label>{{ file.id }}</label>
      <button>{{ file.name }}</button>
    </template>
    <hr>
    <button @click="insertFileIntoAppData">Add File To Drive</button>
    <hr>
    <div class="form-group">
      <label>File Id: </label>
      <input class="form-control" type="text" v-model="fileid">
      <button @click="getFileFromDrive">Get File Id From Drive</button>
      <button @click="deleteFileFromDrive">Delete File Id From Drive</button>
    </div>

  </div>
</template>

<script>
export default {
  name: 'app',
  data () {
    return {
      loggedIn: false,
      displayName : "",
      fileid : "",
      driveFiles: [],
    }
  },
  created() {
    console.log('App.vue created');
    console.log('App.vue created - Loading GAPI Client and Auth2');
    gapi.load('client:auth2', this.initializeGAPI);
    // console.log('App.vue created - Loading GAPI Drive');
    // gapi.load('drive', 'v2', this.initializeDrive);
  },
  methods: {
    initializeGAPI() {
      console.log('App.vue methods initializeGAPI');
        // Initialize the client with API key and People API, and initialize OAuth with an
        // OAuth 2.0 client ID and scopes (space delimited string) to request access.
        gapi.client.init({
            apiKey: 'AIzaSyCkkcuOuaHe5InK-kaC4WMeGCQxrzku9ZE',
            discoveryDocs: [
              "https://www.googleapis.com/discovery/v1/apis/people/v1/rest", 
              "https://www.googleapis.com/discovery/v1/apis/drive/v3/rest"
            ],
            clientId: '776376867893-ef7btf6n7m5rh9dt5nbvmq0c7emouab0.apps.googleusercontent.com',
            scope: 'https://www.googleapis.com/auth/drive.appdata ' +
                    'https://www.googleapis.com/auth/drive.file ' +
                    'https://www.googleapis.com/auth/userinfo.profile ',
        }).then(this.respondToClientInit);

        console.log('App.vue methods initializeGAPI complete');
    },
    respondToClientInit() {
      // Listen for sign-in state changes.
      gapi.auth2.getAuthInstance().isSignedIn.listen(this.updateSigninStatus);

      // Handle the initial sign-in state.
      this.updateSigninStatus(gapi.auth2.getAuthInstance().isSignedIn.get());

      console.log('App.vue methods initializeDrive');
      console.log(gapi.client.drive);
      console.log('App.vue methods initializeDrive drive initialized?');

      console.log('App.vue methods initializeDrive Post Initialized Playground Start');
      this.listFilesFromDrive();
      console.log('App.vue methods initializeDrive Post Initialized Playground End');
    },
    updateSigninStatus(isSignedIn) {
      console.log('App.vue updateSigninStatus isSignedIn: ' + isSignedIn);
      // When signin status changes, this function is called.
      // If the signin status is changed to signedIn, we make an API call.
      if (isSignedIn) {
        this.makeApiCall();
      }
      this.loggedIn = isSignedIn;
    },
    handleSignInClick(event) {
      console.log('App.vue handleSignInClick event: ' + JSON.stringify(event));
      // Ideally the button should only show up after gapi.client.init finishes, so that this
      // handler won't be called before OAuth is initialized.
      gapi.auth2.getAuthInstance().signIn();
    },
    handleSignOutClick(event) {
      console.log('App.vue handleSignOutClick event: ' + JSON.stringify(event));
      gapi.auth2.getAuthInstance().signOut();
    },
    makeApiCall() {
      console.log('App.vue makeApiCall');
      // Make an API call to the People API, and print the user's given name.
      gapi.client.people.people.get({
        'resourceName': 'people/me',
        'personFields': 'photos,metadata,names',
      }).then(this.peopleGetResponse);
    },
    peopleGetResponse(response) {
      console.log('makeApiCall response result')
      console.log(response.result);
      this.displayName = response.result.names[0].givenName;
      console.log('Hello, ' + response.result.names[0].givenName);
      console.log('Picture, ' + response.result.photos[0].url);
      console.log('ID, ' + response.result.metadata.sources[0].id);
    },


    // DRIVE FUNCTIONS
    insertFileIntoAppData() {
      var file = {
        // id: '13DROYKbVba3QjG4dwJ-6OqpHNKgg5GR6D8_P1R6Bn-hj',
        name: "simple-upload-test-1234.txt",
        content: "{'meow':'rawr'}",
      };
      this.saveFile(file, function(arg) {
        console.log(arg);
      });
    },
    saveFile(file, done) {
      function addContent(fileId) {
        return gapi.client.request({
            path: '/upload/drive/v3/files/' + fileId,
            method: 'PATCH',
            params: {
              uploadType: 'media'
            },
            name: file.name,
            body: file.content,
            fields: 'content'
          });
      }

      if (file.id) { //just update
        addContent(file.id).then(function(resp) {
          console.log('File just updated', resp.result);
          done(resp.result);
        })
      } else { //create and update
        gapi.client.drive.files.create({
          name: file.name,
          parents: [ 'appDataFolder'],
        }).then(function(resp) {
          addContent(resp.result.id).then(function(resp) {
            console.log('created and added content', resp.result);
            done(resp.result);
          })
        });
      }
    },
    listFilesFromDrive() {
      console.log('App.vue listFilesFromDrive');
      var request = gapi.client.drive.files.list({
        spaces: 'appDataFolder',
        fields: 'nextPageToken, files(id, name)',
        pageSize: 100
      }).execute(this.listFilesFromDriveResponse);
    },
    listFilesFromDriveResponse(response) {
      console.log('App.vue listFilesFromDrive list response');
      console.log('listing files from drive...');
      console.log(response);
      this.driveFiles = response.files;
      response.files.forEach(function(file) {
        console.log('Found file: ', file.name, file.id);
      });
    },
    getFileFromDrive() {
      gapi.client.drive.files.get({
        fileId: this.fileid,
        alt: 'media'
      }).then(function (response) {
        console.log('App.vue getFileFromDrive get response');
        console.log(response);
        return response;
      });
    },
    deleteFileFromDrive() {
      gapi.client.drive.files.delete({
        fileId: this.fileid,
      }).then(function (response) {
        console.log('App.vue deleteFileFromDrive delete response');
        console.log(response);
        return response;
      });
    }
  }
}
</script>

<style>
</style>
