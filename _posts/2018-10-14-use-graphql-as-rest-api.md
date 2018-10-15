---
layout: post
title: Using GraphQL as REST API
date: 2018-10-14
tags: [ GraphQL, REST ]
---

# Introduction
[GraphQL](https://graphql.org/) helps to define a structure or data model at server side and it brings by nature a set of features to query and modify this data (for clients). 

This a quick example of GraphQL query language can do:

```
Query:
{ 
    posts {
        id
        title
    }
}
```

Results:

```
{
    posts: [{
        id: "XX",
        title: "YY"
    }, 
    ... ]
}
```

Other features:
- We can mark fields as deprecated
- Properties to limit the output

```
{ 
    posts (first : "10") {
        id
        title
    }
}
```

# Getting started

We followed the instructions in (here)[https://graphql.org/graphql-js/]:

```bash
npm init
npm install --save graphql
```

## Install dependencies

```bash
npm install --save express express-graphql request request-promise
```

And then we can start writing our graphql application as:

```js
const express = require('express');
const graphqlHTTP = require('express-graphql');
const schema = require('./schema');

const app = express();
app.use(graphqlHTTP({
  schema: schema,
  pretty: true,
  graphiql: true,
}));
app.listen(4000, () => {
  console.log('Server running at http://localhost:4000')
});
```

This will expose graphql via [graphical UI](https://github.com/graphql/express-graphql) using node express.

## Prepare the schema

In our app, we imported the schema module. Now, we need to write it:

```js
const { GraphQLInt,
    GraphQLString,
    GraphQLBoolean,
    GraphQLID,
    GraphQLList,
    GraphQLObjectType,
    GraphQLSchema
 } = require('graphql');

const BASE_URL = 'http://localhost:4000/';

var allAuthors = {
    // INIT YOUR DATA HERE
};

var allPosts = {
    // INIT YOUR DATA HERE
};

// Construct a schema, using GraphQL schema language
const Author = new GraphQLObjectType({
    name: "Author",
    fields: {
        id: { type: GraphQLInt },
        name: { type: GraphQLString }
    }
});

const Post = new GraphQLObjectType({
    name: "Post",
    fields: {
        id: { type: GraphQLInt },
        title: { type: GraphQLString },
        author: {
            type: Author,
            resolve: (rawPostData) => {
                let authorId = rawPostData.authorId;
                return request(BASE_URL + 'authors/${authorId}').then(JSON.parse); // REST Hateos to author entity
            }
        }
    }
});

const QueryType = new GraphQLObjectType({
    name: "Blog",
    fields: {
        title: { type: GraphQLString },
        authors: {
            type: GraphQLList(Author),
            args: {
                id: { type: GraphQLInt }
            },
            resolve: function (_, { id }) {
                return allAuthors[id];
            } 
        },
        posts: {
            type: GraphQLList(Post),
            args: {
                id: { type: GraphQLInt }
            },
            resolve: function (_, { id }) {
                return allPosts[id];
            }
        }
    }
    
});

module.exports = new GraphQLSchema({
    query: QueryType
});
```

Highligths:
- We first declare the GraphQL types
- Then our schema Blog contains a list of Post and a Post is written by an Author.
- Finally, we mark the discoverable fields with the request method as we do in Post for authors.

## Create some data:

In the above code snippet, we are using an in-memory dictionary to store our data. The only thing to note here is that we can use any other internal storage or even an API call. This is what the resolve method is for. 

More resources I'll explore in the future:
- https://graphql.org/graphql-js/mutations-and-input-types/
- https://graphql.org/graphql-js/constructing-types/

# Conclusion

The source of this quick introduction is from: https://www.infoq.com/presentations/rest-graphql
Git Hub repo is here: https://github.com/Sgitario/graphql-as-rest