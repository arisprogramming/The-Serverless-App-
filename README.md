# Getting Started
npm install --save-dev serverless-iam-roles-per-function@next
sls config credentials --provider aws --key AKIA5MB233D3R5HYCCUR --secret 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 --profile serverless




# TODO: Change the name of the org 
org: aristido
app: serverless-todo-app
service: serverless-todo-app

plugins:
  - serverless-webpack
  - serverless-iam-roles-per-function
  - serverless-reqvalidator-plugin
  - serverless-aws-documentation
  

provider:
  name: aws
  runtime: nodejs12.x
  lambdaHashingVersion: '20201221'

  stage: ${opt:stage, 'dev'}
  region: ${opt:region, 'us-east-1'}

  tracing:
    lambda: true
    apiGateway: true

  # Use these variables in the functions and resouorces section below. For example, 
  # ${self:provider.environment.ATTACHMENT_S3_BUCKET}
  environment:
    PROJECT_NAME: udacity
    TODOS_TABLE: udacity-serverless-application-todo-${self:provider.stage}
    #TODOS_CREATED_AT_INDEX: CreatedAtIndex
    ATTACHMENT_S3_BUCKET: udacity-serverless-application-todo-${self:provider.stage}
    #SIGNED_URL_EXPIRATION: 300

  #logs:
    # Enable API Gateway logs
    restApi: true

  #iam:
   # role:
    #  statements:
     #   - Effect: Allow
      #    Action:
       #     - xray:PutTelemetryRecords
        #    - xray:PutTraceSegments
         # Resource: "*"


functions:

  Auth:
    handler: src/lambda/auth/auth0Authorizer.handler

  # TODO: Configure this function - 
  # Provide iamRoleStatements property for performing Actions on DynamoDB
  GetTodos:
    handler: src/lambda/http/getTodos.handler
    events:
      - http:
          method: get
          path: todos
          cors: true
          authorizer: Auth
     iamRoleStatementsName: ${self:provider.environment.PROJECT_NAME}-gettodo-${self:provider.stage}
     iamRoleStatements:
      - Effect: Allow
        Action:
          - dynamodb:Query
        Resource: arn:aws:dynamodb:${self:provider.region}:*:table/${self:provider.environment.TODO_TABLE}
  # TODO: Configure this function - 
  # Provide iamRoleStatements property. Decide the Actions and AWS Resource. 
  # To refer to an AWS resource defined in the "Resources" section, you can use "Resource: !GetAtt <resource-name>.Arn"
  # To validate incoming HTTP requests, we have provided the request schemas in function definition below. 
 
  CreateTodo:
    handler: src/lambda/http/createTodo.handler
    events:
      - http:
          method: post
          path: todos
          cors: true
          authorizer: Auth
          documentation:
            summary: 'Create a new item'
            description: 'Create a new item'
            requestModels:
              'application/json': CreateTodoRequest
          reqValidatorName: onlyBody
    iamRoleStatementsName: ${self:provider.environment.PROJECT_NAME}-createtodo-${self:provider.stage}
    iamRoleStatements:
      - Effect: Allow
        Action:
          - dynamodb:PutItem
        Resource: arn:aws:dynamodb:${self:provider.region}:*:table/${self:provider.environment.TODO_TABLE}
  # TODO: Configure this function
  # Provide property for setting up CORS, Authorizer, iamRoleStatements, and request schemas
  UpdateTodo:
    handler: src/lambda/http/updateTodo.handler
    events:
      - http:
          method: patch
          path: todos/{todoId}
          cors: true
          authorizer: Auth
          documentation:
            summary: 'Update an item'
            description: 'Update an item'
            requestModels:
              'application/json': UpdateTodoRequest
          reqValidatorName: onlyBody
    iamRoleStatementsName: ${self:provider.environment.PROJECT_NAME}-updatetodo-${self:provider.stage}
    iamRoleStatements:
      - Effect: Allow
        Action:
          - dynamodb:UpdateItem
        Resource: arn:aws:dynamodb:${self:provider.region}:*:table/${self:provider.environment.TODO_TABLE}
  # TODO: Configure this function
  # Provide property for setting up CORS, Authorizer, iamRoleStatements
  DeleteTodo:
    handler: src/lambda/http/deleteTodo.handler
    events:
      - http:
          method: delete
          path: todos/{todoId}
    iamRoleStatementsName: ${self:provider.environment.PROJECT_NAME}-deletetodo-${self:provider.stage}
    iamRoleStatements:
      - Effect: Allow
        Action:
          - dynamodb:DeleteItem
        Resource: arn:aws:dynamodb:${self:provider.region}:*:table/${self:provider.environment.TODO_TABLE}
  # TODO: Configure this function
  # Provide property for setting up CORS, Authorizer, iamRoleStatements
  GenerateUploadUrl:
    handler: src/lambda/http/generateUploadUrl.handler
    events:
      - http:
          method: post
          path: todos/{todoId}/attachment
    iamRoleStatementsName: ${self:provider.environment.PROJECT_NAME}-s3upload-${self:provider.stage}
    iamRoleStatements:
      - Effect: Allow
        Action:
          - s3:*
        Resource: arn:aws:s3:::${self:provider.environment.BUCKET_NAME}/*

resources:
  Resources:
    TodosTable:
      Type: AWS::DynamoDB::Table
      Properties:
        AttributeDefinitions:
          - AttributeName: userId
            AttributeType: S
          - AttributeName: todoId
            AttributeType: S
        KeySchema:
          - AttributeName: userId
            KeyType: HASH
          - AttributeName: todoId
            KeyType: RANGE
        BillingMode: PAY_PER_REQUEST
        TableName: ${self:provider.environment.TODO_TABLE}
    onlyBody:
      Type: 'AWS::ApiGateway::RequestValidator'
      Properties:
        Name: 'only-body'
        RestApiId:
          Ref: ApiGatewayRestApi
        ValidateRequestBody: true
        ValidateRequestParameters: false

    AttachmentsBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: ${self:provider.environment.BUCKET_NAME}
        CorsConfiguration:
          CorsRules:
            - AllowedOrigins:
                - '*'
              AllowedHeaders:
                - '*'
              AllowedMethods:
                - GET
                - PUT
                - POST
                - DELETE
                - HEAD
              MaxAge: 3000

    BucketPolicy:
      Type: AWS::S3::BucketPolicy
      Properties:
        PolicyDocument:
          Id: MyPolicy
          Version: '2012-10-17'
          Statement:
            - Sid: PublicReadForGetBucketObjects
              Effect: Allow
              Principal: '*'
              Action: 's3:GetObject'
              Resource: 'arn:aws:s3:::${self:provider.environment.BUCKET_NAME}/* 