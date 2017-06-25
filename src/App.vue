<template>
  <div class="container">
    <div class="row">
      <div class="col-xs-12">
        <h2>Authentication</h2>
        <button @click="handleSignInClick" >Sign In </button>
        <button @click="handleSignOutClick">Sign Out</button>
        <div v-show="loggedIn">Hello, {{ displayName }}</div>
        
        <hr>

        <h2>Appdata Files created by this Application</h2>
        <table class="table table-bordered table-hover">
          <thead>
            <tr>
              <th>#</th>
              <th>File Name</th>
              <th>File Id</th>
              <th>Operations</th>
            </tr>
          </thead>
          <tbody>
            <template v-for="(file, index) in driveFiles">
              <tr>
                <th scope="row">{{ index }}</th>
                <td>{{ file.name }}</td>
                <td>{{ file.id }}</td>
                <td>
                  <i class="glyphicon glyphicon-remove" v-on:click="deleteFileFromDrive(file.id)">Delete</i>
                  <i class="glyphicon glyphicon-pencil" v-on:click="setEditData(file.id, file.name)">Edit</i>
                </td>
              </tr>
            </template>
          </tbody>
        </table>

        <hr>
        
        <div class="form-group">
          <h2>Edit</h2>
          <div class="input-group">
            <span class="input-group-addon" id="basic-addon1">File Name</span>
            <input type="text" class="form-control" placeholder="Google Drive File Name" aria-describedby="basic-addon1" v-model="edit.name" readonly>
          </div>
          <div class="input-group">
            <span class="input-group-addon" id="basic-addon1">File Id</span>
            <input type="text" class="form-control" placeholder="Google Drive File Id" aria-describedby="basic-addon1" :value="edit.id" readonly>
          </div>
          <textarea class="form-control" rows="3" v-model="edit.content" placeholder="Content"></textarea>
          <button v-on:click="updateFile(edit)">Update File</button>
        </div>

        <hr>
        <div class="form-group">
          <h2>Create a New File</h2>
          <div class="input-group">
            <span class="input-group-addon" id="basic-addon1">File Name</span>
            <input type="text" class="form-control" placeholder="Google Drive File Name" aria-describedby="basic-addon1" v-model="newfile.name">
          </div>
          <br>
          <textarea class="form-control" rows="3" v-model="newfile.content" placeholder="Content"></textarea>
          <button v-on:click="updateFile(newfile)">Create File</button>
        </div>
      </div>
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
      driveFiles: [],
      edit: {
        id: "",
        name: "",
        content: ""
      },
      newfile: {
        name: "",
        content: ""
      }
    }
  },
  created() {
    console.log('App.vue created');
    console.log('App.vue created - Loading GAPI Client and Auth2');
    gapi.load('client:auth2', this.initializeGAPI);
  },
  methods: {
    initializeGAPI() {
      console.log('App.vue methods initializeGAPI');
        // Initialize the client with API key and People API, and initialize OAuth with an
        // OAuth 2.0 client ID and scopes (space delimited string) to request access.
        gapi.client.init({
            apiKey: 'AIzaSyCkkcuOuaHe5InK-kaC4WMeGCQxrzku9ZE',
            discoveryDocs: [
              "https://www.googleapis.com/discovery/v1/apis/people/v1/rest", // loads people api into gapi.client.people
              "https://www.googleapis.com/discovery/v1/apis/drive/v3/rest" // loads drive api into gapi.client.drive
            ],
            clientId: '776376867893-ef7btf6n7m5rh9dt5nbvmq0c7emouab0.apps.googleusercontent.com',
            scope: 'https://www.googleapis.com/auth/drive.appdata ' + // allows access to creating application specific files
                    'https://www.googleapis.com/auth/drive.file ' + // Modify files created with appdata scope
                    'https://www.googleapis.com/auth/userinfo.profile ',  // Name, Email, Photo
        }).then(this.respondToClientInit);

        console.log('App.vue methods initializeGAPI complete');
    },
    respondToClientInit() {
      // Listen for sign-in state changes.
      gapi.auth2.getAuthInstance().isSignedIn.listen(this.updateSigninStatus);

      // Handle the initial sign-in state.
      this.updateSigninStatus(gapi.auth2.getAuthInstance().isSignedIn.get());

      // Initial set up for files table
      this.listFilesFromDrive();
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
        'personFields': 'photos,metadata,names,emailAddresses',
      }).then(this.peopleGetResponse);
    },
    peopleGetResponse(response) {
      console.log('makeApiCall response result')
      console.log(response.result);
      this.displayName = response.result.names[0].displayName + " - " + response.result.emailAddresses[0].value;
    },

    // DRIVE FUNCTIONS
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
        // TODO: Handle Reading from different folders
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
      // TODO: Handle Reading from different folders
      console.log('App.vue listFilesFromDrive');
      var request = gapi.client.drive.files.list({
        spaces: 'appDataFolder',
        fields: 'nextPageToken, files(id, name)',
        pageSize: 100
      }).execute(this.listFilesFromDriveResponse);
    },
    listFilesFromDriveResponse(response) {
      console.log('App.vue listFilesFromDriveResponse');
      console.log(response);
      this.driveFiles = response.files;
    },
    getFileFromDrive(file_id, callback) {
      gapi.client.drive.files.get({
        fileId: file_id,
        alt: 'media'
      }).then(callback);
    },
    deleteFileFromDrive(file_id) {
      gapi.client.drive.files.delete({
        fileId: file_id,
      }).then(this.deleteFileFromDriveResponse);
    },
    deleteFileFromDriveResponse(response) {
      console.log('App.vue deleteFileFromDrive delete response');
      console.log(response);
      this.listFilesFromDrive();
    },
    setEditData(file_id, file_name) {
      this.edit.name = file_name;
      this.edit.id = file_id;
      this.edit.content = "Loading..."
      this.getFileFromDrive(file_id, this.setEditDataGetResponseHandler);
    },
    setEditDataGetResponseHandler(response) {
      console.log(response.body);
      this.edit.content = response.body;
    },
    updateFile(data) {
      this.saveFile(data, function(response) {
        console.log("Editted Data Updated!");
      });
      this.listFilesFromDrive();
    }
  }
}
</script>

<style>
</style>
