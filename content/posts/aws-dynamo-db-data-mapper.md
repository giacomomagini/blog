---
title: "AWS DataMapper for DynamoDB and Typescript — Practical examples"
date: 2020-11-01T15:57:07Z
draft: false
tags: [aws, db, nosql, dynamodb, backend, typescript, nodejs]
---
## Introduction

DynamoDB is a really powerful product, especially when it comes to scalability, reliability and maintainability.
If SQL is out of the table and you run app on AWS you should consider to use it but this not what I'm gonna talk about.

### SDK and NodeJS

Performing operations on Dynamo's tables using the [JavaScript SDK](https://www.npmjs.com/package/aws-sdk) on NodeJS it might feel a bit cumbersome due to the absolute flexibility and the number of options available for each operation ([DyanmoDB official docs](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/DynamoDB.html)) despite TypeScript is fully supported.
But fear not, there's an official alternative!

### AWS DynamoDB DataMapper

[DynamoDB DataMapper](https://github.com/awslabs/dynamodb-data-mapper-js) is a collection of packages aiming to reduce development friction and boilerplate when using DynamoDB on NodeJS with Typescript.
Unfortunately, the documentation isn't exhaustive and guessing how to use some of the different components together it might take some time and sourcecode browsing.

### Let's shed some light on it

I've been working with DynamoDB for a little bit and I've gone through some basic operations that weren't as trivial as they would have been with some good documentation or examples.
I'm gonna go through them and try to explain as best as possible how to use this tool instead of reinventing the wheel.

## Let's start

### Packages

The code is split in 7 packages.

* @aws/dynamodb-auto-marshaller
* @aws/dynamodb-batch-iterator
* @aws/dynamodb-data-mapper-annotations
* @aws/dynamodb-data-mapper
* @aws/dynamodb-data-marshaller
* @aws/dynamodb-expressions
* @aws/dynamodb-query-iterator
  
If you don't find a type or class inside the main package @aws/dynamodb-data-mapper, it might be inside another package and github code search will be on your side.

https://github.com/awslabs/dynamodb-data-mapper-js

### Data types

Although the DataMapper for TypeScript supports complex types such as Date, Dynamo only supports *4* primitive data types *S (String)*, *B (Binary)*, *BOOL (0,1)* and *N (Number)*. It is important to understand how TS types will be mapped onto dynamo's ones to estimate the amount of storage we need (and we will pay for).

A practical example is Date. You can use the Date class and DataMapper will store your date as ISO-8601 formatted string.
This means every date will take 25 Bytes.
Let's assume we don't care about having readable dates on the table we can use UNIX time and store our date as a number that takes only 3 Bytes.

DataMapper allows us to specify a custom *marshaller* for each attribute in the model definition. We'll see how to to do this in the next part.

### Define a table

Below a mapper definition for a table called **dog_table**.

```typescript
import {
  attribute,
  autoGeneratedHashKey,
  table
} from "@aws/dynamodb-data-mapper-annotations";
import { DynamoDB } from "aws-sdk";

import { config } from "../config";

const dateMarshall = (value: Date): DynamoDB.AttributeValue =>
  ({ I: value.getTime() } as DynamoDB.AttributeValue);
const dateUnmarshall = ({ N }: DynamoDB.AttributeValue): Date | undefined =>
  N ? new Date(N) : undefined;

@table("dog-table")
export class Dog {
  @autoGeneratedHashKey()
  id?: string;
  @attribute()
  name?: string;
  @attribute()
  owner?: string;
  @attribute({ marshall: dateMarshall, unmarshall: dateUnmarshall })
  created_at?: Date;
  @attribute({ marshall: dateMarshall, unmarshall: dateUnmarshall })
  updated_at?: Date;
  @attribute()
  deleted?: boolean;
  @attribute({ marshall: dateMarshall, unmarshall: dateUnmarshall })
  deleted_at?: Date;
}

```

I took several design decisions I'm gonna explain:

1. Our `id` will be generated automatically when not specified in the mapper instance. It will be a UUID v4. If you want to handle *hashKey* and *rangeKey* I recommend to take a look at the @aws/dynamodb-data-mapper-annotations README.
2. I chose to store dates as *milliseconds after the UNIX epoch* to save space. To do this I had to specify a *marshall* and an *unmarshall* function.
3. I went for soft deletion adding a boolean attribute `deleted` and a date `deleted_at` to log when that happened.

### Setting up a client

Setting up a mapper instance it's as simple as this

```typescript

import { DataMapper } from "@aws/dynamodb-data-mapper";
import { DynamoDB } from "aws-sdk";

const dynamoDBOptions: DynamoDB.ClientConfiguration = {
  region: "<your_region>",
  // ...
};

const client = new DynamoDB(dynamoDBOptions);
export const mapper = new DataMapper({ client });

```

### Create new records

```typescript
import { Dog } from "./dog";
import { mapper } from "./mapper";

const newDog = new Dog();
newDog.name = "Pluto";
newDog.owner = "Jhon";
newDog.created_at = new Date();
newDog.updated_at = new Date();
const res = await mapper.put(newDog);
// mapper.put(newDog).then(...) if you're not using async/await

```

As you might note here I didn't specify `deleted` and `deleted_at`. This is possible because DynamoDB only needs the key to store a record, the other attributes are optional. Not adding these attributes it's small space optimisation.

### Read records

```typescript

import { Dog } from "./dog";
import { mapper } from "./mapper";

const dog = new Dog();
dog.id = "<uuid>";
const res = await mapper.get(dog);
```

### Update existing records (partial update)

Performing a partial update is certainly the most tricky operation in this post.
There are three important points:

1. Update the attributes ONLY if a record with the given `id` already exists, otherwise, it would turn into a create.
2. Update the attributes ONLY if it's not `deleted`.
3. Update some attributes without erasing the one we didn't specify.

```typescript
import { attributeExists, notEquals } from "@aws/dynamodb-expressions";

import { Dog } from "./dog";
import { mapper } from "./mapper";

const updateDog = new Dog();
updateDog.id = "<uuid>";
updateDog.name = "Plutone";
updateDog.updated_at = new Date();

const updatedDog = await mapper.update(updateDog, {
    onMissing: "skip",
    condition: {
      type: "And",
      conditions: [
        {
          ...attributeExists(),
          subject: "id",
        },
        { ...notEquals(true), subject: "deleted" },
      ],
    },
});

```

We achieved the point number **1** adding the expression `{...attributeExists(), subject: "id"}` to the conditions. Expressions as conditions tell dyanamodb whether an operation is valid or not.

Point **2** was achieved in the same way, combining the previous expression with `{ ...notEquals(true), subject: "deleted" }`. To know more about expressions and how to combine them take a look at the package's README @aws/dynamodb-expressions.

Finally, point **3** was achieved setting the config attribute `onMissing` to `"skip"`. This tell to the data mapper to ignore any undefined attribute in the `dog` instance. On the contrary, if set to `"remove"`, any missing attribute will be removed by the table record.

### Query with an index

DynamoDB is a NoSQL database but it allows you to create [Secondary Indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html) to **query** your data in an efficient way. The alternative would be a **scan** operation that usually uses a lot of resources ($$$).

Back to our example, let's assume we created a **Global Secondary Index** called `owner_index` that allows us to query the table by the attribute `owner`.

```typescript
import { Dog } from "./dog";
import { mapper } from "./mapper";

for await (const dog of mapper.query(
    Dog,
    { owner: "Giacomo" },
    {
      indexName: "owner_index",
      filter: { ...notEquals(true), subject: "deleted" },
    },
 }
)) {
  console.info(dog.name); // Do something with "dog"
}

```

If your **index** has more than one key, you can add it the same way I added `owner`.

If you're not opting for **soft deletion**, this code is still valid except for the line `filter: { ...notEquals(true), subject: "deleted"` that you have to remove.

NOTE: I'm not handling pagination but it is available on DynamoDB. Consider to use it when the amount of record can be high. Have a look at the package RAEDME [@aws/dynamodb-query-iterator](https://github.com/awslabs/dynamodb-data-mapper-js/tree/master/packages/dynamodb-query-iterator)

### Soft deletion

As I already said, I'm assuming we don't want to delete the record physically but we want just mark it as `deleted`.
This will have an impact on the storage if you don't need trace when the deletion has happened or you will never need the data in the future (i.e. for analysis) I'd suggest going for hard deletion which I will show in the next paragraph.

```typescript

import { attributeExists, notEquals } from "@aws/dynamodb-expressions";

import { Dog } from "./dog";
import { mapper } from "./mapper";

const deleteDog = new Dog();
deleteDog.id = "<uuid>";
deleteDog.deleted = true;
deleteDog.deleted_at = new Date();

const deletedDog = await mapper.update(deleteDog, {
    onMissing: "skip",
    condition: {
      type: "And",
      conditions: [
        {
          ...attributeExists(),
          subject: "id",
        },
        { ...notEquals(true), subject: "deleted" },
      ],
    },
});

```

Is it just an update? Well, it is :D

### Delete records

As simple as...

```typescript

import { Dog } from "./dog";
import { mapper } from "./mapper";

mapper.delete(updateDog, Object.assing(new Dog(), { id: "<uuid>" }));

```

NOTE: I've avoided using this shorter pattern to assign attribute value to mapper instance (`Dog`) cause it could lead to security issues or mere mistakes. The second parameter of the **Object.assing** at runtime could contain everything no matter the type, therefore we could accidentally update a record instead of creating a new one if your object contains a property `id`. Long story short, be mindful when using **Object.assing**!

### Types

One note on the packages types.
They might get awkward and push you to use them in the wrong way. One example are mapper classes (like `Dog`), they can't have a constructor because of the `@table` annotation and this means all the attributes have to be optional.
In general, be careful.

## Final

Thanks for reading! If you have any question reach me through one of my social profile!

Hope you enjoyed it!

Cheers