<h1 align="center"><strong>How to run your graphql-yoga server on AWS Lambda with Apex Up</strong></h1>

<br />

Please feel free to make suggested improvements and raise issues. I am relatively new to Typescript and GraphQL.

## What we will use

- [`graphql-cli`](https://github.com/graphql-cli/graphql-cli) - to initalize the project and download graphql-yoga and the boilerplate code.
- [`graphql-yoga`](https://github.com/prisma/graphql-yoga) - based on Apollo Server & Express.
- [`graphql-boilerplates/typescript-graphql-server`](https://github.com/graphql-boilerplates/typescript-graphql-server) - boilerplate code for the initial server.
- [`graphcool/prisma`](https://github.com/graphcool/prisma) - to handle your database and much more
- [`apex/up`](https://github.com/apex/up) - to deploy our app to AWS Lambda and manage everything from uploading node modules to S3, API Gateway, staging and logging.
- [`Microsoft/TypeScript`](https://github.com/Microsoft/TypeScript) - to write the server code.

If you're just starting out with graphql and graphql-yoga, you may find the following tutorial helpful

https://www.howtographql.com/graphql-js/0-introduction/

## Let's get started

Install [GraphQL CLI](https://github.com/graphql-cli/graphql-cli) to bootstrap your GraphQL server:

```sh
npm install -g graphql-cli
```

Use graphql-cli to create your project:

```sh
graphql create up-graphql-yoga-server-example --boilerplate typescript-advanced
```

As it creates the boilerplate project, it will ask you to choose where you'd like to deploy prisma. Select either `prisma-eu1` or `prisma-us1`. In my case I chose, `prisma-eu1` which is hosted in `eu-west-1`. This meant I deployed my server to the same AWS region.

Navigate to the new project and check if the server starts
```sh
cd up-graphql-yoga-server-example

# This will start the server on http://localhost:4000 and open GraphQL Playground in your browser.
yarn dev
```

You will need to compile your Typescript into Javascript so it can run on AWS Lambda. Edit your tsconfig.json file to look like. At time of writing, AWS Lambda only supports Node.js v4.3.2 and 6.10.3 (see [currently supported runtimes](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html):

```json
{
  "compilerOptions": {
    "target": "es6",
    "lib": [ "es6", "dom", "esnext.asynciterable" ],
    "moduleResolution": "node",
    "module": "commonjs",
    "sourceMap": true,
    "rootDir": "src",
    "outDir": "dist"
  }
}
```

**Apex Up** references the `build` and `start` commands in package.json. So you will need to edit the script section of your package.json file so that `start` runs the compiled Javascript code.

```json
"scripts": {
  "local-start": "dotenv -- nodemon -e ts,graphql -x ts-node src/index.ts",
  "dev": "npm-run-all --parallel local-start playground",
  "debug": "dotenv -- nodemon -e ts,graphql -x ts-node --inspect src/index.ts",
  "playground": "graphql playground --port 5000",
  "build": "rimraf dist && tsc",
  "start": "node dist/index.js",
  "local-start-js": "dotenv -- node dist/index.js"
},
```

## Installing and using Apex Up

Many of you will find the [`Apex Up`](https://up.docs.apex.sh/) documentation helpful and well written.

Install up

```sh
npm i -g up
```

On AWS IAM, create a new policy as follows:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "acm:*",
                "cloudformation:Create*",
                "cloudformation:Delete*",
                "cloudformation:Describe*",
                "cloudformation:ExecuteChangeSet",
                "cloudformation:Update*",
                "cloudfront:*",
                "cloudwatch:*",
                "ec2:*",
                "ecs:*",
                "events:*",
                "iam:AttachRolePolicy",
                "iam:CreatePolicy",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:DeleteRolePolicy",
                "iam:GetRole",
                "iam:PassRole",
                "iam:PutRolePolicy",
                "lambda:AddPermission",
                "lambda:Create*",
                "lambda:Delete*",
                "lambda:Get*",
                "lambda:InvokeFunction",
                "lambda:List*",
                "lambda:RemovePermission",
                "lambda:Update*",
                "logs:Create*",
                "logs:Describe*",
                "logs:FilterLogEvents",
                "logs:Put*",
                "logs:Test*",
                "route53:*",
                "route53domains:*",
                "s3:*",
                "ssm:*",
                "sns:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "apigateway:*",
            "Resource": "arn:aws:apigateway:*::/*"
        }
    ]
}
```

Create a new user on IAM with the newly created policy and save the keys to `~/.aws/credentials`

```
[name-of-newly-created-aws-profile]
aws_secret_access_key = insert-key-here
aws_access_key_id = insert-id-here
```

Create an `up.json` file in the parent directory. You might it easier to copy it from [`up.json`](./up.json)

```json5
{
  "name": "name-of-server",
  "profile": "name-of-newly-created-aws-profile", // references your AWS credentials
  "regions": ["eu-west-1"], // the region you wish to deploy to. Typically the same as your prisma deployment
  "lambda": {
    "memory": 512 // how much memory you wish to allocate to lambda. For for details see Apex Up docs
  },
  "proxy": {
    "command": "npm start", //how up starts the server
    "timeout": 25, //timeout of AWS lambda function
    "listen_timeout": 15,
    "shutdown_timeout": 15
  },
  "stages": {
    "development": {
      "proxy": {
        "command": "yarn dev" //called by `up start` to launch dev server
      }
    }
  },
  "environment": { // environmental variables are entered here so update it for your secret and prisma cluster. Apex Up Pro offers encrypted variables so you don't need to save it to Git
    "NODE_ENV": "dev",
    "PRISMA_STAGE": "dev",
    "PRISMA_ENDPOINT": "https://eu1.prisma.sh/public-generated-name/name-of-prisma-server/dev",
    "PRISMA_CLUSTER": "public-generated-name/prisma-eu1",
    "PRISMA_SECRET": "mysecret123",
    "APP_SECRET": "jwtsecret123"
  },
  "error_pages": { //if some one gets lost or goes to the wrong endpoint. An error page is displayed with mailto link
    "variables": {
      "support_email": "support@your-domain.com",
      "color": "#2986e2"
    }
  },
  "cors": { //setting up cors for your local client
    "allowed_origins": ["http://localhost:3000"],
    "allowed_methods": ["HEAD", "GET", "POST"],
    "allowed_headers": ["*"]
  }
}
```

`up` reads `.gitignore` first then `.upignore`. Given that you're using Typescript you don't want ./dist to be saved to Git but `up` needs to deploy it to AWS Lambda to run your Javascript server. So `up` can negate declared `.gitignore` folders/files by adding an exclamation point to `dist`.

Create a `.upignore` file with the following. 

```
!dist
```

## Deploy and test your server

Run your server locally
```sh
up start
```

Deploy your server to staging

```sh
up
```

Deploy your server to staging

```sh
up deploy production
```

**Note** You need to use `GraphQLServer` (not `graphqlLambda`)

Test the server by opening up the playground in your browser. Double check the url in the playground matches the api domain i.e. my-aws-api-url.com/development.

Open the staging endpoint using the `-o` flag.

```sh
up url -o
```

To open your production server use:

```sh
up url -o production
```

Access your logs

```sh
up logs
```
