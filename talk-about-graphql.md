<!-- $theme: gaia -->
<!-- $size: 4:3 -->

# Building GraphQL server on *2018*

### [timqian](https://github.com/timqian)

---
# Me


- Nodejs backend at work
- Full stack at home
- Javascript is the best language ~~in the universe~~ to me

---

<!-- page_number: true -->

# Target audiance

- Some experience on building/using REST API
- Little experience on building/using GraphQL

---

# Table of contents
1. What
1. How
1. Why
1. Issues and Solutions

---

## What is an APP from the perspective of data?

# 🌟`{}`🌟

---

# `{}`

[![](https://user-images.githubusercontent.com/5512552/40905789-9022d7bc-6811-11e8-9024-97770b21ee44.png)](https://twitter.com/tjholowaychuk/status/957853652483416064)

---

## Frontend's Job:

1. Get the `{}` from backend
1. Render the page based on the `{}`
2. Update the `{}` based on user's input
3. Maybe store the update back to backend

---

## Example

In a blog system, we want to render the user with his blogs.

---

## How does frontend get the `{}` through REST API

```bash
GET /users/:id # get user info
GET /users/:id/blogs # get all blogs of user
```

```js
// The object frontend used to render the page
{
  user: {
    id: 1,
    username: 'timqian',
    blogs: [{
      id: 1,
      title: 'hi world'
      content: 'hello world'
    }]
  }
}
```

---

### How does frontend get the `{}` through GraphQL

```graphql
# GraphQL query
query {
  user(id: 1) {
    username
    blogs {
      id
      title
      content
    }
  }
}

```

---

### How does frontend get the `{}` through GraphQL

```js
// returned object from GraphQL backend
{
  user: {
    username: 'timqian',
    blogs: [{
      id: 1,
      title: 'hi world'
      content: 'hello world'
    }]
  }
}
```

Better experience for receiving more complicated object

---

### Want more info about the user and less about the blog?

```diff
query {
  user(id: 1) {
+   id
    username
    blogs {
-     id
      title
-     content
    }
  }
}

```

---

### Want more info about the user and less about the blog?

```js
// returned object from GraphQL backend
{
  user: {
    id: 1,
    username: 'timqian',
    blogs: [{
      title: 'hi world'
    }]
  }
}
```

---

## So *What* is GraphQL

GraphQL is a query language for your API. Basically it is about selecting fields on objects

---

# *How* to implement 

---

#### *How* to implement (1): Define Schema

- GraphQL query language is about selecting fields on objects
- We will need a **Schema** to describe/define the data we can ask for
- **Schema**: a set of types which completely describe the set of possible data you can query on that service

---

###### *How* to implement (1): Define Schema

```gql
# 1. Define schema
type Query {
  user(id: ID!): User!
  blogs: [Blog]
}
type User {
  id: ID!
  username: String!
  blogs: [Blog]
}
type Blog {
  id: ID!
  title: String!
  content: String!
  createdBy: User!
}
schema {
  query: Query 
  # mutation: Mutation
}
```

---

##### *How* to implement (2): Write resolvers
- **Schema** describes all of the fields, arguments, and result types
- Now we need a collection of functions that are called to actually execute these fields, and this collection of functions are called **Resolvers**

---

##### *How* to implement (2): Write resolvers

```js
// 2. Define resolvers as a nested object that
// maps type and field names to resolver functions
const resolver = {
  Query: {
    user: (obj, args) => daos.User.get(args.id),
    blogs: (obj, args) => daos.Blog.getAll(),
  },
  User: {
    blogs: (obj, args) => daos.Blog.getByUser(obj.id),
  },
  Blog: {
    createdBy: (obj, args) => daos.User.get(obj.createdBy),
  },
}
```

---

##### *How* to implement (3): Bind schema and resolver together

```js
// 3. Bind schema and resolver together using 
// graphql-yoga which is based on `graphql-tool`
import { GraphQLServer } from 'graphql-yoga';

const server = new GraphQLServer({ typeDefs, resolvers });

server.start(() => 
  console.log('Server is running on localhost:4000'));
```

---


## Summary of implementing a GraphQL server

1. Write a **Schema** to define the data graph
2. Write **Resolvers** to resolve fileds of the defined data graph

---

## Summary of implementing a GraphQL server

1. Write a **Schema** to define the data graph
2. Write **Resolvers** to resolve fileds of the defined data graph

# So Easy!

but it wasn't that easy before [graphql-tool](https://github.com/apollographql/graphql-tools) was invented

---

# A bit history of GraphQL

---

###### How GraphQL server looks before [graphql-tool](https://github.com/apollographql/graphql-tools) existed

```js
import {
  graphql,
  GraphQLSchema,
  GraphQLObjectType,
  GraphQLString
} from 'graphql';
var schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: 'RootQueryType',
    fields: {
      hello: {
        type: GraphQLString,
        resolve() {
          return 'world';
        }
      }
    }
  })
}); // Mixing schema and resolver together 

```

---

##### History of popularity: 2015 (bottleneck)

<img width="550" alt="screen shot 2018-06-09 at 1 31 04 am" src="https://user-images.githubusercontent.com/5512552/41172589-e2048472-6b86-11e8-94f1-945bd446a678.jpg">

###### - timqian.com/star-history


---

##### History of popularity: 2016 -2018 (boom)

<img width="800" alt="screen shot 2018-06-09 at 1 45 37 am" src="https://user-images.githubusercontent.com/5512552/41172583-dd36ad3a-6b86-11e8-9d06-bbb41fd40766.png">

###### - timqian.com/star-history

---

# *Why* choose GraphQL over REST

- Performance
	- Less roundtrips
- Development experance
	- Self documented (no outdated apidoc anymore!)
	- Less endpoints
	- Ask for what you want
	- Real-time data push (subscription)

---

# Issues and Solutions

- N+1 problem
- Writing test
- Similar code for normal usage

---

# Issue (1): N+1 problem

```gql
# Will do N + 1 database query if there is N blogs
query {
  blogs {
    id
    title
    createdBy {
      id
      name
    }
  }
}

```

Situation can be worse when the query becomes more complex.

---


# Can we do the N user query together?

&nbsp;

---

## Solution: `Dataloader` (1)

```js
const DataLoader = require('dataloader');

// Provid a batch loading function
const myBatchGetUsers = ids => 
  daos.User.whereIn('id', ids);

// Create your data loader
const userLoader = 
  new DataLoader(myBatchGetUsers);
```

---

## Solution: `Dataloader` (2)

Update resolver
```diff
const resolver = {
  Query: {
    user: (obj, args) => daos.User.get(args.id),
  },
  User: {
    blogs: (obj, args) => daos.Blog.getByUser(obj.id),
  },
  Blog: {
-  createdBy: (obj, args) => daos.User.get(obj.createdBy),
+  createdBy: (obj, args) => 
+    userLoader.load(obj.createdBy),
  },
}
```

---

## `Dataloader` Caching
```js
load(key)

clear(key)

loadMany(keys)

clearAll()

```

---


## How does Dataloader work

DataLoader will coalesce all individual loads which occur within a single frame of execution (a single tick of the event loop) and then call your batch function with all requested keys.

<img width="400" alt="screen shot 2018-06-02 at 9 26 24 pm" src="https://cloud.githubusercontent.com/assets/5512552/18698316/b77688e0-7ffb-11e6-9991-cd4c5dad85b5.png">

---

## Application level dataloader?

The official [Readme](https://github.com/facebook/dataloader#creating-a-new-dataloader-per-request) encourage user to create a new DataLoader per request. Because:
1. Many different users with different access permissions. It may be dangerous to use one cache across many users
2. In memary cache can not scale among servers

But they are actually solvable

---

## Application level dataloader?

1. Use dataloader in the dao layer of your app and do ACL on resolver
2. Use redis/memcached as the cache of dataloader

### Refs

- [Disscussions on an issue of dataloader repo](https://github.com/facebook/dataloader/issues/62#issuecomment-392477844)
- [Use redis instead of memary as the cache](https://github.com/DubFriend/redis-dataloader)


---

# Issue (2): Writing test query(String) is error prone 

---

# Issue (2): Writing tests

```gql
# Sample schema
type Query {
    user(id: Int!): User!
}

type User {
    id: Int!
    username: String!
    email: String!
    createdAt: String!
}
```

---

# Issue (2): Writing tests

```gql
# Sample query
query {
    user(id: 1) {
        id
        username
        email
        createdAt
    }
}
```

---

# Issue (2): Writing tests

```gql
# Sample query
query user($id: Int!) {
    user(id: $id) {
        id
        username
        email
        createdAt
    }
}
```

[gql-generator](https://github.com/timqian/gql-generator): generate sample queries for you based on the schema

---

### Issue (3): 70% of the resolvers are doing similar things: query db by ID; query db by foreign key.

---

### Issue (3): 70% of the resolvers are doing similar things: query db by ID; query db by foreign key.

**[Prisma](https://github.com/prismagraphql/prisma)**: Automatically mapping your API to database: 

Define your types and it will do the resolves for you.

---

<img width="708" alt="screen shot 2018-06-09 at 2 14 32 am" src="https://user-images.githubusercontent.com/5512552/41173845-ef795250-6b8a-11e8-9936-5d07da440524.png">

---

<img width="908" alt="screen shot 2018-06-09 at 2 02 01 am" src="https://user-images.githubusercontent.com/5512552/41173887-08789766-6b8b-11e8-96cc-49ac944fad51.png">

<img width="906" alt="screen shot 2018-06-09 at 2 01 51 am" src="https://user-images.githubusercontent.com/5512552/41175361-64ec5ef2-6b8f-11e8-8ce9-6a5fe0d37219.png">


---

## Summary

- Building GraphQL is easy and the tool chain is still evolving
- GraphQL benifits both frontend and backend
- Give it a try in your next awsome project 😉

---


# Questions?
