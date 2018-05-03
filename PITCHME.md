# Graph

---

@div[left-50]

@div
Graph API
@divend

@ul

* What are we using?
* Basic Patterns
* What is missing?
  @ulend

  @divend

@div[right-50]

@div
Forge
@divend

@ul

* How is it set up
* How to query
* How to mutate
* What is dodgy?
  @ulend

  @divend

---

## Graph

### What are we using

---

#### Apollo Server Express

Its an express server using Apollo's express plugin

#### Apollo Upload express

Apollo Upload express gives us the ability to send files as part of GraphQL Mutations

+++

```js
graphQLServer.use(
  '/graphql',
  secure,
  bodyParser.json(),
  apolloUploadExpress(),
  (req, res) =>
    graphqlExpress({
      schema: schema({
        authHeader: req.headers.authorization,
      }),
      tracing: true,
      cacheControl: true,
    })(req, res)
);
```

---

### Graph - Basic Patterns

#### Schema

The most important part of the GraphQL API is arguably the schema, it contains all
the type definitions for our entities.

+++

```js
import { makeExecutableSchema } from 'graphql-tools';
import resolvers from './resolvers';

export const typeDefs = `
type Query {
  tasks(projectId: Int): [Task]
}

type Mutation {
  updateTask(id: Int!, task: TaskUpdate!): Question
}

input TaskUpdate {
  position: Int
}

type Task {
  id: Int
  iconAsset: Asset
  description: String
  position: Int
  title: String
  type: String
  alias: String
  taskList: TaskList
  questions: [Question]
  condition: Condition
}
`;

const schema = ({ authHeader }) =>
  makeExecutableSchema({
    typeDefs,
    resolvers: resolvers({ authHeader }),
  });
```

---

#### Resolvers

Resolvers are where we tie our schema to where we can find our schema. Resolvers return
an object that mirror the schema. When you start the server and there is a mismatch
between what is defined in the resolver and in the schema it will shout at you.

+++

```js
import RestConnector from './connectors';

const resolvers = ({ authHeader }: { authHeader: string }) => {
  const Rest = RestConnector({ authHeader });

  return {
    Query: {
      tasks: (root: RootType, args: ProjectIdType) =>
        Rest.Task.allForProject(args.projectId),
    },
    Mutation: {
      updateTask: (
        root: RootType,
        { id, task }: { id: number, task: Object }
      ) => Rest.Task.update({ id, task }),
    }
    Task: {
     questions: ({ id: taskId }: IdType) => Rest.Question.allForTask(taskId),
   },
  }
}
```

---

#### Connectors

Connectors connect to the actual data and the GraphQL API. We only have a Rest
Connector at the minute, but we could have any connector we would like, in the
future, eg. a local database. This means the front end apps would not have to
care where any data is stored, and access it through the same mechanism.

+++

```js
import { signAssetCollection } from './signAsset';

const RestConnector = ({ authHeader }: { authHeader: string }) => {
  const authFetch = fetch({ authHeader });
  const authPatch = patch({ authHeader });
  const read = (endpoint, key) =>
    authFetch(endpoint).then(R.path(['data', key]));

  const Task = {
    allForProject: (projectId: number) =>
      read(`projects/${projectId}/tasks?include=icon_asset`, 'tasks').then(
        signAssetCollection('iconAsset')
      ),
    update: ({ id, task }: { id: number, task: Object }) =>
      authPatch(`tasks/${id}`, { task }).then(() => null),
  };

  const Question = {
    allForTask: (taskId: number) =>
      read(`tasks/${taskId}/questions`, 'questions'),
  };
};
```

---

### What is missing?

#### Unit Tests

* No unit tests for the Rest Connector

#### Integration Tests

* No end to end test to check an entity can be queried or mutated against the business API

+++

### What is there?

#### Integration Test

```js
test('the schema is valid', () => {
  const authHeader = 'abcdef';

  const result = schema({ authHeader });

  expect(result).toMatchSnapshot();
});
```

This is a shockingly useful test because it validates the whole schema against
the resolvers and gives you errors like:

```
Error: Project.turds defined in resolvers, but not in schema
```

---

# Graph is Good

---

## Client Side

### Set Up

---

#### Easy Peasy Lemon Squeezy

```js
import React from 'react';
import { ApolloProvider } from 'react-apollo';

function ApolloContainer({ children }: Props) {
  return <ApolloProvider client={client}>{children}</ApolloProvider>;
}
```

---

#### Difficult Difficult Lemon Difficult

```js
import { ApolloClient } from 'apollo-boost';
import { ApolloLink } from 'apollo-link';
import { createUploadLink } from 'apollo-upload-client';
import { InMemoryCache } from 'apollo-cache-inmemory';

const authLink = setContext((_, { headers }) =>
  getAccessToken().then((accessToken) =>
    getGraphUrl().then((graphUrl) => ({
      uri: graphUrl,
      headers: {
        ...headers,
        Authorization: accessToken ? `Bearer ${accessToken}` : '',
      },
    }))
  )
);

const link = ApolloLink.from([authLink, createUploadLink()]);

const client = new ApolloClient({
  link,
  cache: new InMemoryCache(),
  dataIdFromObject: (o) => o.id,
});
```

---

## Querying Data

---

#### Tasks Query

Fetch all the tasks for a project

```js
import { gql } from 'apollo-boost';

const TasksQuery = gql`
  query TasksQuery($projectId: Int!) {
    tasks(projectId: $projectId) {
      id
      title
      position
      taskList {
        title
      }
      iconAsset: {
        styles: {
          original
        }
      }
    }
  }
`;
```

---

### Query Component

```js
import React from 'react';
import { Query } from 'react-apollo';
import { gql } from 'apollo-boost';
import TaskListing from './TaskListing';
import LoadingManager from '../LoadingManager';

const TasksQuery = gql`
  ...
`;

function TaskListingContainer({ projectId }: Props) {
  return (
    <Query query={TasksListsQuery} variables={{ projectId }}>
      {LoadingManager(TaskListing)}
    </Query>
  );
}

export default TaskListingContainer;
```

---

### What is this LoadingManager?

```js
const LoadingManager = (ChildComponent, props) => ({
  loading,
  error,
  data,
}) => {
  if (loading) {
    return <div>Loading</div>;
  }

  if (error) {
    return <div>An unexpected error occurred</div>;
  }

  return <ChildComponent {...data} {...props} />;
};
```

---

## Mutating Data

---
