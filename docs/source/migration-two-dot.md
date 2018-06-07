---
title: Migrating to v2.0 Beta
description: How to migrate to Apollo Server 2.0 beta
---

The Apollo Server 2.0 beta dramatically simplifies the API for building a GraphQL server without compromising on features. It's also completely backward compatible, so you don't have to worry about breaking changes when upgrading.

While it's possible to migrate an existing server to the 2.0 beta without any changes, we recommend changing to new patterns we're suggesting in order to take advantage of all the latest Apollo Server features, reduce the boilerplate, and enable future flexibility.  To learn how to migrate to the 2.0 beta from version 1.0, please read the following guide.

> **Note:** In the beta of Apollo Server 2.0 only Express and Hapi are supported.  Additional middleware integrations will be implemented in the official 2.0 release.

### The `gql` tag

Apollo Server 2.0 ships with the `gql` tag for **editor syntax highlighting** and **auto-formatting** with Prettier.  In the future, we will be using it for statically analyzing GraphQL queries, so Apollo Servers requires wrapping your schema with `gql`.

The `gql` tag parses the query string into an AST  and is now exported from the new `apollo-server` package.

```js line=1,3
const { ApolloServer, gql } = require('apollo-server');

const typeDefs = gql`
  type Query {
    hello: String
  }
`;

//Some projects use schemas imported from external files
const typeDefs = gql`${IMPORT_FUNCTION('./schema-file')}`;

//gql can also be used as regular function to convert a string to an AST
const typeDefs = gql(IMPORT_FUNCTION('./schema-file'))
```

### Changes to app dependencies

> Apollo Server 2.0 beta requires Node.js v6 and higher.

Apollo Server 2.0 simplifies implementing a GraphQL server.  Apollo Server 1.0 revolved around providing middleware-based solutions, which had to be added to an application which already existed.  These middleware implementations were tied to the HTTP server in use (e.g. `apollo-server-express` for Express implementations, `apollo-server-koa` for Koa, etc.).

There is a consideration to be made when following the rest of the guide:

* [**Middleware option**](#Middleware): If the application being migrated implements Apollo Server alongside other middleware, there are some packages which can be removed, but adding the `apollo-server` package and switching to using the new `registerServer` API should still simplify the setup.  In this case, check the [Middleware](#Middleware) section.
* [**Stand-alone option**](#Stand-alone): If the application being migrated is only used as a GraphQL server, Apollo Server 2.0 _eliminates the need to run a separate HTTP server_ and allows some dependencies to be removed.  In these cases, the [Stand-alone](#Stand-alone) option will reduce the amount of code necessary for running a GraphQL server.

### Simplified usage

Check out the following changes for Apollo Server 2.0 beta.

* You no longer need to import `body-parser` to set up `apollo-server-express`.
* You no longer need to import `makeExecutableSchema` from `graphql-tools`.
* You no longer need to import `graphqlExpress` and `graphiqlExpress` from `apollo-server-express`.
* You should pass in `typeDefs` and resolvers as parameters to an instance of Apollo Server.
* If the server is only functioning as a GraphQL server, it's no longer necessary to run your own HTTP server (like `express`).

### Middleware

With the middleware option used by Apollo Server 1.0 users, it is necessary to install the beta version of `apollo-server-express` and also add the new `apollo-server` beta.  To do this, use the `beta` tag when installing:

    npm install --save apollo-server-express@beta apollo-server@beta

The changes are best shown by comparing the before and after of the application.

#### Apollo Server 1 (old pattern)

An example of using Apollo Server 1 with the Express framework:

```js
const express = require('express');
const bodyParser = require('body-parser');
const { makeExecutableSchema } = require('graphql-tools');
const { graphqlExpress } = require('apollo-server-express');

const typeDefs = `
  type Query {
    hello: String
  }
`;

const resolvers = {
  Query: {
    hello: () => 'Hello world!'
  },
}

const myGraphQLSchema = makeExecutableSchema({
  typeDefs,
  resolvers,
});

const PORT = 3000;

const app = express();

// bodyParser is needed just for POST.
app.use('/graphql', bodyParser.json(), graphqlExpress({ schema: myGraphQLSchema }));

app.listen(PORT);
```

#### Apollo Server 2 (new pattern)

Now, you can just do this instead:

```js
const express = require('express');
const { ApolloServer, gql } = require('apollo-server');
const { registerServer } = require('apollo-server-express');

const app = express();

const typeDefs = gql`
  type Query {
    hello: String
  }
`;

const resolvers = {
  Query: {
    hello: () => 'Hello world!'
  },
};

const server = new ApolloServer({ typeDefs, resolvers });
registerServer({ server, app });

// normal ApolloServer listen call but url will contain /graphql
server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`)
});
```

### Stand-alone

If you are simply focused on running a production-ready GraphQL server quickly, Apollo Server 2.0 ships with a built-in server and starting your own server (e.g. Express, Koa, etc.) is no longer necessary.

For these cases, it's possible to remove the existing `apollo-server-{integrations}` package and add the new `apollo-server` beta.  If using Express, this can be done by running:

    npm uninstall --save apollo-server-express

    npm install --save apollo-server@beta

An implementation with this pattern would look like:

```js
const { ApolloServer, gql } = require('apollo-server');

// Construct a schema, using GraphQL schema language
const typeDefs = gql`
  type Query {
    announcement: String
  }
`;

// Provide resolver functions for your schema fields
const resolvers = {
  Query: {
    announcement: () =>
      `Say hello to the new Apollo Server! A production ready GraphQL server with an incredible getting started experience.`
  }
};

const server = new ApolloServer({ typeDefs, resolvers });

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

### Adding Additional Middleware to Apollo Server 2

For middleware that is collocated with the GraphQL endpoint, Apollo Server 2 allows middleware mounted on the same path before `registerServer` is called. For example, this server runs an authentication middleware before GraphQL execution.

```js
const express = require('express');
const { ApolloServer, gql } = require('apollo-server');
const { registerServer } = require('apollo-server-express');

const app = express();
const path = '/graphql';

const server = new ApolloServer({ typeDefs, resolvers });

//Mount a jwt or other authentication middleware that is run before the GraphQL execution
app.use(path, jwtCheck);

registerServer({ server, app, path });

// normal ApolloServer listen call but url will contain /graphql
server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`)
});
```

### Using an Existing Schema

For many existing instances of Apollo Server, the schema is created at runtime before server startup, using `makeExecutableSchema` or `mergeSchemas`. Apollo Server 2 stays backwards compatible with these more complex schemas, accepting it as the `schema` field in the server constructor options. Additionally, Apollo Server 2 exports all of `graphql-tools`, so `makeExecutableSchema` and other functions can be imported directly from Apollo Server.

> Note: the string to create these schema will not use th `gql` tag exported from apollo-server.
```js
const { ApolloServer, makeExecutableSchema } = require('apollo-server');

//For developer tooling, such as autoformatting, use the following workaround
const gql = String.raw;

const typeDefs = gql`
  type Query {
    hello: String
  }
`;

const schema = makeExecutableSchema({
  typeDefs,
  resolvers,
});
//mergeSchemas is imported from apollo-server
//const schema = mergeSchemas(...);

const server = new ApolloServer({ schema });

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```