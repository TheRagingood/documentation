## Standard File — Client Development Guide

This guide walks through the essentials of building an application that consumes a Standard File API. This code is based on the Standard Notes client, and uses a non-compileable Javascript-ish pseudocode.

For the full specification, see the [Standard File guide](http://standardfile.org/).

### Authentication
To authenticate a user, we need to make a request to the server to retrieve the password derivation parameters used to generate the user's password. This information is public, and can be retrieved by a GET call to `/auth/params` with `email` as a paramter:

```
getAuthParamsForEmail(email, callback) {
	var request = Restangular.one("auth", "params");
	request.get({email: email}).then(function(response){
	  callback(response.plain());
	})
}
```

Auth params include:
- pw_salt (fed to pw_func)
- pw_cost (the number of iterations for the kdf)


Next, compute the user's password and encryption keys using the user's text field inputted password, the `pw_salt` and `pw_cost` fields as received earlier, and a `PBKDF2` output length of 768 bits:

```
generatePasswordAndKey(password, pw_salt, pw_cost) {		
	var output = PBKDF2(password, pw_salt, { keySize: 768, hasher: CryptoJS.algo.SHA512, iterations: pw_cost }).toString();

	var outputLength = output.length;
	var splitLength = outputLength/3;
	var pw = output.slice(0, splitLength);
	var mk = output.slice(splitLength, splitLength * 2);
	var ak = output.slice(splitLength * 2, splitLength * 3);
	return [pw, mk, ak]
}
```

Here the return values are `pw`, which is the password that will be sent to the server, and `mk`, which is the master encryption key, and 'ak' is the auth key, which are never sent to the server.

From here, it's basic authentication as usual:

```
var request = Restangular.one("auth/sign_in");
var params = {password: pw, email: email};
request.post().then(function(response){
  var jwt = response.token;
  localStorage.setItem("jwt", response.token);
  callback(response);
})
```

Here we capture `jwt`, which is a JSON Web Token, and save it locally. This token must be passed to any future API calls to authenticate the user.

You send this token via an HTTP header:
`Authorization: Bearer _jwt_value_`

### Registration

On registration, the client will choose a cost for PBKDF2. Recommended is 60,000 for native platforms and 3,000 for web platforms.

To register, the client must also generate a salt. For logging in, the salt is returned by the server, but for registering, it must be done locally.

**Generating a salt:**

1. Generate a random key `nonce`.
2. Compute `salt = SHA1(email + ":" + nonce)`

Then, call the same `generatePasswordAndKey` function you used for logging in. Then, when registering, make a POST request to `auth` with the auth params you used, including the salt **and** `pw_nonce`:

```
  var request = Restangular.one("auth");
  var params = {password: pw, email: email, pw_salt: pw_salt, pw_cost: pw_cost};
  request.post().then(function(response){
    localStorage.setItem("jwt", response.token);
    callback(response);
  })
```

Save the JWT.

## Architecture

What follows is a recommended set up for how models and controllers should be structured on the client. This can however be done in whatever way the developer prefers.

### Models

Create a class called `Item`. This model should have all metadata related fields of items returned from the server, including:

**Abstract `Item` class:**

- uuid
- content
- enc_item_key
- content_type
- created_at
- updated_at
- deleted

Then, create subclasses of `Item` for structures that your application supports. In the case of the Standard Notes app, that will be a `Note` class and a `Tag` class:

**Note extends Item:**

- title
- text

**Tag extends Item:**

- title

### Controllers

Three main controllers are relied on for a clean separation of concerns:

- **ItemManager**: responsible for mapping items returned from the server to local models.
- **ApiController**: responsible for communicating and retrieving changes from the server, as well as faciliating encryption/decryption with the help of CryptoHelper.
- **CryptoHelper**: decrypts/encrypts outgoing and incoming items from ApiController.

Note that encryption and decryption should be handled by the ApiController _before_ it passes it off to the ModelManager. It should be treated as an API level thing, and should not be intertwined with normal application logic.

First, let's handle fetching new items.

**ApiController**

```
refreshItems() {
	let request = Restangular.one("items/sync");
	if(self.syncToken) {
		request.syncToken = self.syncToken;
	}
	request.post().then(function(response){
		self.syncToken = response.sync_token;
		self.handleResponseItems(response.retrieved_items);
	})
}
```

```
handleResponseItems(responseItems) {
	CryptoHelper.decryptResponseItems(responseItems);
	ItemManager.sharedInstance.mapResponseItemsToLocalItems(responseItems);
}
```

**CryptoHelper**

```
// Modifies response items in place
decryptResponseItems(responseItems) {
	for item in responseItems {
		self.decryptItem(item);
	}
}
```
We'll go over the `decryptItem` function further below.

**ItemManager**

```
mapResponseItemsToLocalItems(responseItems) {
	var mappedItems = [];
	for responseItem in responseItems {
		if responseItem["deleted"] == true {
			let item = Database.find(uuid: responseItem["uuid"]);
			if(item) {
				Database.remove(item);
			}
			continue;
		}

		let item = Database.findOrCreate(uuid: responseItem["uuid"]);
		item.updateFromJSON(responseItem);
		self.resolveReferencesForItem(item);
	}
}
```

**Item**

```
updateFromJSON(jsonDic) {
    self.uuid = jsonDic["uuid"]
    self.contentType = jsonDic["content_type"]
    self.encItemKey = json["enc_item_key"]
    self.createdAt = dateFromString(jsonDic["created_at"])
    self.updatedAt = dateFromString(jsonDic["updated_at"])

    self.content = jsonDic["content"] // this is just a string

    let contentObject = JSON.parse(self.content)
    mapContentToLocalProperties(contentObject)
}
```

Then each subclass will override their own implementation of `mapContentToLocalProperties`:

**Note**

```
mapContentToLocalProperties(contentObject) {
	// super has no role here, but we'll call it in case we decide to make changes
	super.mapContentToLocalProperties(contentObject)
	self.title = contentObject["title"]
	self.text = contentObject["text"]
}
```

**Tag**

```
mapContentToLocalProperties(contentObject) {
	super.mapContentToLocalProperties(contentObject)
	self.title = contentObject["title"]
}
```

Each Item's content field can have an array of referenced items in its `references` field. This can look like this:

Item:

```
{
  "uuid": "ec549665-824d-4078-a65a-8e1e223e33bf",
  "content_type": "Note",
  "content": {
    "references": [
      {
        "uuid": "cb92d48d-536b-40f8-be79-95de7ec1dff5",
        "content_type": "Tag"
      }
    ],
    "title": "My Note",
    "text": "Hello world."
  },
}
```

`ItemManager.resolveReferencesForItem` simply sets up the local relationships for the models you just created or updated:

```
resolveReferencesForItem(item) {
	item.clearReferences()
	let references = item.contentObject["references"]
	for reference in references {
		let referencedItem = Database.find(reference["uuid"]
		if referencedItem {
			item.addItemAsRelationship(referencedItem)
		}
	}
}
```

And then in the individual model classes, you'd do something like this:

**Note**

```
addItemAsRelationship(item) {
	if item.contentType == "Tag" {
		self.addTag(item);
	}
	super.addItemAsRelationship(item);
}
```

Next, let's handle saving items to the server.

Add a field to your local models `dirty`. Anytime you make a change to an item, set `dirty` equal to `true`.

Then ask the ApiController to upload changes:

ApiController:

```
saveDirty() {
	let dirty = ItemManager.sharedInstance.fetchDirty()
	let itemParams = _map(dirty, function(item){
		return self.paramsForItem(item);
	})

	let request = Restangular.one("items/sync");
	request.items = itemParams;
	request.post().then(function(response(){
		let savedItems = response.saved_items;
		self.handleResponseItems(savedItems, metadataOnly: true)
	})
}
```

Upon completion, the server will return your saved items under the `saved_items` param. You don't want to merge the `content` from these items with your local models because your local models could have updated content. For example, imagine you're dealing with a Note.

The user types the text "Hello", the app sync those changes. By the time the request is complete, the user has typed "World". If you merge the saved_items with your local items, then it will delete "World". For this reason, you should only merge metadata from saved items. This means every field _except_ for `content`, `enc_item_key`, and `auth_hash`.

**ApiController**

```
paramsForItem(item) {
	var params = {}
	params["content_type"] = item.contentType
	params["uuid"] = item.uuid
	params["deleted"] = item.deleted

	let encryptedParams = Crypto.encryptionParamsForItem(item);
	params.merge(encryptedParams);
}
```

**Item**

```
createContentJSONStringFromProperties() {
	var params = self.structureParams()
	return JSON.stringify(params)
}
```

The base class Item will implement the `structureParams()` method, along with subclasses. The Item class will handle setting up references, while the individual subclasses will handle properties specific to that class.

**Item**

```
structureParams() {
	return ["references" : referencesParams()]
}

func referencesParams() {
    fatalError("This method must be overridden")
}
```

Then in individual subclasses, like **Note**:

```
structureParams() {
	let params = {
		"title" : self.title,
     		"text" : self.text
	}

	// must merge with super
	params.mergeWith(super.structureParams())

	return params
}

// Returns an array of dictionaries of related item metadata (uuid and content type only).
referencesParams() {
    var references = []
    for tag in self.tags {
        references.append(["uuid" : tag.uuid, "content_type" : tag.contentType])
    }
    return references
}
```

**CryptoHelper**

Please see the web implementation for encrypting and decrypting items, available [here](https://github.com/standardnotes/web/tree/master/app/assets/javascripts/app/services/encryption).

Next, let's merge `refreshItems()` and `saveDirty()` into one function called `sync()`:

**ApiController**

```
sync() {
	let request = Restangular.one("items/sync");

	let dirty = ItemManager.sharedInstance.fetchDirty()
	let itemParams = _map(dirty, function(item){
		return self.paramsForItem(item);
	})

	request.items = itemParams;

	if(self.syncToken) {
		request.syncToken = self.syncToken;
	}

	request.post().then(function(response){
		self.syncToken = response.sync_token;
		self.handleResponseItems(response.retrieved_items);
		self.handleResponseItems(response.saved_items, metadataOnly: true)
	})
}
```

Done.

### Deleting an item

**ItemManager**

```
setItemToBeDeleted(item) {
    item.deleted = true
    item.dirty = true
}
```

Then sync.

After successful sync, delete the item from your local database.

## Next Steps

Join the [Slack](https://standardnotes.slack.com) group for general development discussion. You can also email [dev@standardnotes.org](mailto:dev@standardnotes.org) for any questions.

Check out the source code for other completed clients!

* [iOS + Android](https://github.com/standardnotes/mobile)
* [Web](https://github.com/standardnotes/web)

For reference, see also [the official Ruby implementation](https://github.com/standardfile/ruby-server) for the Standard File server.

Follow [@standardnotes](https://twitter.com/standardnotes) for updates and announcements.
