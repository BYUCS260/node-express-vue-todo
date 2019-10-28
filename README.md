# Node, Express, and Vue Todo List

This is a simple todo list application built using [Node](https://nodejs.org/en/), [Express](https://expressjs.com/), and [Vue](https://vuejs.org/).

To learn how this is built, clone this repository, then write
code for the back end in `server.js` and for the front end in `index.html` and `script.js`.

## Front end

The `index.html` and `script.js` contain a working front end written with Vue. We will be writing a back end and then modifying this code to use the back end.

To see what we have so far, inside your cloned repository, start up a Python server and view the site in your browser:

```
python -m SimpleHTTPServer
```

You can navigate to `localhost:8000` to see the site.

## If you are using nvm

We previously introduced Node in the [Learning Node and Express](https://github.com/BYU-CS-260-Winter-2019/learning-node-express) activity. If you setup Node using nvm, then you need to first tell nvm which version of node to use:

```
nvm use stable
```

## Initialize a new node project

The first step with node is to initialize a new project:

```
npm init
```

This will ask you a number of questions. Here is how I answered them:

```
package name: (express-vue-todo-solution)
version: (1.0.0)
description: todo list
entry point: (script.js) server.js
test command:
git repository:
keywords:
author:
license: (ISC)
```
**Be sure to make the entry point server.js**

Now you need to install Express. This will automatically save express as a dependency in
`package.json`:

```
npm install express body-parser
```

**Tip**: when putting code into a repository that uses node.js, be sure to create a file called
  `.gitignore` in the top level of your repository that contains:

```
node_modules
```

## setup

We'll start with a basic setup for a back end server. In `server.js`, put the following:

```
const express = require('express');
const bodyParser = require('body-parser');

const app = express();

// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({
  extended: false
}));

// parse application/json
app.use(bodyParser.json());

app.use(express.static('public'));

app.listen(3000, () => console.log('Server listening on port 3000!'));
```

This is the boilerplate we'll use for a Node and Express back end. We first load the modules we need. Then we create an Express app and we configure the body parser library so that it will parse forms and JSON requests.

We setup Node so that it will serve files from the `public` folder like an ordinary web server.

We start the server on port 3000.

## Creating and Reading items

Remember that on the back end a REST API will support CRUD operations -- Create, Read, Update, and Delete. We will start by doing the create and read operations.

Add the following before the `app.listen()` line in `server.js`:

```
let items = [];
let id = 0;
```

Since we don't have a database yet, we will store the items in an array. This means we can store state as long as the back end is running, but once it is killed and restarted, we'll lose everything. This is better than what we have been doing so far (where reloading the web page lost all the data), but we'll want to add a database in subsequent exercises so we can store our data permanently.

Now let's do the create operation:

```
app.post('/api/items', (req, res) => {
  id = id + 1;
  let item = {
    id: id,
    text: req.body.text,
    completed: req.body.completed
  };
  items.push(item);
  res.send(item);
});
```

We support requests to create a todo list item by accepting a POST request on `/api/items`.This request should send two fields: `text` and `completed`. We'll also add an `id` field, which we will increment each time a new item is added. This will let us edit items by specifying the item `id`.

Creating a new item in the todo list is as simple as creating a new object and then pushing it onto the array. We send back a 200 OK  with the created item in the body of the response. The item will get converted to JSON.

Note, for now, our API has one shared todo list for everyone, and *anyone* can create new items. Once we show you how to support authentication, we will have one list per user and users will only be able to access their own list.

Next, we'll support the read operation:

```
app.get('/api/items', (req, res) => {
  res.send(items);
});
```

This is pretty simple. We just send the entire array in a response. It will get converted into JSON, with a 200 OK response code.

Let's test it! In one terminal:

```
node server.js
```

In another terminal:

```
curl -X POST -d '{"text":"get an A on the exam", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X POST -d '{"text":"party all night", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X GET localhost:3000/api/items
```

## Connecting the front end

Currently, the front end in `script.js` initializes items in a local array called `items`. We need to modify this so that it instead gets the list of items from the back end. We also need to use the back end to create items.

Let's start by initializing the list to an empty array instead of having hard-coded data there:

```
  data: {
    items: [],
    text: '',
    show: 'all',
  },
  ```

Now add a `created` section to the Vue app:

```
created: function() {
  this.getItems();
},
```

This will call the `getItems` function when Vue is created. Let's add that function:

```
    async getItems() {
      try {
        const response = await axios.get("/api/items");
        this.items = response.data;
      } catch (error) {
        console.log(error);
      }
    },
```

This uses the axios library to get the items, then store them in the `items` property.

To add new items, let's modify the `addItems` function:

```
    async addItem() {
      try {
        const response = await axios.post("/api/items", {
          text: this.text,
          completed: false
        });
        this.text = "";
        this.getItems();
      } catch (error) {
        console.log(error);
      }
    },
```

This POSTs a new item to the server, and when it is done it fetches the list of items again so that Vue will update the DOM with the new list.

You should be able to test this with the following:

```
node server.js
```

Visit `localhost:3000` in your browser. Notice that when you refresh the page, items are not lost now, because they are stored on the server

## Updating items

To support editing items (which is what we're doing when we mark one as complete), add the following method to `server.js`:

```
app.put('/api/items/:id', (req, res) => {
  let id = parseInt(req.params.id);
  let itemsMap = items.map(item => {
    return item.id;
  });
  let index = itemsMap.indexOf(id);
  if (index === -1) {
    res.status(404)
      .send("Sorry, that item doesn't exist");
    return;
  }
  let item = items[index];
  item.text = req.body.text;
  item.completed = req.body.completed;
  res.send(item);
});
```

In a REST API, we use a PUT request when we want to update an item. The URL will include the item `id`, such as `/api/items/5` to edit item number 5. Node will put this parameter into `req.params.id`.

We will *not* assume that the items are ordered in the `items` array by their `id`. So to find an item with a given `id` we will create an `itemsMap`, which is an array that contains just the item ids. We can then search through this array using `indexOf`. Once we have the item we want, we use the `text` and `completed` fields from `req.body` to change the item.

We return an error if the item isn't found. Like with other requests, we return a 200 OK and send the item back.

Let's test it! In one terminal:

```
node server.js
```

In another terminal:

```
curl -X POST -d '{"text":"get an A on the exam", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X POST -d '{"text":"party all night", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X GET localhost:3000/api/items
curl -X PUT -d '{"text":"party all night", "completed":true}' -H "Content-Type: application/json" localhost:3000/api/items/2
curl -X GET localhost:3000/api/items
```

## Connecting the front end

In `index.html`, we need to call a `completeItem` whenever an item is changed:

```
          <input type="checkbox" v-model="item.completed" @click="completeItem(item)" />
```

Then, in `script.js`, add the `completeItem` method:

```
    async completeItem(item) {
      try {
        const response = axios.put("/api/items/" + item.id, {
          text: item.text,
          completed: !item.completed,
        });
        this.getItems();
      } catch (error) {
        console.log(error);
      }
    },
```

This method uses `axios` to send the PUT request, with the required information in the body of the request.

Notice that when we add an item or edit an item we call `getitems` when it is done.  This enables us to be sure our copy of the data is in sync with the server. A different way to do this is to modify our local copy of the data but *only* if the API call succeeds. This would avoid fetching the entire list each time it is changed, but requires you to be careful to keep the data in sync properly.

You should be able to test this with the following:

```
node server.js
```

Visit `localhost:3000` in your browser.

## Deleting items

To support deleting items, add the following method to `server.js`:

```
app.delete('/api/items/:id', (req, res) => {
  let id = parseInt(req.params.id);
  let removeIndex = items.map(item => {
      return item.id;
    })
    .indexOf(id);
  if (removeIndex === -1) {
    res.status(404)
      .send("Sorry, that item doesn't exist");
    return;
  }
  items.splice(removeIndex, 1);
  res.sendStatus(200);
});
```

In a REST API, we use the DELETE method for deleting items. We again have the `id` of the item in the URL, and we get it from `req.params.id`. Just like with editing items, we can't assume the array is sorted by `id`, so we create a second array that contains item `ids` and then search with `indexOf` to find the item with the item we're looking for.

We return an error if the item isn't found. We return 200 OK if we are able to delete the item.

Let's test it! In one terminal:

```
node server.js
```

In another terminal:

```
curl -X POST -d '{"text":"get an A on the exam", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X POST -d '{"text":"party all night", "completed":false}' -H "Content-Type: application/json" localhost:3000/api/items
curl -X GET localhost:3000/api/items
curl -X DELETE localhost:3000/api/items/2
curl -X GET localhost:3000/api/items
```

## Connecting the front end

We need to modify the front end to call this API. In `script.js`, modify `deleteItem` as follows:

```
    async deleteItem(item) {
      try {
        const response = await axios.delete("/api/items/" + item.id);
        this.getItems();
      } catch (error) {
        console.log(error);
      }
    },
```

Notice how we get the list after the API call succeeds, so we Vue can update the DOM.

We also need to change `deleteCompleted`:

```
    deleteCompleted() {
      this.items.forEach(item => {
        if (item.completed)
          this.deleteItem(item);
      });
    },
```

This loops through the items and sends a request to the server to delete each completed item. These will each run asynchronously since `deleteItem` is an async function!

You should be able to test this with the following:

```
node server.js
```

Visit `localhost:3000` in your browser.
