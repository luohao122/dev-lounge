---
title: How to Setup Your Express app with GraphQL & ApolloServer V4!
series: "ow to Setup Your Express app with GraphQL & ApolloServer V4"
excerpt: A guide on how to Setup Your ExpressJs app with GraphQL & ApolloServer V4
published_date: 2025-03-08 
updated_date: 2025-03-08 
part: 1
tags:
  - javascript
  - typescript
  - nodejs
  - expressjs
  - graphql
  - apollo
---
# GraphQL, Apollo Server v4 + Express.js Setup

## Disclaimer

Some of the information in this guide may change in the future as the aforementioned packages get updated. This guide serves as a tutorial, not a strict requirement.

While you can set up GraphQL with Express.js without Apollo Server, using Apollo makes the development experience smoother and more efficient. Follow the steps below to install and configure the necessary packages.

---

## Project Folder Structure

```
src
|---- controllers
|---- graphql
|-------- schema
|-------- resolvers
|-------- index.ts
|---- middlewares
|---- services
|---- types
|---- routes
|---- utils
|---- app.ts
|---- routes.ts
|---- server.ts
|---- package.json
|---- package-lock.json
|---- tsconfig.json
|---- .env
```

### Source code (Starting Point - Main Branch)

The source code for this project is available in the repository linked below. If you want to follow along with this guide, you can use the main branch as a reference. However, you can also apply these instructions to your own project setup :

