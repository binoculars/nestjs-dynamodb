# NestJS Typed DynamoDB

## Description

Opinated way to use DynamoDB with NestJS and typescript, heavily inspired by [nest-typegoose](https://github.com/kpfromer/nestjs-typegoose)

## Getting Started

First install this module

`yarn add nestjs-typed-dynamodb`

Notice that it will install [dynamodb-data-mapper-annotations](https://github.com/awslabs/dynamodb-data-mapper-js/tree/master/packages/dynamodb-data-mapper-annotations) as a dependency

## Usage

In order to create a DynamoDB connection

**app.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { TypegooseModule } from 'nestjs-typed-dynamodb'
import { CatsModule } from './cat.module.ts'

@Module({
  imports: [
    DynamoDBModule.forRoot({
      AWSConfig: {
        region: 'local',
        accessKeyId: 'null',
        secretAccessKey: 'null',
      },
      dynamoDBOptions: {
        endpoint: 'localhost:8000',
        sslEnabled: false,
        region: 'local-env',
      },
    }),
    CatsModule,
  ],
})
export class ApplicationModule {}
```

To insert records to Dynamo, you first need to create your table, for this we use [dynamodb-data-mapper-annotations](https://github.com/awslabs/dynamodb-data-mapper-js/tree/master/packages/dynamodb-data-mapper-annotations) (under the hood). Every decorator in that package is exposed in this package as well **BUT CAPITALIZED** .

**cat.schema.ts**

```typescript
import {
  Attribute,
  AutoGeneratedHashKey,
  RangeKey,
  Table,
  VersionAttribute,
} from 'nestjs-typed-dynamodb'
import * as nanoid from 'nanoid'

@Table('cat')
class Cat {
  @RangeKey({ defaultProvider: nanoid })
  id: string

  @HashKey()
  breed: string

  @Attribute()
  age: number

  @Attribute()
  alive?: boolean

  // This property will not be saved to DynamoDB.
  notPersistedToDynamoDb: string
}
```

Note: nanoid is only used a way to assign a random id, feel free to use whatever you want

**cats.service.ts**

```typescript
import { Injectable } from '@nestjs/common'
import { ReturnDDBModel, InjectDDBModel } from 'nestjs-typed-dynamodb'
import { Cat } from './cat.schema'
import { CatInput } from './cat.input'

const ReturnModel = ReturnDDBModel<Cat>()

@Injectable()
export class CatsService {
  constructor(
    @InjectDDBModel(Cat)
    private readonly catModel: typeof ReturnModel,
  ) {}

  async findAll(): Promise<Cat[]> {
    return this.catModel.find()
  }

  async findById(id: string): Promise<Cat> {
    return this.catModel.findById(id)
  }

  async create(input: CatInput): Promise<Cat> {
    return this.catModel.create(input)
  }

  async delete(input: string): Promise<DynamoDB.DeleteItemOutput> {
    return this.catModel.findByIdAndDelete(input)
  }

  async update(id: string, item: CatInput): Promise<Cat> {
    return this.catModel.findByIdAndUpdate(id, item)
  }

  async find(input: Partial<CatInput>): Promise<Cat[]> {
    return this.catModel.find(input)
  }
}
```

Now you can use your service as you wish!

## Async configuration

**app.module.ts**

```typescript
import { Module } from '@nestjs/common'
import { TypegooseModule } from 'nestjs-typegoose'
import { CatsModule } from './cat.module.ts'

@Module({
  imports: [
    DynamoDBModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: async (config: ConfigService) => ({
        AWSConfig: {
          region: 'local',
          accessKeyId: 'null',
          secretAccessKey: 'null',
        },
        dynamoDBOptions: {
          endpoint: config.get<string>('DYNAMODB_URL', 'localhost:8000'),
          sslEnabled: false,
          region: 'local-env',
        },
      }),
      inject: [ConfigService],
    }),
  ],
})
export class ApplicationModule {}
```