[Source Code - Main Branch](https://github.com/luohao122/minimal-api-app)

### Install Required Packages

```sh
npm install @apollo/server graphql @graphql-tools/merge @graphql-tools/schema cors graphql-ws ws
```

These additional packages help with merging type definitions, handling CORS, and enabling WebSocket support.

### Install Type Definitions

Run the following command to install the necessary type definitions for cors, express, and ws:

```sh
npm install --save-dev @types/cors @types/express @types/ws
```

### 🚀 Important Notes

#### **Specify the Correct `@types/express` Version**
Apollo Server v4 requires a specific version of `@types/express` to ensure compatibility.

In your `package.json`, set:

```json
"@types/express": "^4.17.21"
```

then run the below command again
```sh
npm install --save-dev  @types/express
```

### Adjust TypeScript Configuration

If you are using the provided source code, update your tsconfig.json to avoid type conflicts with Apollo Server v4:

```json
"module": "NodeNext",
"moduleResolution": "nodenext"
```

This ensures proper module resolution when using Apollo Server with TypeScript.

### Key Takeways
- Always use the main branch of the repository as the starting point, if you are using the provided source code.
- Ensure you install the correct version of @types/express for Apollo Server v4.
- Update tsconfig.json as shown above to prevent TypeScript conflicts.

---

## Setting Up GraphQL with Apollo Server

### 1. Creating a GraphQL Schema

A GraphQL schema consists of **Type Definitions** and **Resolvers**.

#### Defining Types

```ts
// src/graphql/schema/author.ts
import { buildSchema } from "graphql";

export const authorSchema = buildSchema(`#graphql
  type Author {
    firstName: String
    lastName: String
  }

  type AuthorResult {
    authors: [Author]!
  }

  type Query {
    getAuthors: AuthorResult
  }
`);
```

This schema defines:

- `Author`: Represents an author.
- `AuthorResult`: Contains an array of authors.
- `Query`: Supports a `getAuthors` query that returns an `AuthorResult`.

#### Example GraphQL Response

```json
{
  "authors": [
    { "firstName": "Hao", "lastName": "Luong" },
    { "firstName": "Jack", "lastName": "Luong" }
  ]
}
```

#### Writing Resolvers

```ts
// src/graphql/resolvers/authors.ts
export const AuthorResolver = {
  Query: {
    getAuthors() {
      return {
        authors: [{ firstName: "Hao", lastName: "Luong" }],
      };
    },
  },
};
```

Resolvers define how queries return data. The method name (`getAuthors`) must match the query defined in the schema.

#### Combining Resolvers

```ts
// src/graphql/resolvers/index.ts
import { AuthorResolver } from "./authors";

export const resolvers = [AuthorResolver];
```

---

### 2. Creating an Executable Schema

```ts
// src/graphql/index.ts
import { makeExecutableSchema } from "@graphql-tools/schema";
import { resolvers } from "./resolvers";
import { mergeTypeDefs } from "@graphql-tools/merge";
import { authorSchema } from "./schema/author";

const mergedSchema = mergeTypeDefs([authorSchema]);

export const schema = makeExecutableSchema({
  typeDefs: mergedSchema,
  resolvers,
});
```

This allows us to modularize GraphQL type definitions.

---

### 3. Setting Up Apollo Server

#### Import Necessary Packages

```ts
// src/server.ts
import http from "http";

import express, { json } from "express";
import dotenv from "dotenv";
import cors from "cors";

import { ApolloServer } from "@apollo/server";
import { expressMiddleware } from "@apollo/server/express4";
import { ApolloServerPluginDrainHttpServer } from "@apollo/server/plugin/drainHttpServer";

import { ApolloServerPluginLandingPageDisabled } from "@apollo/server/plugin/disabled";
import { ApolloServerPluginLandingPageLocalDefault } from "@apollo/server/plugin/landingPage/default";
import { WebSocketServer } from "ws";

import { useServer } from "graphql-ws/use/ws";
import { schema } from "./graphql";

import { appRoutes } from "@app/routes";
```

#### Initialize Express and HTTP Server

```ts
dotenv.config();

const PORT = 3001;

const app = express();
const httpServer = http.createServer(app);
```

#### Create Apollo Server Instance

```ts
const apolloServer = new ApolloServer({
  schema,
  plugins: [
    ApolloServerPluginDrainHttpServer({ httpServer }),
    {
      async serverWillStart() {
        return {
          async drainServer() {
            await serverCleanup.dispose();
          },
        };
      },
    },
    process.env.NODE_ENV === "production"
      ? ApolloServerPluginLandingPageDisabled()
      : ApolloServerPluginLandingPageLocalDefault({ embed: true }),
  ],
});
```

This setup ensures proper cleanup and conditionally enables the Apollo Server landing page in development.

---

### 4. Setting Up WebSocket for GraphQL Subscriptions

```ts
const wsServer = new WebSocketServer({
  server: httpServer,
  path: "/graphql",
  perMessageDeflate: false,
});

const serverCleanup = useServer({ schema }, wsServer);
```

WebSocket is required for real-time GraphQL subscriptions.

---

### 5. Configuring Middleware

```ts
const standardMiddleware = async () => {
  app.use(json({ limit: "200mb" }));
  app.use(cors());

  await apolloServer.start();

  app.use(
    "/graphql",
    cors({ origin: "*", credentials: true }),
    express.json({ limit: "200mb" }),
    express.urlencoded({ extended: true, limit: "200mb" }),
    expressMiddleware(apolloServer, {
      context: async ({ req, res }) => ({ req, res }),
    })
  );
};
```

Apollo Server v4 uses `expressMiddleware` instead of `applyMiddleware`.

---

### 6. Defining API Routes

```ts
const routesMiddleware = () => {
  appRoutes(app);
};
```

This step is optional; follow your own route setup if preferred.

---

### 7. Starting the Server

```ts
const startServer = async () => {
  try {
    console.log(`Server has started with process id ${process.pid}`);
    httpServer.listen(PORT, () => {
      console.log(`Server is running on port ${PORT}`);
    });
  } catch (error) {
    console.error(`startServer error`, error);
  }
};

export const start = async () => {
  await standardMiddleware();
  routesMiddleware();
  await startServer();
};
```

---

### 8. Testing the GraphQL Server

Run your server and visit:

```
http://localhost:3001/graphql
```

The Apollo Server playground will appear in development mode.

---

## Conclusion

You have successfully set up a **Node.js + Express.js + GraphQL** server with **Apollo Server v4**. This guide is intended for learning purposes, and the provided structure is not necessarily production-ready.

Happy coding! 🚀
